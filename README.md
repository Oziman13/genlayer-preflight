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

## Live app

**Try it:** https://oziman13.github.io/genlayer-preflight/ (reading needs no wallet)

| | |
|---|---|
| Network | GenLayer Studionet |
| Chain ID | 61999 |
| RPC | https://studio.genlayer.com/api |
| Contract | `0x1B1838955D84b48d0b548C278759f58b0dD6ae83` |
| Explorer | https://explorer-studio.genlayer.com/address/0x1B1838955D84b48d0b548C278759f58b0dD6ae83 |

The app is a single static page (`index.html`) built on the official
`genlayer-js` SDK (loaded from jsDelivr). There is no backend: every number on
the page is read from the contract, and every verdict is written by it.

**How the flow works**
1. The page reads the program name, pass threshold, rubric and past
   submissions straight from the contract.
2. Connect a browser wallet. The page switches it to GenLayer Studionet.
3. Submit a public URL. The page estimates fees where the network supports it,
   sends `submit_for_review`, and waits for the validators' decision.
4. It then checks that the contract call itself succeeded (an accepted
   transaction alone does not prove that) and re-reads the submissions.
   Pending, success and failure states are all shown, and a failed submission
   can be retried.

**Sample results already on-chain**

| Submitted URL | Verdict | Score | Rule 0 (source public) | Rule 1 (README explains consensus) | Rule 2 (has tests) |
|---|---|---|---|---|---|
| this repository | READY | 100 | PASS | PASS | PASS |
| `genlayer-community-pulse` | NEEDS_WORK | 40 | PASS | FAIL | FAIL |

The second row is the point: the same rubric that approves a repository with a
README and tests rejects one without them.

**Run it locally:** open `index.html` in a browser. No build step.

**Known limitations**
- Studionet is a hosted development network and can be reset, which would
  clear the on-chain history shown in the app.
- Verdicts are LLM judgments against the rubric. They are not proof that the
  work is correct.
- A review takes a minute or more, because the page is fetched and judged by
  several validators. Only public pages can be reviewed.
- The app does not yet let the program owner edit the rubric; that is done
  through the contract methods directly.

**Roadmap:** deploy on Bradbury, add rubric editing for the owner, show a fee
quote before signing.

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

## Deploying your own

Constructor: `Preflight(program_name: str, pass_threshold_percent: int)`.
After deploying, the owner adds rules with `add_rule(text, weight, mandatory)`.
The live instance used by the app is listed under "Live app" above.
