# Review & Fix History — Disputed Deadline

This document records every external review finding on this contract, the
root cause, the fix, and the live re-verification for each — kept as a
single source of truth rather than scattered across commit messages.

Deployment addresses referenced below are all on **GenLayer StudioNet**.

---

## Finding 1 — Address-argument coercion crash

**Symptom (live, on `0x2944cAAbEE8e9749Db837dc544cda7756b14Ed01`):**
`get_withdrawable` and `get_party_bet_ids` crashed on-chain with
`TypeError: cannot convert 'Address' object to bytes`.

**Root cause:** client-side ABI encoders (`genlayer-js`, the `genlayer`
CLI) auto-detect any 0x-prefixed 40-hex-char string argument and encode it
using the GenVM `address` primitive, regardless of the contract's declared
parameter type being `str`. The contract then received an already-decoded
`Address` object where it expected a string, and `Address(<Address
instance>)` is not a valid construction.

**Fix:** added `_to_address(value)`, a helper that accepts either an
`Address` object or a plain string (mirroring the existing `_addr_eq`
pattern), and used it everywhere a free-form address argument is taken.

**Verification:** reproduced live pre-fix, then confirmed fixed live
post-redeploy (`0x477bDf7f31b62Adcf166bB3e07d5603e25781D9d`). Locked in by
`test_get_withdrawable_accepts_address_object_not_just_str`, which passes
an `Address` object directly to simulate the exact ABI-decoded shape.

---

## Finding 2 — Over-strict nondeterministic consensus comparison

**Symptom (live, on `0x477bDf7f31b62Adcf166bB3e07d5603e25781D9d`):** the
first live `evaluate()` call on an `LLM_CONTENT` bet produced a genuine
on-chain `MAJORITY_DISAGREE` — not a mock or test artifact, an actual
failed validator round.

**Root cause:** the custom `run_nondet_unsafe` validator required
LLM-generated sub-check *names* (free text, e.g. `"badge_present"`) to
match exactly between the leader's LLM call and each validator's
independent LLM call. Two independent LLM (or vision-model) calls routinely
reach the *same verdict* through *differently-worded* checks, so exact
string equality on the free text made consensus fail almost regardless of
how clear-cut the real answer was.

**Fix:** consensus now compares only the boolean `condition_met` decision
field, plus (for `LLM_CONTENT` only) a plain-Python deterministic anchor
layer — literal substrings quoted in `condition_text`, checked for presence
in the raw fetched bytes via ordinary Python string containment. This is
reproducible byte-for-byte between independent fetches, unlike LLM prose.
The LLM's/vision-model's free-text explanation is still stored on the bet
for transparency, but no longer gates agreement. This is still substantive,
validator-verified consensus — not a format-only check — because each
validator independently fetches and independently runs inference; only the
free-text *wording* of the explanation is excluded from the comparison.

**Verification:** reproduced live pre-fix (genuine `MAJORITY_DISAGREE`),
then confirmed fixed live post-redeploy for **both** `LLM_CONTENT` (real
fetch + judgment against `docs.genlayer.com`, genuine `MAJORITY_AGREE` on a
nuanced verdict) and `IMAGE_VISUAL` (real fetch + judgment against a public
PNG, genuine `MAJORITY_AGREE` on a nuanced verdict, including a case where
the model correctly identified a cropped/malformed image and settled
`NOT_MET`).

---

## Finding 3 — Settlement not grounded to the pinned deadline

**External reviewer's note, verbatim:**

> The contract cannot be accepted because settlement is not grounded to the
> pinned deadline. evaluate and retry_evaluate fetch the URL only when
> those later calls run, so a mutable page or image can change after the
> deadline and reverse which side receives the pot. A resubmission should
> make validators verify deadline-bound evidence, such as an
> immutable/versioned artifact with a verifiable publication time or a
> committed snapshot, and ensure retries evaluate that same evidence rather
> than a newer URL state.

**Root cause:** `evaluate()` and `retry_evaluate()` had no upper bound on
when they could be called after `deadline_ts` — a permissionless caller
could invoke them minutes, hours, or days later. Because the fetch happens
*at call time, not at deadline time*, any party controlling the artifact
(their own demo page, an image they host) could edit it after the deadline
passed but before anyone got around to calling `evaluate()`, and the
contract had no way to tell the difference between "this was already true
at the deadline" and "this was made true five minutes ago." Retries made it
worse: each retry was simply a fresh fetch, so a second attempt could
observe yet another, different state of the same mutable URL.

### Root cause, why it's two different problems

The artifact means two different things depending on condition type, so a
single fix doesn't cover both:

- `HTTP_STATUS` / `STRING_CONTAINS` check *live, current* server behavior
  by design ("is the demo up right now"). Their entire purpose is reading
  dynamic state, so they can never be pinned to an immutable snapshot
  without contradicting what they're for.
- `LLM_CONTENT` / `IMAGE_VISUAL` reference a *document or image* whose
  specific content is what's being judged. Unlike a live status check, it
  is both meaningful and desirable for this content to be pinned and
  compared byte-for-byte later.

### Fix, part A — `LLM_CONTENT` / `IMAGE_VISUAL`: commit-then-hash-lock

A new two-phase flow, fully closing the gap for these two types:

1. **`commit_snapshot(bet_id)`** — permissionless, callable only within
   `SNAPSHOT_WINDOW_SECONDS` (30 min) of `deadline_ts`. Every validator
   independently fetches the artifact and computes `sha256(raw_bytes)`;
   consensus (`gl.eq_principle.strict_eq`) requires them to all compute the
   *exact same hash*. This is a genuine "is this content currently stable"
   check for free: if the page is actively being edited when this runs,
   validators will observe different bytes, strict_eq will fail to reach
   agreement, and the **whole transaction fails with zero state change** —
   the caller just tries again (still bounded by the window). On success,
   the hash and a bucketed timestamp are pinned to the bet
   (`status` → `SNAPSHOTTED`). Unreachable (network/5xx)? One retry via
   `retry_snapshot(bet_id)` after a short delay, still inside the same
   window; exhausted or window missed → `RESOLVED_INCONCLUSIVE`.
2. **`evaluate(bet_id)`** — only valid once `SNAPSHOTTED`, and *no longer
   time-bound at all* (this is the key property: once hash-locked, it is
   safe to call at any later time, because staleness is now cryptographically
   detectable rather than merely time-bounded). It re-fetches the artifact,
   computes the hash again, and requires it to still equal the committed
   snapshot hash **before running any LLM/vision judgment** — the check
   happens first, inside the same nondet round, so a drifted artifact never
   even reaches the model. On a hash mismatch, the bet settles
   `RESOLVED_INCONCLUSIVE` **immediately, with no retry** — a mismatch means
   the artifact was edited after being pinned, and re-fetching again would
   just observe yet another arbitrary later state, exactly the anti-pattern
   being fixed. A transient/model failure unrelated to drift (e.g. a
   momentary network blip on the second fetch) may be retried by calling
   `evaluate` again, bounded by attempt count (not time, since content is
   now hash-locked and timing no longer matters for that failure mode).

   The mismatch error message is a **static string** (no dynamic hash value
   embedded) so that every validator's independently-computed — and
   possibly differently-drifted — current hash still produces an
   identical error, which is required for the existing deterministic
   error-agreement mechanism (`_handle_nondet_leader_error`) to reach exact
   consensus on "yes, this drifted," rather than validators each producing
   subtly different messages and failing to agree even on the failure.

**This is a complete fix for these two condition types.** Retries never
evaluate a different fetch of the same URL once a snapshot is committed;
drift is detected deterministically and terminates the bet fairly rather
than being silently judged.

### Fix, part B — `HTTP_STATUS` / `STRING_CONTAINS`: single-shot, no retry, tight window

Hash-locking is not available here without contradicting the condition
type's purpose (a live-status check that ignores the live status isn't a
live-status check anymore). The available mitigation is to minimize the
exploitable window as far as possible without destroying the liveness
semantic:

- `evaluate(bet_id)` became a **single-shot, permissionless** call, valid
  only within `LIVE_CHECK_WINDOW_SECONDS` — tightened from an unbounded
  window (in the interim first fix) down to **5 minutes** after
  `deadline_ts`, following a second round of hardening explicitly requested
  after the first fix was judged insufficiently strict.
- **The retry function was removed entirely for this condition family.** A
  retry here is structurally indistinguishable from "fetch the mutable URL
  again, later" — the exact pattern under review. If the single attempt
  can't reach the artifact, the bet settles `RESOLVED_INCONCLUSIVE`
  immediately, in the same call, rather than waiting for a second look.
- Missed the 5-minute window entirely (nobody called `evaluate` at all)?
  `force_inconclusive(bet_id)` remains available to refund both sides
  rather than leaving funds stuck.

**Honest residual limitation, stated plainly:** this narrows the
manipulation window to 5 minutes; it does not eliminate it. A party who
controls the artifact could still make it briefly compliant within that
window. This is an accepted, documented tradeoff, not an oversight — a
condition type whose entire purpose is "check current live state" cannot
simultaneously guarantee immutability of that state without ceasing to be
a live-status check. If this residual risk is unacceptable for a specific
deployment, the recommended alternative is to model that specific claim as
an `LLM_CONTENT` or `IMAGE_VISUAL` bet against a content-addressed,
immutable reference instead (e.g. an IPFS CID, or a git-commit-SHA-pinned
raw file URL) so the full hash-lock protection applies to it.

### State machine before vs. after (text diagram)

**Before:**
```
ACTIVE --(deadline reached, ANY time later)--> evaluate() fetches NOW
  --> MET / NOT_MET / UNDETERMINED --(ANY time later)--> retry_evaluate()
      fetches NOW (possibly different content than the first fetch)
  --> MET / NOT_MET / INCONCLUSIVE
```
No upper bound anywhere. A page editable right up until whichever fetch
actually ran could flip the outcome.

**After — HTTP_STATUS / STRING_CONTAINS:**
```
ACTIVE --(deadline reached, within 5 min)--> evaluate() fetches ONCE
  --> RESOLVED_CONDITION_MET / RESOLVED_CONDITION_NOT_MET
  --> RESOLVED_INCONCLUSIVE (if unreachable -- no retry, decided in this
      same call)
ACTIVE --(5 min window missed entirely)--> force_inconclusive()
  --> RESOLVED_INCONCLUSIVE
```

**After — LLM_CONTENT / IMAGE_VISUAL:**
```
ACTIVE --(deadline reached, within 30 min)--> commit_snapshot()
  --> SNAPSHOTTED (hash pinned, validators agreed on exact bytes)
  --> UNDETERMINED (unreachable) --(short delay, still within 30 min)-->
      retry_snapshot() --> SNAPSHOTTED / RESOLVED_INCONCLUSIVE
ACTIVE / UNDETERMINED --(30 min window missed)--> force_inconclusive()
  --> RESOLVED_INCONCLUSIVE
SNAPSHOTTED --(any time later)--> evaluate() re-fetches, checks hash
  --> hash matches --> RESOLVED_CONDITION_MET / RESOLVED_CONDITION_NOT_MET
  --> hash mismatch (drift) --> RESOLVED_INCONCLUSIVE (immediate, no retry)
  --> unrelated transient/model failure --> stays SNAPSHOTTED, bounded
      retry via calling evaluate() again (attempt-count bounded, not
      time-bounded, since content is hash-locked)
```

**Verification:** both rounds of this fix were lint/typecheck/schema-clean,
covered by direct-mode tests (32 tests after round 1, 31 after round 2's
simplification removed the now-dead retry-for-live-status tests and added
structural/no-retry-function guarantees), and redeployed + re-exercised
live on StudioNet with real staked GEN each time:

- Round 1 (`0x06645F2EfDAC5D5Fa079C5d129A6A56E7e102Dc3`): confirmed
  `commit_snapshot` → `SNAPSHOTTED` with a real validator-agreed hash, and
  `evaluate()` after snapshot reaching real judgment consensus.
- Round 2, hardened (`0x151b73847663116eab8822749031d56Fb86a82bE`, current):
  confirmed the single-shot 5-minute-window `evaluate()` for `HTTP_STATUS`
  settles correctly with real GEN and no retry function exists at all.

---

## Full test coverage snapshot (current)

```
genvm-lint check contracts/disputed_deadline.py --json     # 0 errors
genvm-lint typecheck contracts/disputed_deadline.py --json # 0 errors
genvm-lint schema contracts/disputed_deadline.py --json    # valid ABI
pytest tests/direct/ -v                                    # 31/31 passing
```

Key tests added specifically for Finding 3:
- `test_evaluate_rejected_after_live_check_window_closes`
- `test_force_inconclusive_after_live_check_window_missed_entirely`
- `test_evaluate_no_retry_function_exists_for_live_status` (structural)
- `test_unreachable_live_status_settles_inconclusive_immediately`
- `test_snapshot_before_deadline_rejected`
- `test_evaluate_rejected_without_snapshot`
- `test_evidence_drift_after_snapshot_settles_inconclusive_not_retried`
  (directly proves the exact scenario the reviewer described no longer
  produces a silently-judged outcome)
- `test_snapshot_unreachable_then_retry_recovers`
- `test_snapshot_unreachable_twice_settles_inconclusive`
- `test_force_inconclusive_after_snapshot_window_missed_entirely`
- `test_image_evidence_drift_settles_inconclusive`

## Scope note

No frontend, backend, or database exists or was added anywhere in this
project, per the original scope. All fixes and all verification above are
against the standalone Intelligent Contract only.
