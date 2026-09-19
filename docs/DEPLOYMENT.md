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

## Pre-deployment checks (already run, re-run after any edit)

```bash
genvm-lint check contracts/disputed_deadline.py --json
genvm-lint typecheck contracts/disputed_deadline.py --json
genvm-lint schema contracts/disputed_deadline.py --json
pytest tests/direct/ -v
```

All four currently pass clean (0 lint errors, 0 typecheck errors, valid ABI
schema, 22/22 direct tests). A clean `schema` extraction is what prevents the
StudioNet "could not load contract schema" deploy-time error — if you modify
the contract, re-run it before deploying.

## Deploying (GenLayer Studio UI)

1. Open GenLayer Studio, connect a StudioNet wallet funded with test GEN.
2. Upload `contracts/disputed_deadline.py` as a new contract.
3. Deploy with no constructor arguments (`__init__` takes none).
4. Note the deployed contract address — you'll need it for every call below.

## Deploying (genlayer-cli)

```bash
genlayer deploy --contract contracts/disputed_deadline.py --network studionet
```

Follow the CLI's interactive prompts for the funded account to deploy from.

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

**Condition type selection:**
- `HTTP_STATUS` / `STRING_CONTAINS` — fully deterministic, no LLM call, cheapest and most reliable. Use whenever the claim reduces to a status code or a literal substring.
- `LLM_CONTENT` — the fetched page needs interpretation beyond a substring match (e.g. "the page's UI shows the feature is live").
- `IMAGE_VISUAL` — `artifact_url` must resolve directly to image bytes; validators run vision interpretation against `expected_image_description`.

### 2. Counterparty joins (matches the stake exactly)

```
join_bet(bet_id)
```
Attach exactly the same GEN amount as the creator staked. This pins the bet
into `ACTIVE` — `artifact_url` / `condition_text` / `deadline_ts` become
immutable from this point on.

### 3. Wait for the deadline, then evaluate (anyone can call)

```
evaluate(bet_id)
```
Reverts if called before `deadline_ts`. Runs validator consensus over the
pinned artifact/condition and settles deterministically. Returns
`{status, condition_met, sub_checks}` (or `{status: "UNDETERMINED", ...}` if
the artifact was unreachable).

### 4. If UNDETERMINED: one automatic retry

```
retry_evaluate(bet_id)   # only after UNREACHABLE_RETRY_DELAY_SECONDS (30 min)
```
If this also fails to reach the artifact, the bet auto-settles to
`RESOLVED_INCONCLUSIVE` (full refund both sides) — no third attempt exists.

If nobody calls `retry_evaluate` at all within
`UNREACHABLE_RETRY_MAX_WAIT_SECONDS` (3 days) of the first attempt, anyone
may call:
```
force_inconclusive(bet_id)
```

### 5. Withdraw your balance (pull-based, not automatic)

Settlement only credits an internal ledger — nobody's wallet is touched by
`evaluate`/`retry_evaluate` itself. Each party must withdraw:

```
withdraw()
```
Pays out the caller's entire `withdrawable_wei` balance and zeroes it first.

### Escape hatches

```
cancel_bet(bet_id)               # creator only, before anyone joins
timeout_unjoined_reclaim(bet_id) # anyone, after join_deadline_ts if nobody joined
```

## Views (read-only, free)

```
get_bet(bet_id)              # full bet record
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

It covers: min-stake/lead-time validation, join/cancel/timeout lifecycle,
both deterministic condition types (met + not-met), LLM-content evaluation,
the adversarial-content-injection mitigation (deterministic anchor layer
overriding a manipulated LLM verdict), image-visual evaluation, the
unreachable → retry → INCONCLUSIVE path, early/duplicate `evaluate()`
rejection, the "no self-reported evidence function exists" structural
guarantee, and fund conservation across every terminal path.

Direct mode does **not** exercise real validator disagreement/consensus
rotation — that requires an integration-mode run against a live GenLayer
network with multiple validators, which is what you'll be doing once you
deploy to StudioNet and hand me the contract address.

## What to send back after deployment

Once you deploy, send me:
1. The deployed contract address
2. The network (StudioNet)

I'll then run through the interaction flow above end-to-end with real GEN
stakes to confirm: bet creation, joining, deadline enforcement, evaluation
consensus on at least the `HTTP_STATUS` and one nondeterministic condition
type, the INCONCLUSIVE/retry path, and pull-based withdrawal.
