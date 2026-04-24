---
layout: post
title: "Splitting Audit from Decision: The Final Validator"
date: 2026-04-23
related:
  - /2026/04/10/multi-agent-tdd-loop/
  - /2026/04/19/pit-mutation-testing-ralph-loop/
---

The [multi-agent Ralph loop](https://ranjithkannan.com/2026/04/10/multi-agent-tdd-loop/) uses six agents to implement, test, and review code autonomously. This post is about what happens at the end: how the loop decides it's actually done, and why the thing that checks completion shouldn't be the thing that decides it.

## The Problem We Hit

The original loop stopped when the worker said it was done. The worker's exit signal was a specific string in its output. If the string showed up, the loop exited.

This worked on small specs. On bigger ones, two failure modes kept showing up.

**False positives.** The worker would signal completion with tasks still unchecked, or with a requirement not actually implemented. It had written code that felt complete to it. The tests it wrote for that code passed. From its point of view the job was done. Nobody had cross-checked the work against the original spec.

**Cross-component drift.** On one spec, the worker wrote an action that returned `{success: true, details: {passed: false}}`. The CDK path extractor read `$.countResult.Payload.passed`, one level shallower than the action actually wrote. The unit tests passed because they ran against a mock that also used the shallow path. The worker's tests, the worker's mocks, and the worker's CDK all agreed with each other. They were all wrong. Integration deploy caught it.

Both failures have the same shape. A single agent writes code, writes the test for that code, grades its own output, and ships. When it's wrong, everything it produced is wrong in the same way. There's no independent check.

## What We Added

Two pieces, deliberately separated.

The **validator** is an LLM agent. It runs at the end, after every other task is checked. Its only job is to produce `validation-report.md`. It cannot mark anything complete, cannot signal the loop to stop, cannot edit source, cannot even read `progress.txt`.

The **gate** is a shell script. It runs after the validator. It reads the validator's report, reads `tasks.md`, reads git history, and decides completion deterministically.

The split matters. The validator's job is to be honest about what it finds. The gate's job is to be firm about what that means.

## What the Validator Produces

Three tables, each with cells that are either PASS, FAIL, or UNKNOWN. No other values, no qualifiers, no narrative.

**Requirements traceability.** One row per requirement. For each: which production file implements it, which test covers it, and a status. If there's no implementation, the row is FAIL. If there's an implementation but no test, UNKNOWN.

**Cross-component integrity.** This is the one that catches the drift bug. For every integration boundary in the spec, the validator traces the contract in both directions. Every field the consumer reads, it greps the producer to confirm the field is populated at that path with that name. Every field the producer writes, it checks a consumer actually reads it. Every mock, it diffs against the real implementation's output shape.

There are four questions it has to answer explicitly:

1. If every mock were swapped for the real implementation, would the existing tests still pass?
2. Is every field referenced in a condition actually populated upstream?
3. Is every field written by the producer actually consumed somewhere?
4. Do any two components name the same concept differently?

**Integration test integrity.** One row per integration test. Does the test exercise a real requirement? Are there weakened assertions? The validator flags things like `assertTrue(true)`, commented-out asserts, catch-all exception swallowing, and early returns that make a test pass without running its assertions.

## What the Validator Is Forbidden From Doing

These are hard rules in the system prompt:

It cannot read `progress.txt`. Every entry in that file was written by the agents the validator is auditing. Reading it primes the validator with the conclusions it's supposed to verify.

It cannot mark any task `[x]`, including FINAL. That decision belongs to the gate.

It cannot signal the loop to stop. The gate decides.

It cannot add FIX tasks. If the validator finds problems, a human should read the report and decide what to fix. Auto-generated FIX tasks would let the loop thrash on things that might actually be design issues.

It cannot edit source, tests, or config. Its output is exactly one file: `validation-report.md`.

The point is that the validator has no way to manipulate the outcome of its own audit. It can only describe what it found.

## What the Gate Does

Three inputs, one decision.

First, it figures out which files belong to the spec. For every package under `src/*/`, it runs:

```bash
git log origin/main..HEAD --grep="^<spec-name>:" --name-only
```

The union goes into `changed-files.txt`. This requires the convention that every agent commits with the prefix `<spec-name>:`. Without the prefix, files drop off the radar.

Second, it parses `validation-report.md` for FAIL and UNKNOWN rows.

Third, it counts unchecked non-FINAL tasks in `tasks.md`.

Then:

- Report has no FAIL or UNKNOWN, and every non-FINAL task is checked: mark FINAL as `[x]`, signal COMPLETE, exit 0. Loop ends.
- Report has gaps but all non-FINAL tasks are checked: mark FINAL as `[x]`, signal COMPLETE-WITH-GAPS, exit 0. Loop ends. The report stays on disk for the human.
- Non-FINAL tasks still unchecked: exit non-zero without marking FINAL. Loop continues.

## Why FINAL Is One-Shot

This was an intentional design choice. If the validator finds a real cross-component mismatch or a missed requirement, the cheapest fix is ten minutes of a human reading the report and adding the right FIX tasks. Letting the loop run another round against the same agents that produced the gaps costs money and time and usually doesn't help.

The loop knows when to hand back to a human. Full autonomy isn't the goal. Correct output is.

## Why We Split the Role

The first version had the validator both audit and decide. We moved away from it for two reasons.

**Conflict of interest.** If the validator's output can end the loop, the validator has an incentive to mark ambiguous things PASS. Saying "looks good" gets it out of a long audit faster than writing careful UNKNOWN rows. Splitting the decision removes that incentive. The validator is graded on honesty, not on outcome.

**Auditability.** The gate's logic is about 80 lines of shell. Anyone can read it and know exactly what rule was applied. If someone asks "why did the loop stop on iteration 47?", the answer is in the gate script and the validation report, not hidden in an LLM judgment call.

This is the same principle that runs through the rest of the loop: deterministic gates over probabilistic reviews. The LLM produces the evidence. The shell script renders the verdict.

## What This Actually Caught

The cross-component drift case, described above. The CDK reads `$.countResult.Payload.passed`. The action writes `$.countResult.Payload.details.passed`. The mock matches the CDK, not the action. Unit tests pass. The worker signals completion.

The validator's cross-component check greps the CDK path against what the action actually writes. Notices the mock matches the CDK but not the action. Flags the mock as masking the bug. Row marked FAIL. Gate signals COMPLETE-WITH-GAPS. Human reads the report, fixes the CDK path, adjusts the mock, reruns.

That bug would have cost a deploy cycle to find. The validator caught it on the dev desktop.

## Honest Limitations

The validator works off the changed-files list. If the spec required modifying a file that nobody modified, because the worker forgot it entirely and no task was written for it, the validator can still miss it. The requirements traceability table is the defense, but the validator has to notice the file is missing from the list.

Opus is not cheap. One final-validator run on a large report costs real money. We run it once per spec, gated by the shell script, which limits the cost.

The gate uses `origin/main` as the base for file collection. If the spec is on a branch that's out of sync, the file list can be incomplete. We haven't hit this in practice, but it's worth knowing.

## What This Cost to Build

One prompt file, roughly 150 lines. One shell script, roughly 120 lines. One routing change in the loop script. An entry in the spec steering so the FINAL task gets appended to every spec's `tasks.md`.

Existing agents unchanged. Existing prompts unchanged. The rest of the loop doesn't know the validator exists. All of the wiring is in two files.

---

*This post is part of a series on the multi-agent Ralph loop. See [the main architecture post](https://ranjithkannan.com/2026/04/10/multi-agent-tdd-loop/) and [wiring mutation testing into the loop](https://ranjithkannan.com/2026/04/19/pit-mutation-testing-ralph-loop/).*
