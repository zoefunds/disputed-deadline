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

**Live, verified deployment on GenLayer StudioNet:**

```
0xc8fe0Ea32E6848b7D5Be9C2Ee331c5d547261463
```

## Contents

- [`contracts/disputed_deadline.py`](contracts/disputed_deadline.py) — the contract (~1,300 lines)
- [`tests/direct/test_disputed_deadline.py`](tests/direct/test_disputed_deadline.py) — 23-case direct-mode test suite
- [`docs/DEPLOYMENT.md`](docs/DEPLOYMENT.md) — full StudioNet interaction guide and live-test log

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

## Trust boundary (what's pinned, what's independent, what's deterministic)

- **Pinned at creation, immutable afterward:** `artifact_url`,
  `condition_type`, `condition_text`, `deadline_ts`. Neither party can move
  the goalposts after staking.
- **No self-reported evidence.** No function anywhere lets a party submit a
  screenshot, a claimed response, or any payload as "proof."
- **Deadline enforcement is deterministic.** `evaluate()` can only run
  at/after the pinned deadline, exactly once per attempt window. Early and
  duplicate calls are rejected explicitly (verified live: 5-6/6 validators
  independently agree on the rejection).
- **Consensus is over a structured result**, never raw text:
  `{condition_met: bool, checked_at: int, sub_checks: [...]}`. For the two
  deterministic condition types (`HTTP_STATUS`, `STRING_CONTAINS`) this uses
  `strict_eq` — no LLM call is made where none is needed. For the two
  nondeterministic types (`LLM_CONTENT`, `IMAGE_VISUAL`) this uses a custom
  `run_nondet_unsafe` validator that reruns the fetch and inference
  independently and compares the **decision field** exactly, plus a
  plain-Python deterministic anchor layer where a literal substring is
  quoted in the condition text.
- **Explicit INCONCLUSIVE path.** If the artifact is unreachable, the
  contract allows exactly one automatic retry after a delay before falling
  back to a defined `RESOLVED_INCONCLUSIVE` state that refunds both sides in
  full — never a default win for either party.
- **Deterministic settlement only.** `_apply_outcome` / `_apply_inconclusive_refund`
  contain zero nondeterministic calls; the nondeterministic step decides a
  boolean, the deterministic step moves money, and they are architecturally
  separated in the source.
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

## Live StudioNet test summary

Every lifecycle path and all four condition types were exercised end-to-end
on GenLayer StudioNet with real staked GEN (0.01 GEN/side) — **no admin or
owner writes were used anywhere**, because the contract has none.

| Path | Result |
|---|---|
| Create / join with real GEN stake | Escrow correct, ledger fields correct |
| Early `evaluate()` | Rejected — 5/5 validators independently agree |
| Duplicate `evaluate()` after settlement | Rejected |
| `HTTP_STATUS` (deterministic, `strict_eq`) | Real fetch of a live 200 endpoint, validators agreed, correct settlement |
| `LLM_CONTENT` (nondeterministic) | Real fetch + real LLM judgment against `docs.genlayer.com`; genuine `MAJORITY_AGREE` on a nuanced `NOT_MET` verdict |
| `IMAGE_VISUAL` (nondeterministic) | Real image fetch + real vision-model judgment; genuine `MAJORITY_AGREE` on a nuanced `NOT_MET` verdict (cropped rendering detected) |
| Unreachable artifact → retry → `INCONCLUSIVE` | First attempt `UNDETERMINED` (network-level block, classified `TRANSIENT`), retry after delay settled `RESOLVED_INCONCLUSIVE`, both stakes refunded |
| Fund conservation | Total withdrawable across both parties exactly matched total staked in every case |
| Pull-based `withdraw()` | Succeeded once per party; second call rejected — 6/6 validators agree, no double payout |
| `cancel_bet` (before anyone joins) | Refunded, ledger zeroed; duplicate cancel rejected (6/6 validators) |
| `timeout_unjoined_reclaim` | Called by an unrelated third party (proving it's permissionless); refund correctly routed to the original creator, not the caller |

### Bugs found and fixed during live testing

Two real, load-bearing bugs surfaced only once the contract was actually
staked against real StudioNet consensus — direct-mode mocked tests alone
could not have caught either:

1. **Address-argument coercion.** `get_withdrawable` / `get_party_bet_ids`
   crashed on-chain (`TypeError: cannot convert 'Address' object to bytes`).
   Client-side ABI encoders (genlayer-js, the `genlayer` CLI) auto-detect
   any 0x-prefixed 40-hex-char string argument and encode it as the GenVM
   `address` primitive regardless of the parameter's declared `str` type.
   Fixed with a `_to_address()` helper that accepts either an `Address`
   object or a plain string.
2. **Over-strict nondeterministic consensus.** The first live `evaluate()`
   on an `LLM_CONTENT` bet hit a genuine on-chain `MAJORITY_DISAGREE`. The
   validator comparison required LLM-generated sub-check *names* (free
   text) to match exactly between the leader and each validator's
   independent LLM call — but two independent calls routinely reach the
   same verdict through differently-worded checks. Fixed: consensus now
   compares only the `condition_met` boolean decision field plus the
   plain-Python deterministic anchor checks; the LLM's free-text
   explanation is stored for transparency but no longer gates agreement.
   Retested live and confirmed working for both `LLM_CONTENT` and
   `IMAGE_VISUAL`.

Both fixes are covered by the direct-mode test suite (23/23 passing) and
verified against real consensus after redeployment.

## Running the tests

```bash
genvm-lint check contracts/disputed_deadline.py --json
genvm-lint typecheck contracts/disputed_deadline.py --json
pytest tests/direct/ -v
```

See [`docs/DEPLOYMENT.md`](docs/DEPLOYMENT.md) for the full StudioNet
interaction flow and deployment guide.
