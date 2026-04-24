---
layout: post
title: "Wiring Mutation Testing into an Autonomous Coding Loop"
date: 2026-04-19
related:
  - /2026/04/10/multi-agent-tdd-loop/
  - /2026/04/23/final-validator-ralph-loop/
---

This post describes how I wired PIT mutation testing into the [multi-agent Ralph loop](https://ranjithkannan.com/2026/04/10/multi-agent-tdd-loop/) as an automated gate. It assumes you're familiar with mutation testing and why it matters. What's new here is turning PIT from an interactive analysis tool into a deterministic gate inside an agent pipeline.

## The Problem

In the Ralph loop, three LLM-graded gates check each implementation task: TEST-FIRST, REQ-CHECK, and PRE-CR. When all three agree the work is good, the loop moves on. Bugs still ship.

The blind spot is structural. If the worker weakened an assertion, the reviewer reads the weakened assertion and agrees it matches the spec, because it does, on the narrow interpretation the assertion expresses. The LLM that wrote the test and the LLM that reviewed it share the same blind spot.

PIT solves this because its mutations aren't authored by an LLM. They come from bytecode transformations the model can't see or pre-empt. A test that passes REQ-CHECK but survives a boundary mutation was never really testing the boundary. PIT surfaces that.

## Where PIT Fits

The existing per-task structure for a Java implementation task:

```
- [ ] 2.TEST-FIRST  → ralph-test-writer writes failing tests
- [ ] 2.            → ralph-worker makes them pass, commits
- [ ] 2.REQ-CHECK   → ralph-req-reviewer: tests cover requirements?
- [ ] 2.PRE-CR      → ralph-quality-reviewer: code quality OK?
```

PIT inserts between REQ-CHECK and PRE-CR:

```
- [ ] 2.TEST-FIRST  → ralph-test-writer
- [ ] 2.            → ralph-worker
- [ ] 2.REQ-CHECK   → ralph-req-reviewer
- [ ] 2.PITEST      → ralph-pit-gate.sh (shell, no LLM)
- [ ] 2.PRE-CR      → ralph-quality-reviewer
```

The ordering is deliberate. REQ-CHECK runs first because it's cheap and catches missing tests. There's no point running PIT against a test suite that doesn't cover the requirements. PRE-CR runs last, which means the quality reviewer reads test code that has already been strengthened against mutations. Higher signal than reviewing tests PIT hasn't scrutinized yet.

## Task-Scoped Mutation Runs

Running PIT against an entire package takes three to five minutes. A twenty-task spec running full-package PIT on every task would add over an hour. The gate needs per-task scoping.

PIT supports `-DtargetClasses=<fqcn,...>` to mutate only named classes. The question is how to determine the right list without asking an LLM.

The answer: derive the list from the task's actual git commit, not from the task description or the filesystem. After each agent invocation, the loop records which files changed in that iteration. The gate reads that record. The loop script is the only component that needs to observe git state.

## State Tracking

Before each agent invocation, the loop snapshots each package's HEAD SHA:

```bash
STATE_DIR="$(pwd)/.ralph/state"
BEFORE_STATE="$STATE_DIR/iter-${ITERATION}-before.txt"
for pkg in "$(pwd)"/src/*/; do
    [ -d "$pkg/.git" ] || continue
    echo "$(basename "$pkg"):$(git -C "$pkg" rev-parse HEAD)" >> "$BEFORE_STATE"
done
```

After the agent exits, the loop diffs each package between the pre-invocation SHA and current HEAD. Files that changed get written to `.ralph/state/latest-task-diff.txt`:

```
OrderService/src/main/java/.../CreateOrderAction.java
OrderService/src/test/java/.../CreateOrderActionTest.java
```

This is derived purely from git. It's ground truth. The worker commits exactly what it changed, and the diff sees it. No reliance on commit message parsing, no guessing which package was touched.

## The Gate's Scope Rule

`ralph-pit-gate.sh` consumes `latest-task-diff.txt`:

1. Read the file. Group entries by package (first path segment).
2. For each package with production Java (`src/main/java/*.java`): for each production class `Foo.java`, check whether `FooTest.java` exists anywhere under `*/src/test/java/**`. If yes, include the FQCN in the target list.
3. Skip any package where the target list is empty.
4. Run `mvn pitest:mutationCoverage -DtargetClasses=<comma-joined>` in each package that has targets.
5. Parse the XML report. Coverage = killed / (killed + survived).
6. If every package met the threshold, PASS. Otherwise FAIL.

The "matching `*Test.java` exists" filter is the scope rule. It handles exclusions without explicit lists: POJOs without tests auto-skip, constants auto-skip, CDK TypeScript auto-skips because it's not Java. If a new class type appears that shouldn't be mutation-tested, the convention that untested code doesn't need mutation testing handles it without a build file change.

## The Gate Contract

The gate has three outcomes.

**PASS**: mutation coverage ≥ 70% across all target classes. Mark `N.PITEST` as `[x]` in tasks.md, append `PITEST PASS — <coverage>% (<killed>/<total>)` to progress.txt, exit 0. Loop moves on to `N.PRE-CR`.

**FAIL**: at least one target class fell below threshold. Append the HTML report path to progress.txt so the worker can read it directly on the next iteration. Uncheck `N.` (the impl task) so the worker picks it up. Leave `N.PITEST` unchecked so the gate re-runs after the worker commits. Exit 1.

**SKIP**: no production Java in the diff, or no matching tests. Mark `N.PITEST` as `[x]` with a SKIP log entry explaining why. Exit 0.

The FAIL path is the one that matters. When the gate unchecks the impl task, the loop routes the next iteration back to the worker. The worker reads progress.txt, sees `PITEST FAIL`, opens the report, and strengthens specific assertions by line number. No LLM judgment about what strengthening means: the surviving mutations are listed in the report with file, line, and mutator type.

## Why the Gate Is a Shell Script

Every decision the gate makes is a deterministic operation on the XML report: count killed mutations, count survived mutations, compute coverage, compare to threshold. No language model is involved because none is needed.

An LLM in this position would also create a perverse incentive. If the model knows its coverage score is being watched, it could be tempted to write tests that kill specific mutations without genuinely proving behavior. The deterministic gate closes that loophole. The threshold is a number, not a judgment.

This is the same reasoning behind every shell gate in the loop. The agents produce artifacts. The gates grade them.

## Placement Options

Not every spec wants PIT on every task. CDK-heavy specs produce mostly TypeScript; mutation testing is irrelevant there. Early prototype specs may want to move fast without the overhead.

The loop's spec steering asks at creation time where PITEST gates should be placed:

1. After every impl task with a TEST-FIRST sibling (recommended)
2. After specific tasks
3. After each logical group
4. Only at the end
5. Skip for this spec

Option 1 matches the test-first discipline: if the worker was required to write tests first, those tests should withstand mutation. The gate auto-skips non-applicable tasks, so option 1 is cheap even on specs with a mix of Java and TypeScript tasks. The PITEST steps log SKIP and move on without adding wall-clock time.

## Runtime Cost

Measured on a single-package Java service:

| Task shape | Classes mutated | Wall-clock |
|---|---|---|
| Single new class + test | 1 | ~30–60s |
| Class + helper + util | 2–3 | ~1–2 min |
| Whole-package (no scope flag) | ~50 | ~3–5 min |

Per-task scoping is worth the engineering. On a twenty-task spec with PIT on every impl task, scoped runs add fifteen to twenty minutes of total wall-clock. Full-package runs would add over an hour.

## What This Doesn't Replace

PIT validates that tests are strong enough to distinguish the current implementation from close variants. It does not catch requirements the implementation doesn't address. If the spec says "validate input" and the implementation doesn't validate, there's no mutation to survive — the original behavior is the bug. REQ-CHECK catches this.

It doesn't catch cross-component contract mismatches. Two components reading the same JSON path with different field names is invisible to PIT. The Final Validator catches this.

It doesn't catch integration-level behavior. PIT runs only the unit test suite. The Integration Tester catches what unit tests can't.

PIT is one layer. The others remain.

---

*This post is part of a series on the multi-agent Ralph loop. The [main post](https://ranjithkannan.com/2026/04/10/multi-agent-tdd-loop/) covers the full system architecture.*
