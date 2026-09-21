# Appeal: Deadline-grounding fix — rejection point resolved and verified live

## Original rejection reason and its fix

> The contract cannot be accepted because settlement is not grounded to the
> pinned deadline. evaluate and retry_evaluate fetch the URL only when
> those later calls run, so a mutable page or image can change after the
> deadline and reverse which side receives the pot. A resubmission should
> make validators verify deadline-bound evidence, such as an
> immutable/versioned artifact with a verifiable publication time or a
> committed snapshot, and ensure retries evaluate that same evidence rather
> than a newer URL state.

`evaluate()`/`retry_evaluate()` had no upper bound on when they could run
after `deadline_ts`, and each retry was simply a fresh fetch of the same
mutable URL — so a party controlling the artifact could edit it any time
after the deadline but before a permissionless caller happened to invoke
`evaluate()`, and the contract had no way to distinguish "true at the
deadline" from "made true five minutes ago."

Fixed with two condition-type-specific mechanisms, because "the artifact"
means different things depending on what's being checked:

**`LLM_CONTENT` / `IMAGE_VISUAL`** — these reference a document or image
whose specific byte content is what's being judged, so they get a full
commit-then-hash-lock fix. A new `commit_snapshot()` step, callable only
within a bounded post-deadline window, has every validator independently
fetch the artifact and compute `sha256(raw_bytes)`; consensus
(`gl.eq_principle.strict_eq`) requires them to all compute the *exact same
hash* before it is pinned to the bet. `evaluate()` is then hash-locked, not
time-bound: it re-fetches and requires the hash to still match the pinned
value **before running any LLM/vision judgment** — the check happens first,
inside the same nondeterministic round, so drifted content never reaches
the model. A mismatch settles the bet `RESOLVED_INCONCLUSIVE` immediately,
with no retry, because a retry there would just observe yet another
arbitrary later state — exactly the pattern under review. This is a
complete closure of the finding for these two condition types.

**`HTTP_STATUS` / `STRING_CONTAINS`** — these check *live, dynamic* server
state by design ("is the demo up right now"), so they cannot be hash-locked
without contradicting their own purpose. The available mitigation is to
shrink the exploitable window as far as possible: `evaluate()` is now a
**single-shot, permissionless** call valid only within a **5-minute**
`LIVE_CHECK_WINDOW_SECONDS` after the deadline, and **the retry function
for these two condition types was removed entirely** — a retry would be
structurally indistinguishable from "fetch the mutable URL again, later."
If the one attempt cannot reach the artifact, the bet settles
`RESOLVED_INCONCLUSIVE` in that same call, never a later one. This is a
**narrowed mitigation, not a complete elimination** — see the "Known,
documented limitation" section below, stated as plainly here as in the
project's own review record, since this appeal should not overclaim what
was fixed for this condition family.

## Code

- Fix commit: https://github.com/zoefunds/disputed-deadline/commit/5200db404080a7fba99a25207953efb4c1ec575b
- Contract source: https://github.com/zoefunds/disputed-deadline/blob/main/contracts/disputed_deadline.py
- Direct-mode test suite (31 tests, including a test that reproduces the exact evidence-drift scenario from the rejection and confirms it now settles fairly instead of being silently judged): https://github.com/zoefunds/disputed-deadline/blob/main/tests/direct/test_disputed_deadline.py
- Full review record (root cause, fix, before/after state machines, for this and two earlier findings): https://github.com/zoefunds/disputed-deadline/blob/main/REVIEW.md
- Deployment/interaction guide: https://github.com/zoefunds/disputed-deadline/blob/main/docs/DEPLOYMENT.md

## Live production verification

**Contract address (GenLayer StudioNet):**
`0x151b73847663116eab8822749031d56Fb86a82bE`

The StudioNet block explorer (`genlayer-explorer.vercel.app`) was returning
HTTP 503 at verification time, so no explorer hyperlinks are included below
— every transaction hash cited can be independently confirmed with
`genlayer receipt <hash>` against StudioNet, or via the CLI's `get_bet`
read calls against the contract address above.

### `LLM_CONTENT` hash-lock, working end to end

Real bet, real GEN (0.01 GEN/side), real fetch, real LLM judgment, real
multi-validator consensus on a stable static artifact
(`raw.githubusercontent.com/zoefunds/disputed-deadline/main/README.md`,
condition: page mentions "Disputed Deadline" as the contract name):

- `commit_snapshot` — tx `0x3102ed5a0611cffe04740ef10b8bcbcec28d0b983b729e8fafe5b7fcb3aa51d3`, `MAJORITY_AGREE`. Validators independently fetched the artifact and agreed on the exact same SHA-256: `76b047c6681bb9947dddaf5f391dbdd34f9acd9698d052a81354a36735dc6ecd`. Bet status → `SNAPSHOTTED`.
- `evaluate` — tx `0xa6581319b83e38a440361ca0e318391a080268362fcafbb2bced13558b19c0e9`, `MAJORITY_AGREE`. Re-fetched, confirmed the hash still matched, ran independent LLM judgment, and settled `RESOLVED_CONDITION_MET` with a real, non-scripted reasoning string ("The page prominently shows the heading 'Disputed Deadline,' which exactly matches the contract name specified in the condition...") and the deterministic literal-anchor sub-check (`literal_present:"Disputed Deadline"`) also passing.

### Strictness proof: the exact-hash-match requirement is real and enforced

An earlier attempt at the identical mechanism against a genuinely dynamic
page (`docs.genlayer.com`) produced two consecutive, independent
`MAJORITY_DISAGREE` results on `commit_snapshot` — direct evidence that
validators must actually reproduce byte-identical content for consensus to
succeed, not a check that rubber-stamps agreement:

- tx `0x88503b7f497a35a932b9430a9b0dd8a966b433c92a3845b602ab03c9de8f15af` — `MAJORITY_DISAGREE`
- tx `0x81735d1da5123dbe7b0832bdb90b79064d24a5d2b20bcb787578986b29249d88` — `MAJORITY_DISAGREE`

In both cases, `get_bet` confirmed **zero state mutation**: `status`
remained `ACTIVE` and `snapshot_attempts` remained `0` after each failed
transaction — funds stayed safe and the bet remained retryable, exactly as
designed. (The bet was recreated against the stable static artifact above
to complete the success-path proof.)

### `HTTP_STATUS` single-shot, no-retry window, working correctly

Real bet, real GEN, real 200-status fetch of `httpbin.org/status/200`:

- `evaluate` — tx `0x13d8b639a4294ee2dc541ec8eb98365650a7d3c4749b447876fb258f15069801`, `MAJORITY_AGREE`. Called 113 seconds after `deadline_ts` — well inside the 300-second window — settled `RESOLVED_CONDITION_MET` on the first and only attempt (`evaluate_attempts: 1`). No retry function exists for this condition type; the contract's ABI confirms `retry_evaluate` has been removed entirely.

### Known, documented limitation — stated plainly, not glossed over

The `HTTP_STATUS` / `STRING_CONTAINS` fix narrows the manipulation window
from unbounded to 5 minutes with no retry; it does not make manipulation
impossible. A party who controls the artifact could still make it briefly
compliant within that 5-minute window. This is an accepted, explicit
tradeoff rather than an oversight: a condition type whose entire purpose is
"check current live server state" cannot simultaneously guarantee
immutability of that state without ceasing to check live state at all. This
is documented in the same terms in `REVIEW.md`. Where full immunity to this
class of manipulation is required for a specific claim, the recommended
pattern is to model it as `LLM_CONTENT` or `IMAGE_VISUAL` against a
content-addressed, immutable reference instead, which receives the full
hash-lock protection demonstrated above.

---

Requesting re-review of main at commit
`5200db404080a7fba99a25207953efb4c1ec575b`, which includes the fix, its
direct-mode test suite, and this verification record.
