# Disputed Deadline — Deployment & Interaction Guide

Standalone GenLayer Intelligent Contract. No frontend, no backend, no
database. This file covers GenLayer Studio / StudioNet CLI/Studio-UI
interaction only.

## What this is

Two parties stake equal native GEN on opposite sides of the claim:

> "Artifact A will satisfy condition C by deadline block-time T."

Validators independently fetch artifact A themselves at/after T and reach
consensus on a structured result. A deterministic function then pays out.
See the contract's module docstring in
[`contracts/disputed_deadline.py`](../contracts/disputed_deadline.py) for the
full trust-boundary spec this implements.

## Current verified deployment

**`0x151b73847663116eab8822749031d56Fb86a82bE`** — GenLayer StudioNet.

This is the fourth deployment. See [`REVIEW.md`](../REVIEW.md) at the repo
root for the full history of what an external review flagged, the root
cause of each issue, and how each was fixed and re-verified live. In short:

1. An address-argument coercion crash in two view functions — fixed.
2. An over-strict consensus comparison on the LLM/vision paths that made
   real validator agreement fail almost regardless of the correct answer —
   fixed.
3. **Settlement not grounded to the pinned deadline** — the finding that
   forced this deployment. `evaluate()` used to fetch the artifact whenever
   the transaction happened to land, with no upper bound, so a mutable page
   could be edited between the deadline and an arbitrarily later call. Fixed
   with two different mechanisms depending on condition type (below).

## How evidence is grounded to the deadline (current design)

The fix is condition-type-specific, because "the artifact" means two
different things depending on what's being checked:

### Live-status conditions — `HTTP_STATUS`, `STRING_CONTAINS`

These check *current, dynamic* server state ("is my demo up right now") and
can never be hash-locked without defeating their own purpose. The
mitigation here is to make the window in which that live check can happen
as small as practically possible, and to remove retries entirely:

- `evaluate(bet_id)` is a **single-shot, permissionless** call, only valid
  inside `LIVE_CHECK_WINDOW_SECONDS` (5 minutes) of `deadline_ts`.
- There is **no retry function** for these two condition types. A retry
  would just be a second, later fetch of the same mutable URL — exactly the
  pattern that must be avoided. If the one attempt can't reach the artifact,
  the bet settles `RESOLVED_INCONCLUSIVE` **in that same call**.
- Miss the 5-minute window entirely (nobody called `evaluate` at all)?
  Anyone may call `force_inconclusive(bet_id)` to refund both sides.

This does not make manipulation *impossible* — a party could still edit
their page within that 5-minute window — but it shrinks the exploitable
window from "unbounded" to "as tight as the primitive's own liveness
semantics allow." This tradeoff, and why it can't be eliminated further
without changing what these two condition types mean, is documented in
`REVIEW.md`.

### Content/image conditions — `LLM_CONTENT`, `IMAGE_VISUAL`

These reference a document or image whose *exact byte content* matters, so
they get a much stronger guarantee: a two-phase snapshot-then-judge flow.

1. **`commit_snapshot(bet_id)`** — permissionless, only within
   `SNAPSHOT_WINDOW_SECONDS` (30 minutes) of `deadline_ts`. Every validator
   independently fetches the artifact and computes its SHA-256; consensus
   (`strict_eq`) requires them to all get the *exact same bytes*, which
   doubles as a genuine "is this content currently stable" check. On
   success, the hash is pinned to the bet (`status` → `SNAPSHOTTED`).
   Unreachable? One retry via `retry_snapshot(bet_id)` after a short delay,
   still bounded by the same window; exhausted → `RESOLVED_INCONCLUSIVE`.
2. **`evaluate(bet_id)`** — only valid once `SNAPSHOTTED`, and **no longer
   time-bound** at all, because it's hash-locked instead: it re-fetches and
   requires the content to still hash to the pinned value before judging.
   Any drift since the snapshot → `RESOLVED_INCONCLUSIVE` immediately,
   **never** judged against the newer, unpinned content.

## Pre-deployment checks (already run, re-run after any edit)

```bash
genvm-lint check contracts/disputed_deadline.py --json
genvm-lint typecheck contracts/disputed_deadline.py --json
genvm-lint schema contracts/disputed_deadline.py --json
pytest tests/direct/ -v
```

All four currently pass clean (0 lint errors, 0 typecheck errors, valid ABI
schema, 31/31 direct tests). A clean `schema` extraction is what prevents the
StudioNet "could not load contract schema" deploy-time error — if you modify
the contract, re-run it before deploying.

## Deploying (genlayer-cli)

```bash
genlayer deploy --contract contracts/disputed_deadline.py --network studionet
```

Follow the CLI's interactive prompts for the funded account to deploy from.
(StudioNet is gasless — a 0 GEN balance does not block deploy or writes.)

## Core interaction flow

### 1. Create a bet (creator stakes GEN)

```
create_bet(
  title="Demo ships a working /health endpoint",
  side="CONDITION_MET",                 # or "CONDITION_NOT_MET"
  artifact_url="https://myapp.example.com/health",
  condition_type="HTTP_STATUS",         # HTTP_STATUS | STRING_CONTAINS | LLM_CONTENT | IMAGE_VISUAL
  condition_text="",                    # optional human label; required for LLM_CONTENT/IMAGE_VISUAL
  deadline_ts=<unix ts, >=5 min out>,
  expected_http_status=200,             # only used by HTTP_STATUS
  expected_substring="",                # only used by STRING_CONTAINS
  expected_image_description="",        # only used by IMAGE_VISUAL
  join_window_seconds=0,                # 0 = default 3-day join window
)
```
Attach the stake as the transaction's native value (GEN). Returns `bet_id`.
`join_window_seconds` must close well before `deadline_ts`, and
`deadline_ts` itself must leave enough room for the relevant check window
below (5 min for live-status types, 30 min for content/image types).

**Condition type selection:**
- `HTTP_STATUS` / `STRING_CONTAINS` — fully deterministic, no LLM call, cheapest and most reliable. Use whenever the claim reduces to a status code or a literal substring, and the check is genuinely about *live* server state.
- `LLM_CONTENT` — the fetched page needs interpretation beyond a substring match (e.g. "the page's UI shows the feature is live"). Content is hash-locked once snapshotted.
- `IMAGE_VISUAL` — `artifact_url` must resolve directly to image bytes; validators run vision interpretation against `expected_image_description`. Also hash-locked once snapshotted.

### 2. Counterparty joins (matches the stake exactly)

```
join_bet(bet_id)
```
Attach exactly the same GEN amount as the creator staked. This pins the bet
into `ACTIVE` — `artifact_url` / `condition_text` / `deadline_ts` become
immutable from this point on.

### 3a. HTTP_STATUS / STRING_CONTAINS — evaluate within 5 minutes of the deadline

```
evaluate(bet_id)
```
Reverts if called before `deadline_ts` or more than `LIVE_CHECK_WINDOW_SECONDS`
(5 min) after it. Single-shot: settles `RESOLVED_CONDITION_MET`,
`RESOLVED_CONDITION_NOT_MET`, or — if the artifact couldn't be reached —
`RESOLVED_INCONCLUSIVE`, all in this one call. There is no retry function
for these condition types. Missed the window entirely? Call
`force_inconclusive(bet_id)` instead.

### 3b. LLM_CONTENT / IMAGE_VISUAL — snapshot, then judge

```
commit_snapshot(bet_id)     # within 30 min of deadline_ts
```
Pins a validator-agreed content hash. If unreachable, wait a short delay
and call `retry_snapshot(bet_id)` (one retry, still bounded by the 30-minute
window); if that also fails, the bet auto-settles `RESOLVED_INCONCLUSIVE`.
Missed the window without ever snapshotting? Call `force_inconclusive(bet_id)`.

Once `SNAPSHOTTED`:
```
evaluate(bet_id)
```
No longer time-bound. Re-fetches, requires the content hash to still match,
then runs LLM/vision judgment. A hash mismatch settles
`RESOLVED_INCONCLUSIVE` immediately (evidence drifted, never judged against
the newer state). A transient/model hiccup unrelated to drift may be
retried by calling `evaluate` again (bounded by attempt count, not time).

### 4. Withdraw your balance (pull-based, not automatic)

Settlement only credits an internal ledger — nobody's wallet is touched by
`evaluate`/`commit_snapshot` themselves. Each party must withdraw:

```
withdraw()
```
Pays out the caller's entire `withdrawable_wei` balance and zeroes it first.

### Escape hatches

```
cancel_bet(bet_id)               # creator only, before anyone joins
timeout_unjoined_reclaim(bet_id) # anyone, after join_deadline_ts if nobody joined
force_inconclusive(bet_id)       # anyone, once the relevant window has closed
                                  # without a terminal outcome
```

## Views (read-only, free)

```
get_bet(bet_id)              # full bet record, including snapshot_content_hash
get_bet_summary(bet_id)      # status/stake/outcome only
get_bet_count()
get_party_bet_ids(address)   # bets an address is party to
get_withdrawable(address)    # pending pull-payout balance
get_activity(bet_id, offset, limit)  # event log
get_platform_stats()
get_config()                 # all tunable constants, for client-side display
```

## Testing before you stake real GEN

Run the direct-mode suite (fast, no network, no validator consensus):

```bash
pytest tests/direct/ -v
```

31 tests cover: min-stake/lead-time validation, join/cancel/timeout
lifecycle, both deterministic condition types (met + not-met), the
single-shot no-retry live-status window, LLM-content evaluation, the
adversarial-content-injection mitigation, image-visual evaluation, the
commit_snapshot → evaluate hash-lock flow for both content/image types, the
evidence-drift → immediate-INCONCLUSIVE path (the exact scenario the
deadline-grounding review finding described), early/duplicate `evaluate()`
rejection, structural guarantees (no self-reported evidence function, no
retry function for live-status types), and fund conservation across every
terminal path.

Direct mode does **not** exercise real validator disagreement/consensus
rotation — that requires a live GenLayer network with multiple validators,
which is what every deployment above has been tested against on StudioNet.

## What to send back after deployment

Once you deploy, send the deployed contract address and confirm the
network (StudioNet). Full interaction flow above can then be exercised
end-to-end with real GEN stakes.
