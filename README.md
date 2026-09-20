# Disputed Deadline

A reusable GenLayer Intelligent Contract primitive for adversarial
deadline-based bets:

> **Two parties agree a task must be publicly visible by a deadline. Neither
> one gets to decide alone whether it was.**

Two parties stake equal native GEN on opposite sides of the claim
`"Artifact A will satisfy condition C by deadline block-time T."` At or
after T, GenLayer validators **independently fetch artifact A themselves**
(a web page, or an image the artifact resolves to) and **independently
evaluate condition C**. They reach Equivalence-Principle consensus on a
structured result — never on free text — and a fully deterministic
settlement function pays the correct side out of the shared pot.

This is a standalone Intelligent Contract only: no frontend, no backend, no
database, no authentication system. The deliverable is one deployable
`.py` contract, its direct-mode test suite, and this documentation.

**Current verified deployment on GenLayer StudioNet:**

```
0x151b73847663116eab8822749031d56Fb86a82bE
```

This is the fourth deployment, after an external review flagged a real
deadline-grounding gap and two rounds of live-tested fixes. See
[`REVIEW.md`](REVIEW.md) for the complete history: what was flagged, root
cause, the fix, and live re-verification for every round.

## Contents

- [`contracts/disputed_deadline.py`](contracts/disputed_deadline.py) — the contract (~1,670 lines)
- [`tests/direct/test_disputed_deadline.py`](tests/direct/test_disputed_deadline.py) — 31-case direct-mode test suite
- [`docs/DEPLOYMENT.md`](docs/DEPLOYMENT.md) — full StudioNet interaction guide
- [`REVIEW.md`](REVIEW.md) — full record of external review findings and fixes

## Why this needs GenLayer

The core trust problem: two adversarial parties both have an incentive to
misreport what actually happened at the deadline. A centralized backend or
either party's own claim cannot be trusted as the source of truth. GenLayer
validators independently fetching and evaluating the pinned artifact —
then reaching consensus on a structured, code-comparable result — is what
makes an unbiased, non-self-reported outcome possible at all.

Nothing here is a "GenLayer wrapper for storage": the contract has no
`submit_evidence`-style function anywhere. The only evidence path is
validators fetching the pinned URL themselves, inside a nondeterministic
block, at or after the deadline.

## Trust boundary

- **Pinned at creation, immutable afterward:** `artifact_url`,
  `condition_type`, `condition_text`, `deadline_ts`. Neither party can move
  the goalposts after staking.
- **No self-reported evidence.** No function anywhere lets a party submit a
  screenshot, a claimed response, or any payload as "proof."
- **Evidence is grounded to the deadline, not to whenever a transaction
  happens to land** (see `REVIEW.md` for the finding that drove this):
  - `HTTP_STATUS` / `STRING_CONTAINS` — `evaluate()` is a **single-shot,
    no-retry** call, only valid within `LIVE_CHECK_WINDOW_SECONDS` (5 min)
    of the pinned deadline. An unreachable artifact settles
    `RESOLVED_INCONCLUSIVE` in that same call; there is no later re-fetch.
  - `LLM_CONTENT` / `IMAGE_VISUAL` — a `commit_snapshot()` step, itself
    bounded to a tight post-deadline window, pins a validator-agreed
    SHA-256 of the raw fetched bytes. `evaluate()` is then hash-locked (not
    time-bound): it re-fetches and requires the exact hash to still match
    before judging. Any drift since the snapshot settles
    `RESOLVED_INCONCLUSIVE` immediately — the newer content is never judged.
- **Consensus is over a structured result**, never raw text:
  `{condition_met: bool, checked_at: int, sub_checks: [...]}`. For the two
  deterministic condition types this uses `strict_eq`. For the two
  nondeterministic types this uses a custom `run_nondet_unsafe` validator
  that reruns the fetch and inference independently and compares the
  **decision field** exactly, plus a plain-Python deterministic anchor
  layer where a literal substring is quoted in the condition text.
- **Deterministic settlement only.** `_apply_outcome` /
  `_apply_inconclusive_refund` contain zero nondeterministic calls; the
  nondeterministic step decides a boolean, the deterministic step moves
  money, and they are architecturally separated in the source.
- **Adversarial content resistance.** The LLM/vision step is constrained to
  narrow factual extraction against a pinned condition, explicitly told to
  treat any instruction-like text embedded in fetched content as untrusted
  data, and — for `LLM_CONTENT` — capped by a plain-Python literal-substring
  check the model cannot argue its way around.
- **Pull-based withdrawal, never push.** Settlement only credits an internal
  `withdrawable_wei` ledger; a separate `withdraw()` call performs the
  actual transfer, following zero-then-transfer ordering.
- **No re-trigger drift.** Once a bet is settled, there is no path to
  re-run evaluation against a different fetch of the same URL.

## Known, documented tradeoff

`HTTP_STATUS` / `STRING_CONTAINS` check *live, dynamic* server state by
design, so they cannot be hash-locked the way `LLM_CONTENT`/`IMAGE_VISUAL`
are without defeating their purpose. The 5-minute window narrows the
manipulation surface as far as it can go without abandoning the liveness
semantic, but it does not eliminate it — a party could still edit their
page within that window. This tradeoff is deliberate and documented rather
than hidden; see `REVIEW.md` for the full reasoning.

## Live StudioNet verification

Every lifecycle path across four deployments (see `REVIEW.md` for the full
blow-by-blow) has been exercised with real staked GEN — **no admin or owner
writes were used anywhere**, because the contract has none.

| Path | Result |
|---|---|
| Create / join with real GEN stake | Escrow correct, ledger fields correct |
| Early / duplicate `evaluate()` | Rejected — validators independently agree |
| `HTTP_STATUS` (deterministic, `strict_eq`) | Real fetch of a live 200 endpoint, validators agreed, correct settlement |
| `LLM_CONTENT` (nondeterministic) | Real fetch + real LLM judgment against `docs.genlayer.com`; genuine `MAJORITY_AGREE` on a nuanced verdict |
| `IMAGE_VISUAL` (nondeterministic) | Real image fetch + real vision-model judgment; genuine `MAJORITY_AGREE` on a nuanced verdict |
| Unreachable artifact (live-status) | Single-shot `evaluate()` settles `RESOLVED_INCONCLUSIVE` immediately, no retry |
| Unreachable artifact (content/image) | `commit_snapshot()` → `UNDETERMINED` → `retry_snapshot()` → `RESOLVED_INCONCLUSIVE`, both stakes refunded |
| Fund conservation | Total withdrawable across both parties exactly matched total staked in every case |
| Pull-based `withdraw()` | Succeeded once per party; second call rejected — no double payout |
| `cancel_bet` / `timeout_unjoined_reclaim` | Both confirmed live, including permissionless-trigger correctness |

## Running the tests

```bash
genvm-lint check contracts/disputed_deadline.py --json
genvm-lint typecheck contracts/disputed_deadline.py --json
genvm-lint schema contracts/disputed_deadline.py --json
pytest tests/direct/ -v
```

See [`docs/DEPLOYMENT.md`](docs/DEPLOYMENT.md) for the full StudioNet
interaction flow and deployment guide, and [`REVIEW.md`](REVIEW.md) for the
complete review-and-fix history.
