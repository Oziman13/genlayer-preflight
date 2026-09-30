# Preflight

A rule-based, on-chain review board built as a GenLayer Intelligent Contract.

One deployment = one review program (a grant round, a bounty board, a
contribution category, a hackathon track...). The deployer becomes the
**owner** and defines a **rubric**: an ordered list of plain-language rules,
each with a weight and a mandatory/advisory flag.

Anyone can then call `submit_for_review(url)` with a public URL. Validators
fetch the page and judge it against every rule, and the contract turns those
per-rule verdicts into a deterministic weighted score and a `READY` /
`NEEDS_WORK` decision, pinned to the exact rubric version it was judged
against.

## Why this is more than a thin LLM wrapper

- **Structured, multi-criteria judgment**, not a single yes/no. Every rule is
  judged independently as PASS/FAIL, then aggregated by weight.
- **Owner-only rubric management** with access control (`add_rule`,
  `update_rule`, `transfer_ownership`), so a program's standards are set
  deliberately and can evolve.
- **Rubric versioning.** Every submission stores the exact rubric version it
  was judged against, so changing the rules later never retroactively
  reinterprets a past decision.
- **A custom equivalence check**, not the built-in `strict_eq` or
  `prompt_comparative`. See below.
- **Deterministic aggregation.** Once the per-rule PASS/FAIL list is agreed
  on by consensus, scoring, mandatory-rule enforcement, and the final
  verdict are plain Python -- no further LLM judgment is trusted.
- **Explicit prompt-injection defense.** Fetched page content is wrapped as
  untrusted data with a direct instruction to ignore anything inside it that
  tries to redirect the model's task.
- **Never trusts a broken response.** A fetch failure or an unparseable
  model reply defaults the affected rule to FAIL -- a hostile or broken
  submission can never win by making the judge crash or omit an answer.

## Consensus design

A generic `gl.eq_principle.prompt_comparative` call asks "do these two
free-text answers mean the same thing", which is the wrong question for a
multi-rule review: two honest re-runs of the same rubric against the same
page will often flip one borderline rule while agreeing on the rest, and a
whole-answer equivalence check has no way to say "close enough".

Instead, Preflight defines its own leader/validator pair through
`gl.vm.run_nondet_unsafe`:

1. The **leader** judges every rule and returns a structured, index-aligned
   PASS/FAIL list.
2. Each **validator** independently re-fetches the same URL and re-judges
   the same rubric on its own.
3. A validator accepts the leader only if **enough of its own per-rule
   verdicts match the leader's** -- not all of them, and not just one.
   "Enough" is `ceil(80% of the rule count)`, so a 5-rule rubric tolerates
   one disagreement, while a 1- or 2-rule rubric still requires an exact
   match.

This is the "thoughtful equivalence check" a rubric-based reviewer needs:
consensus on stable, discrete outcomes, not on exact wording, and not on a
single coarse yes/no.

## Contract interface

**Rubric management** (owner only)
- `add_rule(text, weight, mandatory)`
- `update_rule(index, text, weight, mandatory)`
- `set_pass_threshold(percent)`
- `transfer_ownership(new_owner)`

**Review flow** (anyone)
- `submit_for_review(url) -> submission_id`

**Reads**
- `get_rubric_json()`, `get_rubric_version()`, `get_rule_count()`
- `get_pass_threshold()`, `get_owner()`, `get_program_name()`
- `get_submission_count()`, `get_submission_json(submission_id)`

## Storage note

Rules and submissions are kept in parallel `TreeMap`s of primitives
(`TreeMap[u32, str]`, `TreeMap[u32, u32]`) instead of a `TreeMap` of a
`@allow_storage` dataclass. Community reports show that assigning a freshly
built dataclass into a `TreeMap` slot can fail with a storage serialization
error in Studio; primitive maps are the proven, boring alternative.

## Testing

Tests run with [`genlayer-test`](https://pypi.org/project/genlayer-test/) in
**direct mode** -- fully in-memory, no local GenVM node or testnet
required.

```
pip install genlayer-test
python -m pytest tests/direct -v
```

`tests/direct/test_preflight_rubric.py` covers the rubric registry: owner
access control, rule validation, ownership transfer (including address
canonicalization), and rubric versioning.

`tests/direct/test_preflight_review.py` covers the review flow: input
validation, deterministic score/verdict aggregation, mandatory-rule
overrides, malformed-LLM-output fallback, rubric-version pinning, the
prompt-injection wrapper actually being sent, and -- using `mock_web`,
`mock_llm`, and `run_validator` -- the custom 80%-agreement consensus logic
itself: accepting a validator that disagrees on one rule out of five,
rejecting one that disagrees on two, and requiring an exact match on a
2-rule rubric.

28/28 tests pass.

## Deployed on GenLayer Studio testnet

Constructor: `Preflight(program_name: str, pass_threshold_percent: int)`

A live instance, configured for the Portal's own "Intelligent Contracts"
rubric (source readable, README explains consensus, has automated tests),
is linked in the GitHub repo description / Portal submission evidence.
