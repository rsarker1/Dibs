# IDEA: Live Bill Splitting App

> **For Claude Code:** This file is the idea, not the spec. Use it to write `SPEC.md` for the **MVP only** (section 4). The full version (section 5) is context so the MVP's design doesn't block it later. Do not spec or build full-version features. Before writing the spec, interview me on every item in section 9 (Open questions) and anything else that's ambiguous.

---

## 1. One-line summary

At a group meal, one person photographs the receipt, everyone opens a link on their own phone, taps the items they had, and the app splits the bill (including tax and tip) exactly to the cent, updating live for everyone.

## 2. Why this exists

- **The problem:** splitting an itemized bill at the table is slow and error-prone. Someone does mental math, tax and tip get split evenly even when orders weren't, and amounts don't add up.
- **Why not just ask an AI with a photo of the receipt:** a prompt can't coordinate several people claiming items on their own phones at the same time, and it can't guarantee that every share adds up to the receipt total to the cent. This app's reason to exist is **live multi-person coordination** plus **exact money math**.
- **Existing apps** (Splitwise and others) handle splitting. The angle here is the live claiming session: no accounts, everyone claims their own items at the table, totals update in real time.

## 3. Goals

### Product goals
- Used at real dinners by me and friends; every dinner puts it on several phones.
- Faster and less awkward than one person doing the math.

### Portfolio goals
This is one of two portfolio projects for a job search targeting NYC fintech/commerce companies (e.g. Capital One, Whatnot). It should demonstrate:
- **Money correctness:** integer-cent math, deterministic rounding, totals that always reconcile.
- **Real-time systems:** WebSockets, concurrent updates, reconnect handling, multiple server instances.
- **Professional engineering:** tests (including property-based tests), CI, deployment on AWS, monitoring, a design doc with reasoned decisions.
- **AI-assisted development:** built with Claude Code using a spec-first workflow; specs and `CLAUDE.md` kept in the repo.

### Required stack
- **Frontend:** React + TypeScript (phone-first web app; no native app)
- **Backend:** Node.js + TypeScript (API + WebSocket server)
- **Receipt reading:** Python service using a vision-capable LLM
- **Database:** Postgres
- **Infra:** Docker (Docker Compose locally), AWS for deployment
- **Version control/CI:** Git, GitHub Actions

## 4. MVP scope (build this first)

### 4.1 Core flow

1. **Host creates a session and scans the receipt.**
   - Host takes or uploads a photo of the receipt.
   - The Python receipt service extracts line items (name, quantity, price), subtotal, tax, any service charge, and total.
2. **Host reviews and corrects the parsed receipt (required step).**
   - Host can edit, add, delete, or merge items, and fix tax/total.
   - The app shows a warning if item prices don't add up to the receipt subtotal.
   - Host sets the tip (percentage or fixed amount).
   - The session cannot be opened to guests until the host confirms the receipt.
3. **Guests join.**
   - Via a shareable link or QR code shown on the host's phone.
   - Guests enter only a display name. No accounts, no sign-up.
4. **Live claiming.**
   - Each person taps the items they had on their own phone.
   - Every claim/unclaim appears on all phones in real time.
   - Shared items: if multiple people claim the same item, it is split evenly among them.
   - Items with quantity > 1 (e.g. "3 × beer") can be claimed per unit.
   - Everyone sees a running total for themselves.
5. **Totals.**
   - Tax, service charge and tip are split in proportion to each person's share of the item subtotal.
   - All amounts are exact to the cent, and the sum of everyone's totals equals the receipt total (plus tip) exactly.
6. **Settle up.**
   - Host locks the session once all items are claimed (see open questions for unclaimed items).
   - Settle screen shows what each person owes the host.
   - Host can mark each person as paid.

### 4.2 Money rules (non-negotiable)

- All money is stored and computed as **integer cents**. No floating-point arithmetic for money anywhere: frontend, backend, or database.
- The splitting logic lives in **one pure, dependency-free module** shared by frontend and backend (or used only on the backend with results pushed to clients). It must be deterministic: the same inputs always produce the same outputs.
- **Rounding:** when an amount can't divide evenly, distribute leftover cents using a deterministic method (e.g. largest-remainder), so the parts always sum exactly to the whole. The tie-break rule must be documented.
- **Invariant:** sum of all participants' totals == receipt total + tip, to the cent, for every possible receipt. This is enforced by property-based tests.

### 4.3 Real-time and concurrency requirements

- **Simultaneous claims:** two people tapping the same item at the same instant must produce a consistent result for everyone (no lost updates, no double-counting).
- **Idempotent actions:** each client action carries a client-generated ID so retries after a dropped connection don't apply twice.
- **Reconnects:** a phone that loses signal and reconnects resyncs to the current session state.
- **Multiple server instances:** a change made through one server instance reaches clients connected to another. Preferred approach: Postgres `LISTEN/NOTIFY` (keeps the stack to Postgres, no Redis).

### 4.4 Session lifecycle

`draft` (host editing receipt) → `open` (guests claiming) → `locked` (claims frozen, totals final) → `settled` (all marked paid)

- Only the host can edit the receipt, lock, or unlock.
- Sessions expire after a period of inactivity (duration to decide).

### 4.5 Security and privacy (MVP level)

- Session links use unguessable tokens.
- Host permissions use a separate host token; guests can't act as host.
- Only display names are collected. No emails, phone numbers, or payment details.
- Receipt images are deleted after a set retention period (to decide).
- Basic rate limiting on session creation and receipt uploads.

### 4.6 Explicitly out of scope for the MVP

- User accounts and login
- Saved groups and running balances across multiple dinners
- Payment links or any money movement
- Uneven splits of a single item (e.g. "I had two-thirds of the pizza")
- Multiple currencies
- Discounts, coupons, gift cards on the receipt
- Native mobile apps
- Push notifications

## 5. Full version (future, context only; do not build yet)

Listed roughly in priority order:

1. **Groups with running balances:** recurring friend groups, a double-entry ledger of who owes whom across many dinners, and debt simplification (minimize the number of payments needed to settle a group).
2. **Optional accounts:** keep history and groups across devices; guests can still join without one.
3. **Uneven splits:** custom ratios or amounts for a single item.
4. **Payment handoff:** links that open a payment app with the amount pre-filled. The app itself never holds or moves money.
5. **Receipt handling improvements:** discounts, coupons, service charges, comps; learning from host corrections to improve parsing.
6. **Multiple currencies** for travel.
7. **Notifications:** reminders for unpaid balances.
8. **Spending history:** per-person and per-group summaries over time.

**Design implication for the MVP:** structure the data model so that sessions can later belong to a group and feed a ledger, but do not build the ledger now.

## 6. Testing strategy

- **Splitting math:** property-based tests (e.g. fast-check) generating thousands of random receipts, claim patterns, tax/tip values; assert the to-the-cent invariant and determinism. Plus hand-written edge cases (1 item / 1 person, every item shared by all, $0.01 items, 100% tip).
- **Receipt reading:** a labeled test set of 15–20 of my own real receipts with expected outputs; report per-field accuracy and run it in CI whenever the receipt service changes.
- **Concurrency:** tests that fire simultaneous claims at the same item and verify the final state.
- **End-to-end:** multiple simulated browsers in one session (e.g. Playwright) claiming, disconnecting, reconnecting, and settling.

## 7. Milestones (≈10 hours/week; ~6 weeks best case, 8–10 realistic)

| Week | Milestone |
|---|---|
| 1 | Repo, Docker Compose, Postgres schema, splitting-math module with property tests |
| 2 | Receipt service (vision LLM → structured JSON) and host review/edit screen; receipt test set |
| 3–4 | Sessions, join links/QR, live claiming over WebSockets, concurrency, reconnects, multi-instance fan-out |
| 5 | Settle-up screen, phone-first polish. **Use it at a real dinner this week, even before deploying.** |
| 6 | AWS deployment, GitHub Actions CI/CD, basic monitoring and logging |

## 8. MVP definition of done

- [ ] Used successfully at at least one real dinner with 3+ people on separate phones
- [ ] Property tests prove the to-the-cent invariant across thousands of generated receipts
- [ ] Receipt test set runs in CI with an accuracy report
- [ ] Simultaneous claims and reconnects behave correctly in automated tests
- [ ] Deployed on AWS, reachable by a public link, with CI running tests on every push
- [ ] `README.md` with a demo GIF, architecture diagram, and setup instructions
- [ ] Short design doc covering the decisions in section 9 and why each was made

## 9. Open questions (resolve during spec interview)

1. **Unclaimed items at lock time:** block locking until everything is claimed, have the host absorb them, or split them evenly among everyone?
2. **Tip base:** tip on the pre-tax subtotal (common default) or on the total? Let the host choose?
3. **Concurrency approach:** row-level locks vs. optimistic version numbers for claims. Which, and why?
4. **Where the splitting math runs:** backend only (clients just display results) or shared module on both sides for instant local feedback?
5. **WebSockets on AWS:** load balancer + containers on ECS vs. API Gateway WebSocket API + Lambda. Compare cost, complexity, and how each handles multi-instance fan-out.
6. **Receipt service interface:** synchronous call from the Node API, or async via a queue (S3 upload → queue → Python worker → result)?
7. **Which vision model** for receipt reading, and what happens when its output is low-confidence?
8. **Session expiry and receipt image retention periods.**
9. **Rounding tie-break rule:** who gets the extra cent when remainders are equal?
10. **Name collisions:** two guests named "Alex" in one session.

## 10. Notes for the spec

- Keep the MVP small. If something isn't needed for "a real dinner where everyone claims items and the totals are right," it's out.
- Every design decision in section 9 should be recorded in the spec with the chosen option and a one-line reason, since these become interview talking points.
- Break the spec into thin, testable milestones that each leave the app in a working state.
