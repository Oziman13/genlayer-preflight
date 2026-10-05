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
multi-rule review: it has no notion of *which* disagreements matter.

Instead, Preflight defines its own leader/validator pair through
`gl.vm.run_nondet_unsafe`:

1. The **leader** judges every rule and returns a structured, index-aligned
   PASS/FAIL list.
2. Each **validator** independently re-fetches the same URL and re-judges
   the same rubric on its own.
3. A validator accepts the leader only if their verdicts agree on **every
   rule that can influence the stored result**. Any rule with `weight > 0`
   can change the score, the threshold outcome, or (when mandatory) force
   `NEEDS_WORK`, so a single disagreement on such a rule rejects the
   leader. Only **disabled rules (weight 0)** may differ, because the
   aggregation skips them entirely and they provably cannot change the
   outcome.

Consensus is therefore bound to the final result itself. An earlier
iteration accepted "at least 80% of rules match", which a reviewer
correctly pointed out could wave through a leader whose *mandatory*-rule
verdict contradicted the validator's own judgment (4 of 5 rules agree, the
disagreeing one is mandatory, so the leader stores `READY` while the
validator's judgment implies `NEEDS_WORK`). That counter-example is now a
regression test.

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
`mock_llm`, and `run_validator` -- the custom consensus logic itself: identical verdicts are accepted; any disagreement on a weighted rule is rejected; the reviewer's mandatory-rule counter-example (4 of 5 agree, the mandatory one doesn't) is rejected; and disagreement is tolerated only on disabled (weight 0) rules.

29/29 tests pass.

## Deployed on GenLayer Studio testnet

Constructor: `Preflight(program_name: str, pass_threshold_percent: int)`

A live instance, configured for the Portal's own "Intelligent Contracts"
rubric (source readable, README explains consensus, has automated tests),
is linked in the GitHub repo description / Portal submission evidence.
