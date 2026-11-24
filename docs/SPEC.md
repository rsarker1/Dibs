# SPEC: SplitLive MVP

Status: agreed (2026-11-23). Source idea: [IDEA.md](IDEA.md). This spec covers the **MVP only**. Where it differs from IDEA.md (stack, lifecycle states), this spec wins.

---

## 1. Product summary

At a group meal, the host photographs the receipt and corrects what the app read. Everyone at the table then opens a link on their own phone and taps the items they had. Totals, including tax, service charge and tip, update live on every phone, and they always add up to the bill exactly to the cent. When everything is claimed, the host locks the session and marks people as paid.

The MVP is done when it works at a real dinner where everyone claims items and the totals are right. Anything not needed for that is out of scope (§12).

---

## 2. Stack

| Layer | Choice |
|---|---|
| Frontend | React + TypeScript, phone-first single-page app (Vite) |
| Backend | Node.js + TypeScript: REST API, WebSocket push, receipt parsing |
| Money math | `packages/split-math`: pure TypeScript with no dependencies, used by both web and API |
| Receipt reading | Claude vision via `@anthropic-ai/sdk`, called from the API (no separate service) |
| Database | Postgres 16 |
| Local dev | Docker Compose (Postgres, 2 API instances, Caddy as the load balancer, web) |
| Deploy | AWS: ECS Fargate (Spot) + ALB + RDS Postgres + S3, defined in Terraform |
| CI/CD | GitHub Actions |
| Tests | Vitest, fast-check, Playwright |
| Repo | pnpm monorepo: `apps/web`, `apps/api`, `packages/split-math`, `infra/`, `evals/` |

---

## 3. Decision log

Each entry gives the choice and a one-line reason. These are the interview talking points.

| # | Decision | Choice | Why |
|---|---|---|---|
| D1 | Python receipt service | **Dropped**; parsing lives in the Node API | The only job is one LLM call. A second language would split the money math across languages and add a service, a CI job and a deploy target for nothing. |
| D2 | Source of truth for the bill | **Line items.** Subtotal = Σ items; grand total = subtotal + tax + service + tip. Printed subtotal and total are checks only. The host **cannot confirm** while they don't match. | One authority makes the to-the-cent invariant well defined and testable. |
| D3 | Discounts | Out of scope. The host edits item prices by hand to absorb them. | Negative amounts complicate proportional allocation and rounding. Revisit if the receipt eval shows discounts are common. |
| D4 | Quantity > 1 lines | When the receipt is confirmed, the line is expanded into N **units**. The line total is spread across the units with largest-remainder (e.g. $20.00 / 3 → 667, 667, 666). | Every unit is then an ordinary claimable item, so sharing a unit needs no special case. |
| D5 | People without phones | The host can add a "phone-less" participant and **claim or unclaim for any participant**. Guests only toggle their own claims. | Dead batteries are normal at real dinners. It also lets the host clean up forgotten claims. |
| D6 | Lifecycle | Three states, `draft → open → locked`. "Settled" is derived (every non-host participant is paid). | Paid flags can be toggled back off, so settled is a view, not a transition. Payments will become ledger entries later. |
| D7 | Unlock | Host only. Unlocking **clears all paid flags**, after a confirmation. | Totals can change after an unlock, so a paid flag would be stale. |
| D8 | Unclaimed units at lock | **Locking is blocked** and the unclaimed units are listed. A helper, "split remaining among everyone", creates ordinary claims. | Money is never allocated silently. |
| D9 | Tip base | Pre-tax **item subtotal**, excluding the service charge. Percentage or fixed amount. | The common convention. The fixed-amount option covers everything else, without adding a toggle. |
| D10 | Concurrency | Claims are rows with `PRIMARY KEY (unit_id, participant_id)`. Claim = `INSERT … ON CONFLICT DO NOTHING`; unclaim = `DELETE`. Explicit claim/unclaim actions, never toggle. | The operations commute and are idempotent, so the design makes lost updates impossible instead of detecting them. No row locks or version checks are needed for claims. |
| D11 | Ordering and resync | A per-session `version` is incremented in the same transaction as each change. | Gives a total order for broadcasts and a cheap check for "am I stale?". |
| D12 | Where the math runs | Shared `split-math` package. **The server is authoritative**; the client recomputes locally so taps feel instant, and the server's next snapshot replaces the local result. | Same code, same results, instant feedback. |
| D13 | Mutation transport | Mutations go over **HTTP POST** with a client action ID. The WebSocket is server-to-client push only. | Request/response gives clean acks, retries, rate limiting and error codes. The WebSocket stays simple: snapshots out, nothing in. |
| D14 | Broadcast payload | **Full session snapshots**, not deltas. | Sessions are a few KB. Resync after a reconnect is the same code path as a normal update. |
| D15 | Multi-instance fan-out | Postgres `LISTEN/NOTIFY`. The payload is only `{sessionId, version}`; each instance reloads the snapshot itself. | No Redis. Small payloads stay under NOTIFY's 8 KB limit. |
| D16 | AWS shape | ECS Fargate Spot (2 tasks) + ALB + single-AZ RDS + S3. No NAT gateway (tasks in public subnets, security groups allow only the ALB). Terraform, torn down when idle. | Managed-AWS story at ~$35–40/month while running and ~$0 when torn down. Spot interruptions exercise the reconnect path. |
| D17 | Receipt parsing call | Synchronous HTTP request (60 s timeout) with a fallback to manual entry. | The host waits on this screen either way. A queue adds parts and no benefit at this scale. |
| D18 | Vision model | `claude-opus-5-5` with structured outputs. Compare against `claude-haiku-5-5` on the eval set; switch only if accuracy matches. | Accuracy first. The eval decides cost trade-offs, not guesswork. |
| D19 | Model money output | The model returns amounts as the **decimal strings printed on the receipt** (e.g. `"12.50"`). Code parses them to integer cents. | The model copies text instead of doing arithmetic, and there is no float anywhere. |
| D20 | Low confidence | Shown through **deterministic checks** on the review screen (§6.3), plus an optional `unreadable` flag per line from the model. | LLM confidence scores aren't trustworthy; arithmetic checks are. |
| D21 | Retention | Unopened drafts: deleted after 24 h. Open and locked sessions: deleted 30 days after last activity. Receipt images: deleted 24 h after upload (S3 lifecycle rule). | Settling up often happens the next day. The parsed items, not the photo, are the record. |
| D22 | Rounding | **Round once, at the end**, over the whole bill, with largest-remainder. Ties go to the earliest joiner; the host is always first. | Everyone is within 1¢ of their exact share. Stray cents land on the host, who is the one being paid back. |
| D23 | Duplicate names | Allowed, displayed as "Alex (2)". Identity is a per-participant token, never the name. | Names are labels; the token is what identifies a person. |
| D24 | Host token recovery | After confirming, the host sees a "Save your host link" screen with a copy button. No other recovery. | No accounts in the MVP. |
| D25 | Breakdown display | Per person: claimed items shown as fractions of their **exact** price (e.g. "⅓ of Nachos $14.00"), plus **one total**. No rounded per-component amounts. | Any amount shown must reconcile. Rounded component amounts can't reconcile with a total that is rounded once. |
| D26 | Editing while open | While `open`, the host may edit a line's name or total and the tax, service charge and tip. Quantity changes and deleting lines are only allowed if that line has no claims; adding lines is always allowed. | Typos get noticed mid-meal, and tip is often decided at the end. Claims are never silently destroyed. |

---

## 4. Domain model

### 4.1 Concepts

- **Session**: one bill at one meal. Has a join token (shared) and a host token (secret).
- **Line item**: a receipt line: name, quantity, line total in cents.
- **Unit**: one claimable piece of a line item. Lines are expanded into units when the receipt is confirmed (D4).
- **Participant**: someone at the table: the host, a guest who joined by link, or a phone-less person the host added.
- **Claim**: "this participant had (a share of) this unit". A unit with k claims is split evenly k ways.
- **Obligation**: computed after lock. Participant owes the host N cents, plus a paid flag.

### 4.2 Schema (Postgres)

All money columns are `bigint` cents with `CHECK (… >= 0)`. There are no `numeric`, `real` or `float` money columns.

```
sessions
  id                    uuid pk
  join_token            text unique         -- 128-bit random, base64url
  host_token_hash       bytea               -- sha256(host token)
  status                text check in ('draft','open','locked')
  version               bigint not null default 0
  tax_cents             bigint
  service_cents         bigint
  tip_mode              text check in ('percent','fixed')
  tip_bps               int                 -- basis points, 1800 = 18%; used when percent
  tip_fixed_cents       bigint              -- used when fixed
  printed_subtotal_cents bigint             -- check value only (D2)
  printed_total_cents   bigint              -- check value only (D2)
  currency              text not null default 'USD'   -- fixed in the MVP
  created_at, last_activity_at, opened_at, locked_at  timestamptz

receipt_images
  id uuid pk, session_id fk, storage_key text, uploaded_at timestamptz

line_items
  id uuid pk, session_id fk, position int, name text,
  quantity int check (quantity between 1 and 50),
  line_total_cents bigint, unreadable bool default false

units                                         -- created on confirm; re-derived on edit (D26)
  id uuid pk, session_id fk, line_item_id fk, unit_index int, amount_cents bigint
  unique (line_item_id, unit_index)

participants
  id uuid pk, session_id fk, display_name text, name_suffix int null,
  token_hash bytea null,                      -- null for phone-less participants
  is_host bool, phoneless bool, join_seq int,  -- host = 0, then 1, 2, …
  paid_at timestamptz null, created_at timestamptz
  unique (session_id, join_seq)

claims
  unit_id fk, participant_id fk, created_at
  primary key (unit_id, participant_id)

applied_actions                               -- idempotency (§7.3)
  session_id fk, action_id uuid, response jsonb, created_at
  primary key (session_id, action_id)

rate_limit_buckets
  key text pk, window_start timestamptz, count int
```

All child rows use `ON DELETE CASCADE` from `sessions`, so expiry is a single delete.

**Hook for the future version (not built):** later, a session gets a nullable `group_id` and participants get a nullable `person_id`. A locked session's obligations become ledger entries. Nothing in the MVP schema blocks this, and none of it is built now.

### 4.3 Limits

These keep the property-test domain and abuse surface bounded:
- ≤ 100 line items and ≤ 200 units per session
- ≤ 30 participants
- Each line ≤ $10,000.00; grand total ≤ $100,000.00
- Display name 1–30 characters, trimmed

---

## 5. Money math: `packages/split-math`

Pure, deterministic, no dependencies, no I/O, no `Date`, no randomness. Number money values are always safe integers. Intermediate products use `BigInt`. **No floating-point arithmetic for money.** A lint rule bans `/`, `Math.round`, `Math.floor`, `toFixed` and `parseFloat` on money values in this package; all division goes through one helper.

### 5.1 Functions

```ts
allocate(totalCents: number, weights: bigint[], tieOrder: number[]): number[]
  // Largest-remainder. Returns parts summing exactly to totalCents.
  // Ties broken by tieOrder ascending (lower index wins).

expandUnits(lineTotalCents: number, quantity: number): number[]
  // allocate(lineTotal, [1,1,…], [0,1,…]), so earlier units get the extra cents.

tipCents(itemSubtotalCents: number, tip: {mode:'percent', bps:number} | {mode:'fixed', cents:number}): number
  // percent: round-half-up(subtotal × bps / 10000), in integer math.

computeSplit(input: SplitInput): SplitResult
```

### 5.2 `computeSplit`

Input: units (`id`, `amountCents`), claims, participants (`id`, `joinSeq`), tax, service, tip.

1. `S` = Σ unit amounts (the item subtotal); `G` = S + tax + service + tip (the grand total). `S > 0` is guaranteed by confirm validation.
2. Every unit with zero claims is assigned to a pseudo-participant **`UNCLAIMED`**, ordered after all real participants. This makes the invariant hold while the session is open, too. At lock, `UNCLAIMED` must be 0 (D8).
3. Exact item share: unit `u` with `k` claimers gives each claimer `amount_u / k`. Over a common denominator `D` = lcm of all claimer counts, each party `p` has an integer numerator `N_p`, where Σ N_p = S × D.
4. Each party's exact grand-total share is `N_p × G / (S × D)`. Run `allocate(G, N, joinSeq order)` **once** (D22).
5. Output: per-participant `totalCents`, the `UNCLAIMED` amount, and per-participant claimed units with fractions `{unitId, numerator: 1, denominator: k}` for display (D25).

### 5.3 Invariants (property-tested, §10.1)

- **Reconciliation:** Σ totals (including `UNCLAIMED`) == G exactly.
- **Fairness bound:** each party's total is within 1¢ of its exact rational share (floor or ceil).
- **Determinism:** the same input gives the same output, and the order of claims or units in the input doesn't change the result.
- **Monotonicity:** claiming an additional unit never decreases your total.
- `expandUnits` parts sum to the line total and differ by at most 1¢.
- With no tax, service or tip, each person's total equals the rounded exact item share.

### 5.4 Display formatting

`formatCents(n)` → `"$1,234.56"` uses integer division and modulo by 100 only. Parsing user input (`"12.5"`, `"12.50"`, `"$12"`) to cents is a strict regex, never `parseFloat`. Invalid input is rejected.

---

## 6. Core flows

### 6.1 Host creates a session

1. `POST /api/sessions` → `{sessionId, joinToken, hostToken}`. The host's participant row is created (`join_seq = 0`). The host enters a display name.
2. The host link is `/s/{joinToken}#host={hostToken}`. The token is in the URL **fragment**, so it never reaches server or ALB logs. The web app saves it in `localStorage` for that session.

### 6.2 Receipt upload and parsing

1. The client resizes the photo so its long edge is ≤ 2000 px and converts it to JPEG (accepts JPEG, PNG, WebP; ≤ 10 MB before resizing).
2. `POST /api/sessions/:id/receipt` (host). The API stores the image (S3, or local disk in dev) and calls Claude synchronously with a 60 s timeout.
3. Model call: `claude-opus-5-5`, image + instructions, structured output schema:
   ```
   { items: [{ name: string, quantity: integer, line_total: string, unreadable: boolean }],
     subtotal: string|null, tax: string|null, service_charge: string|null, total: string|null }
   ```
   Amounts are decimal strings copied from the receipt (D19). Code parses them to cents and **rejects any value that doesn't parse**, rather than guessing. Any printed tip line is ignored, since the tip is set by the host.
4. If parsing succeeds, the draft is pre-filled. On timeout, model error or invalid output, the host sees "Couldn't read this receipt. Retry or enter items manually."
5. The host can always skip the upload and enter items manually.

### 6.3 Host review (draft)

- Edit, add, delete or reorder lines; merge two lines (names joined, totals summed, quantities summed).
- Edit tax, service charge, printed subtotal and printed total. Set the tip as a percentage (preset buttons 15/18/20/22% + custom) or a fixed amount.
- **Checks**, shown live and highlighted per line:
  - Σ line totals == printed subtotal
  - printed subtotal + tax + service == printed total
  - lines marked `unreadable`
- **Confirm** is enabled only when: ≥ 1 line, every amount valid, S > 0, both reconciliation checks pass, and every limit (§4.3) is respected. On confirm: lines are expanded into units, `status = open`, and the "Save your host link" screen is shown, followed by the join QR code and link.

### 6.4 Guests join

- Opening `/s/{joinToken}` on a phone shows a name prompt → `POST /api/sessions/:id/participants` → `{participantId, participantToken}`, stored in `localStorage`. Returning to the link on the same browser skips the prompt.
- Joining is allowed only while `open`. A `draft` session shows "The host is still setting up". A `locked` session is read-only and shows the totals; nobody can join.
- A duplicate name gets the next suffix, "Alex (2)" (D23). A guest can rename themselves.

### 6.5 Live claiming (open)

- Each unit is a row on screen: the unit's name and price, avatars or initials of claimers, and a highlighted state if you claimed it. Multi-unit lines show as grouped rows ("Beer 1 of 3").
- Tapping sends `claim` or `unclaim` (§7.2). The UI updates optimistically and the server snapshot is authoritative.
- A sticky footer shows **your running total** and the session-wide "Unclaimed: $X.XX".
- Host extras: a participant picker to claim on behalf of anyone, "Add phone-less person", "Remove participant" (only if they have no claims), and receipt edits within D26.

### 6.6 Lock and settle

- **Lock** (host): rejected with the list of unclaimed units if `UNCLAIMED > 0`. The helper "Split remaining among everyone" creates claims for every participant on every unclaimed unit, in one action.
- **Locked view** (everyone): each participant's total and what they owe the host. The host's own row reads "Host" and owes nothing.
- **Mark paid / unpaid** (host): one toggle per participant. When every non-host participant is paid, show "Settled ✓".
- **Unlock** (host): confirm dialog "This clears all paid marks" → `status = open`, paid flags cleared (D7).

### 6.7 Permissions

| Action | Host | Guest |
|---|---|---|
| Edit receipt, tax, service, tip | ✓ (draft; limited while open, D26) | ✗ |
| Confirm, lock, unlock, mark paid | ✓ | ✗ |
| Claim or unclaim for self | ✓ | ✓ (open only) |
| Claim or unclaim for others | ✓ | ✗ |
| Add phone-less participant / remove participant with no claims | ✓ | ✗ |
| Rename self | ✓ | ✓ |

Server checks every action against the bearer token; the UI hiding a button is never the check.

---

## 7. API and real-time protocol

### 7.1 Auth

Every request sends `Authorization: Bearer <token>`. The token is either a host token or a participant token. The server hashes it and looks it up. Tokens are 128-bit random values in base64url; only the sha256 hash is stored.

### 7.2 Endpoints

```
POST   /api/sessions                                   create (rate-limited)
POST   /api/sessions/:id/receipt                       upload + parse (host, rate-limited)
PUT    /api/sessions/:id/draft                         replace draft lines + amounts (host, draft)
POST   /api/sessions/:id/confirm                       draft → open (host)
POST   /api/sessions/:id/participants                  join (open)
GET    /api/sessions/:id/snapshot                      current state (any token)
POST   /api/sessions/:id/actions                       all open/locked mutations (below)
GET    /ws?session=:id&token=…                         push channel
```

`POST /actions` body: `{ actionId: uuid, type, ...payload }`. Types:

`claim {unitId, participantId?}` · `unclaim {unitId, participantId?}` · `splitRemaining` · `editLine {lineId, name?, lineTotalCents?, quantity?}` · `addLine` · `deleteLine` · `setAmounts {tax?, service?, tip?}` · `addPhoneless {name}` · `removeParticipant {participantId}` · `rename {name}` · `lock` · `unlock` · `setPaid {participantId, paid}`

`participantId` defaults to the caller; only the host may name someone else. The response is `{version, snapshot}` or a typed error (`not_host`, `wrong_status`, `unclaimed_units`, `has_claims`, `limit_exceeded`, …).

### 7.3 Applying an action (one transaction)

```
BEGIN
  INSERT INTO applied_actions (session_id, action_id) … ON CONFLICT DO NOTHING
    → on conflict: ROLLBACK and return the stored response (idempotent retry)
  SELECT … FROM sessions WHERE id = $1 FOR UPDATE      -- serializes status checks + version bump
  check permission + status
  apply the change (e.g. INSERT claim ON CONFLICT DO NOTHING)
  UPDATE sessions SET version = version + 1, last_activity_at = now() RETURNING version
  UPDATE applied_actions SET response = …
  NOTIFY session_changed, '{"sessionId":"…","version":N}'
COMMIT
```

Locking the session row serializes writes **per session**. That's fine for ≤ 30 people tapping, and it means status checks such as "lock vs. a late claim" can't race. It's the only lock. Claims themselves stay commutative (D10), so the lock only orders writes; it never rejects a claim because someone else was faster.

`NOTIFY` inside a transaction is delivered only on commit, so a broadcast can never get ahead of the data.

### 7.4 WebSocket and fan-out

- Each API instance holds **one dedicated `LISTEN session_changed` connection** (outside the pool) and a map of `sessionId → local sockets`.
- On a notification for a session with local sockets: load the snapshot once and push it to those sockets if its `version` is newer than what each socket last received. Bursts are coalesced: if a load is already in flight for that session, mark it dirty and reload once more afterwards.
- On connect: authenticate the token, subscribe, and send the current snapshot right away. **Reconnect and initial load are the same code path** (D14).
- If the LISTEN connection drops: reconnect with backoff, then re-push snapshots for every session with local sockets, because notifications may have been missed.
- Heartbeat: ping every 25 s (keeps the ALB idle timeout from closing the socket); drop sockets that miss 2 pongs.

### 7.5 Client behaviour

- **Pending action queue:** each action keeps its `actionId` until acknowledged. Timeouts or network errors retry with the **same** ID and backoff.
- **Optimistic state** = last server snapshot + pending actions, replayed through `split-math`. When a snapshot arrives, actions it already includes are dropped from pending.
- **Reconnect:** exponential backoff (0.5 s → 10 s cap, with jitter). Also reconnect on `visibilitychange` (phones suspend background tabs) and `online` events. A "Reconnecting…" banner appears after 2 s offline.

---

## 8. Security and privacy

- Join and host tokens: unguessable, 128-bit (§7.1). The host token lives only in the URL fragment and `localStorage`.
- Only display names are stored. No emails, phone numbers or payment details.
- Receipt images: private S3 bucket, no public URLs, deleted 24 h after upload by a bucket lifecycle rule (D21). DB rows are deleted by the cleanup job.
- Rate limits (Postgres fixed window): create session 10/h per IP; receipt upload 20/h per IP; actions 120/min per participant. Errors return `429` with `Retry-After`.
- Request body limits; upload MIME type checked by magic bytes.
- CORS locked to the app's origin. Security headers via the web server.
- Expiry job runs every 10 minutes on whichever instance holds `pg_try_advisory_lock`. It deletes unopened drafts older than 24 h and sessions inactive for 30 days.

---

## 9. Infrastructure

### 9.1 Local

`docker compose up` runs Postgres, `api-1`, `api-2`, Caddy (load balancer, round-robin, WebSocket upgrade) and the web dev server. Two API instances run locally **from day one of real-time work**, so cross-instance bugs show up early.

For the real dinner before AWS (§11, M5), expose the laptop with a tunnel (e.g. Cloudflare Tunnel) so phones on cellular can reach it.

### 9.2 AWS (Terraform, `infra/`)

- VPC with 2 public subnets; **no NAT gateway**. ECS tasks get public IPs; their security group allows inbound traffic only from the ALB.
- ALB (HTTPS via ACM; idle timeout 120 s) → ECS Fargate service, 2 tasks, 0.25 vCPU / 0.5 GB, Fargate Spot.
- RDS Postgres `db.t4g.micro`, single-AZ, security group allows only the ECS tasks. Automated backups on.
- S3 bucket for receipt images (private, 1-day lifecycle) and a bucket or CloudFront for the static web build.
- Secrets (`ANTHROPIC_API_KEY`, DB URL) in SSM Parameter Store, injected into tasks.
- `terraform destroy` / `apply` documented in the README. Data survives in an RDS final snapshot.
- Expected cost: roughly $35–40/month while running; near $0 when torn down. Verify with the AWS pricing calculator when deploying.

### 9.3 CI/CD (GitHub Actions)

- Every push and PR: install, lint, typecheck, unit + property tests, API integration tests against a Postgres service container.
- PRs touching `apps/api/src/receipt/**` or `evals/**`: run the receipt eval (§10.2) and post the accuracy report as a job summary. Needs `ANTHROPIC_API_KEY` as a repo secret; costs about $0.50 per run.
- `main`: Playwright E2E against the Compose stack, then build images → ECR → ECS deploy (only when the infrastructure is up).

### 9.4 Monitoring

- Structured JSON logs (pino) to CloudWatch, with a request ID and session ID on every line. Never log tokens.
- Metrics: open WebSocket connections, actions/sec, action latency, NOTIFY-to-push lag, receipt parse latency and failure rate.
- One CloudWatch alarm: ALB 5xx rate > 1% for 5 minutes.

---

## 10. Testing

### 10.1 Money math (`packages/split-math`)

- **Property tests (fast-check):** thousands of generated receipts within §4.3's limits (random units, prices from 1¢ to the max, 0–30 participants, random claim patterns including all-shared and none-claimed, tax, service, tip at 0–100% and fixed amounts). Assert every invariant in §5.3.
- **Hand-written edge cases:** 1 item / 1 person; every item shared by all; 1¢ items split 3 ways; 100% tip; $0 tax; 3-way ties on remainder; quantity 3 at $20.00; maximum-size receipt.
- Coverage target 100% for this package.

### 10.2 Receipt eval (`evals/receipts/`)

- 15–20 of Rahul's real receipts with hand-labeled expected JSON. **Redact card digits and other personal details before committing.**
- Report per-field accuracy: item count, per-line name match (normalized), per-line total exact, tax, service, total, and whether the reconciliation checks pass out of the box.
- Run for `claude-opus-5-5` and `claude-haiku-5-5` once, to make D18's comparison.

### 10.3 API integration (Vitest + real Postgres)

- 2 to 30 parallel `claim` requests on the same unit → final state has exactly those claimers, and the versions are strictly increasing.
- The same `actionId` sent twice (including concurrently) → applied once, and both responses are identical.
- A claim racing a lock → either the claim lands and the lock sees it, or the claim fails with `wrong_status`. Never a locked session with a change after locking.
- Permissions table (§6.7): each forbidden action is rejected.
- Cross-instance: a client on instance A sees a change made through instance B.

### 10.4 E2E (Playwright)

- 3–4 browser contexts in one session: host confirms, guests join, claim and share items, one goes offline and claims while offline, then reconnects and converges; lock, mark paid, settled.
- Browsers connected to different API instances (the Compose stack).

---

## 11. Milestones

About 10 hours/week. Realistic plan 8–10 weeks; the dates below are the stretch. Each milestone leaves `main` working and deployable.

| M | Target week | Deliverable | Done when |
|---|---|---|---|
| M0 | 1 | Monorepo, Compose (Postgres), **CI running**, `split-math` with property tests | CI green; every §5.3 invariant property-tested |
| M1 | 2 | Schema + migrations, create session, **manual** receipt entry and review screen with checks, confirm | A host can create a session, type in a receipt and open it |
| M2 | 3 | Photo upload + Claude parsing pre-fills the review; eval harness + CI job | Eval report generated for both models; D18 decided |
| M3 | 4 | Join by link/QR, claim/unclaim over HTTP, WebSocket snapshots, running totals, idempotency (one instance) | Two phones on the LAN see each other's claims live |
| M4 | 5 | 2 instances behind Caddy, LISTEN/NOTIFY fan-out, reconnect/resync, concurrency tests (§10.3) | Cross-instance + concurrency tests green |
| M5 | 6 | Lock/unlock, settle, paid flags, host-claims-for-others, phone-less participants, "split remaining", phone-first polish. **Real dinner** via tunnel | Used at a real dinner with 3+ phones |
| M6 | 7 | Terraform AWS stack, deploy pipeline, rate limits, expiry job, monitoring + alarm | Public HTTPS link works; deploys from `main` |
| M7 | 8 | Playwright E2E in CI, README (demo GIF, architecture diagram, setup), design doc from §3 | Every item in §13 checked |

---

## 12. Out of scope for the MVP

Accounts and login · groups and running balances · payment links or any money movement · uneven splits of one item · discounts, coupons, gift cards (D3) · multiple currencies · native apps · push notifications · editing the receipt in ways that would remove claims (D26) · host-token recovery beyond the saved link (D24) · a breakdown of each person's share into tax/tip/service components (D25).

---

## 13. Definition of done

- [ ] Used at a real dinner with 3+ people on separate phones
- [ ] Property tests prove the to-the-cent invariant across thousands of generated receipts
- [ ] Receipt eval runs in CI with an accuracy report
- [ ] Simultaneous claims, idempotent retries and reconnects pass automated tests, across 2 instances
- [ ] Deployed on AWS at a public HTTPS link; CI runs tests on every push
- [ ] `README.md` with demo GIF, architecture diagram and setup instructions
- [ ] Short design doc built from §3, with the reason for each decision
