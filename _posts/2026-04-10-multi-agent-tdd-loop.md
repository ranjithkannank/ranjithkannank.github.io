---
layout: post
title: "Building a Multi-Agent TDD Loop for Autonomous Code Development"
date: 2026-04-10
related:
  - /2026/04/19/pit-mutation-testing-ralph-loop/
  - /2026/04/23/final-validator-ralph-loop/
---

Most AI coding workflows put one agent in charge of everything: write the code, write the tests, review the result. This post describes a different approach, a multi-agent TDD system with six specialized agents and competing incentives, that I've been running in production. The task execution is orchestrated by a [Ralph loop](https://ghuntley.com/ralph/), an autonomous bash loop that drives Claude Code against a task list until everything is done. The agents are what make it work.

## The Problem

When you ask an AI to write code and then review its own work, it's like asking a student to grade their own exam. The same blind spots that produced the code produce the review. The agent writes tests that fit its implementation, not tests that verify the requirements. If the implementation is wrong, the tests are wrong in the same way. Both agree, but both are wrong.

This isn't a hallucination problem. It's an incentive problem.

## The Insight: Game Theory

The fix isn't better prompts. It's better structure.

In game theory, a principal-agent problem occurs when one party acts on behalf of another but has different incentives. The classic solution is mechanism design: structuring the rules so that self-interest aligns with the principal's goals.

Applied to AI code generation: if you create agents with competing objectives in separate contexts, neither can cut corners because the other is watching. The payoff structure looks like this:

| Agent | Objective | Incentive |
|---|---|---|
| Test-Writer | Define what correct looks like | Writes from requirements before Worker exists |
| Worker | Make tests pass | Knows Reviewer will audit every line |
| Reviewer | Find violations | Has no fix burden, only reports |

In practice the system uses two reviewers, one for correctness and one for quality, but the incentive logic is the same for both. The Worker knows scrutiny is coming. The Reviewer has no reason to go easy.

This is a Nash equilibrium: neither agent benefits from changing its strategy.

## The Architecture: 6 Agents

The system uses six specialized agents, each running in a separate context with its own system prompt. They cannot see each other's instructions.

```
Requirements → Test-Writer → Worker → Correctness Reviewer → Quality Reviewer
                                                                     ↓ (pass)
                  Final Validator ← Integration Tester ← Deployed System
```

The **Test-Writer** reads the requirements and writes failing unit tests before any code exists. These tests define the contract: what correct looks like. The Worker must make them pass without modifying the test assertions.

The **Worker** implements the code to make the pre-written tests pass. It can add tests for edge cases it discovers, but cannot modify or delete the Test-Writer's tests. It follows the spec's design doc for architecture decisions.

The **Correctness Reviewer** verifies that unit tests prove the implementation matches the task requirements. It produces an evidence table mapping each sub-task to its test, and never writes code.

The **Quality Reviewer** audits code against the team's quality rules with adversarial intent. It uses severity scoring: high violations are blocking, medium and low violations accumulate points. If the score exceeds the threshold, the task bounces back to the Worker.

The **Integration Tester** writes and runs integration tests against the real deployed system. It has no knowledge of the implementation, only the requirements and API contract. It tests both prescribed scenarios (requirement verification) and creative scenarios: concurrency, wrong-state, rapid-fire transitions.

The **Final Validator** cross-references requirements, design, tasks, progress log, and actual code to verify all requirements are met. If deviations are found, it creates fix tasks that feed back into the loop. It uses the most capable model (Opus) because it runs once and the quality of that single validation matters most.

## Why Separate Contexts Matter

With a single agent "switching roles," the role switch is just a prompt instruction. The agent can choose to ignore it, and its context still contains everything it wrote before. With separate agents, each one has its own system prompt loaded independently and runs in a separate Claude Code session. The Worker literally cannot see the Reviewer's instructions. The Reviewer has zero incentive to downplay findings because it doesn't have to fix them. The Worker has full incentive to write clean code because every bounce-back costs a full iteration.

The separation isn't cosmetic. It's what makes the incentive structure real.

## TDD: Why Tests Before Code

The Test-Writer runs in a separate context before the Worker is dispatched. It reads only the requirements and writes failing tests. The Worker's only job is to make those tests pass.

The Worker is forbidden from modifying the Test-Writer's test assertions. It can add tests for edge cases, but the contract tests are immutable. The tests define what correct looks like, and the Worker must conform, not the other way around.

This closes the original incentive problem: the contract is defined before the Worker exists, by an agent that has never seen the implementation.

A natural objection at this point is that writing specs and tests before any code sounds like waterfall. It isn't, for two reasons. First, the specs are scoped to individual components, not the whole system. The process starts with a high-level design of the distributed system, then iteratively decomposes into smaller components. Each component gets its own spec when it's time to build it, not all upfront. Second, even within a single component, the spec is a living artifact. Marc Brooker [draws the distinction clearly](https://brooker.co.za/blog/2026/04/09/waterfall-vs-spec.html): spec-driven development is not waterfall because the spec iterates rather than getting locked before implementation starts. Every time a reviewer finds a violation and logs it to `progress.txt`, that's new information that sharpens the spec. The agents iterate against it. That's the opposite of freezing requirements upfront.

## The Task Flow

For each implementation task, the inner loop runs:

```
TEST-FIRST → Implement → REQ-CHECK → PRE-CR → (next task)
```

After all tasks complete and the system is deployed, the outer loop runs:

```
Deploy → INTEG → FINAL → (fix tasks if needed, repeat)
```

If any reviewer finds violations, the implementation task is unchecked and the Worker fixes it in the next iteration. The loop terminates when all tasks are checked and both the Integration Tester and Final Validator are satisfied.

## The Bounce-Back Mechanism

When a reviewer finds violations, it doesn't fix them. It logs the report to `progress.txt` and unchecks the implementation task. The next iteration dispatches the Worker, which reads the violations and fixes them. Then the reviewer re-audits.

```
Iteration 1:  Test-Writer      → writes failing tests → marks TEST-FIRST [x]
Iteration 2:  Worker           → implements (makes tests pass) → marks task [x]
Iteration 3:  Req-Reviewer     → audits correctness → PASS → marks REQ-CHECK [x]
Iteration 4:  Quality-Reviewer → audits quality → FAIL (score: 12/10)
                               → logs to progress.txt, unchecks task
Iteration 5:  Worker           → reads progress.txt, fixes violations → marks task [x]
Iteration 6:  Quality-Reviewer → re-audits → PASS (score: 4/10) → marks PRE-CR [x]
```

The Worker is explicitly told, in its system prompt, that each bounce-back costs a full iteration: time, tokens, and credits. This aligns with prospect theory. The Worker is loss-averse. The risk of losing an iteration is more motivating than the certainty of a harmless warning. If the Worker writes clean code upfront, knowing the audit is coming, the review tasks pass on the first try.

## Integration Tests as Deterministic Proof

Unit tests mock dependencies. They prove the code does what the code does, not that the system works. The LLM reviewers check code quality and test coverage, but they're reading code, not running it. Neither can prove the deployed system actually satisfies the requirements.

Integration tests are different. When the Integration Tester invokes the real deployed Lambda and asserts the expected behavior, that's deterministic proof that the requirement is met. The test either passes or fails. No LLM judgment involved.

The Integration Tester reads `requirements.md`, not the implementation code. It writes tests from the requirements alone, so it can't be biased by knowing how the code works. This creates a verification chain:

| Step | Who | Type |
|---|---|---|
| 1 | Test-Writer writes tests from requirements | Defines the contract |
| 2 | Worker writes code to satisfy tests | Implements the contract |
| 3 | Req-Reviewer verifies tests match requirements | Probabilistic |
| 4 | Quality-Reviewer verifies code follows rules | Probabilistic |
| 5 | Build compiles and runs unit tests | **Deterministic** |
| 6 | Integration tests verify the deployed system | **Deterministic** |
| 7 | Final Validator cross-references everything | Probabilistic |

Steps 5 and 6 are the real gates. The LLM steps add value, catching things the deterministic gates miss, but they are not the primary defense.

## Model Selection

Not every agent needs the same model. The Final Validator gets Opus: it runs once, it needs deep cross-referencing across multiple documents, and the quality of that single run matters most. The Worker, Test-Writer, Req-Reviewer, and Integration Tester run on Sonnet, where reasoning through implementation and comprehension is the main requirement. The Quality Reviewer runs on Haiku. It's mechanical pattern matching against rules, and it runs on every task, then again on every bounce-back. The cost compounds fast. Haiku is fast, cheap, and accurate enough for rule-based auditing.

## The Loop Script

A bash script orchestrates the agents. It reads the task list, determines the next unchecked task, dispatches to the correct agent based on task type (TEST-FIRST, implementation, REQ-CHECK, PRE-CR, INTEG, FINAL), and repeats until all tasks are complete or max iterations are reached.

The core structure is exactly Geoffrey Huntley's original Ralph loop:

```bash
while :; do
  NEXT_TASK=$(get_next_unchecked_task tasks.md)
  AGENT=$(get_agent_for_task "$NEXT_TASK")
  cat "${AGENT}_prompt.md" spec.md tasks.md progress.txt | claude-code
done
```

Each agent's system prompt is concatenated with the spec context and piped to Claude Code. Each iteration is a fresh context, no memory leaks between agents. The agents communicate through files: the task list for task state, and a progress log for violation reports, decisions, and learnings.

## Parallel Workstreams

For larger projects with independent components, the loop extends naturally. Each workstream gets its own git worktree with an isolated branch and build artifacts. Without this, parallel Workers would step on each other's compiled output.

Multiple Ralph loops run simultaneously, and CRs are raised independently per workstream. When all workstreams complete, they're merged and the Integration Tester and Final Validator run against the combined system.

## What I Learned

Formal verification finds bugs no test will catch. Using TLA+/Quint to model-check the state machine before writing any code found a bug, a missing context variable on Step Function resume, that would have only surfaced in production. The spec is cheap to write and the model checker is exhaustive: it checks every possible state, not just the ones you think to test. I now treat formal verification as a prerequisite, not an afterthought.

The human's job shifts from writing code to writing specs, and that changes how you use your time. The requirements document, the design document, and the task structure become the most important artifacts. The better the spec, the better the AI-generated code. The worse the spec, the more bounce-backs and wasted iterations. If the requirements are ambiguous, the agents argue about them through violation reports and corrections, which is expensive. Clarity up front is cheap.

In practice, this has changed my time allocation dramatically. Take a typical sprint task that requires a low-level design: a component that would take roughly five days to complete. Previously I'd spend two days on design and three on implementation, writing code, debugging, iterating. Now I spend three to four days on design and one to two on implementation. The implementation runs largely unattended.

The same shift applies at a larger scale. When designing a distributed system, something that might take thirty days end to end, the high-level design itself benefits from the same discipline. I spend more time on architecture, formal verification, and getting the component specs right before any agent writes a line of code. That shift is what makes TLA+ feasible. It's not a luxury you can afford when you're also writing all the code, but it becomes the obvious move when you're not.

The payoff is fewer production bugs. A formally verified design with a well-structured spec produces fewer surprises when it ships. The rework that used to happen after the fact gets moved to the design phase, where it's far cheaper to fix.

No matter how much you delegate to AI, you can't skip human review entirely. After the Final Validator passes and before raising a code review, I do a manual self-review of the output. This is a deterministic gate, not probabilistic like the LLM reviewers, because it's a human decision: approve or don't. That gate catches the things all the agents missed and gives you confidence before the code goes to other people.

The system works because the structure does the work, not the prompts. Each agent is doing a narrow job with clear incentives. The human's job is to write a spec good enough that the agents have no room to go wrong.

---

*Credit to [Geoffrey Huntley](https://ghuntley.com/ralph/) for the Ralph loop technique that this system builds on. Further reading: Marc Brooker on [spec-driven development vs waterfall](https://brooker.co.za/blog/2026/04/09/waterfall-vs-spec.html) and [specifications as the future of AI-assisted development](https://kiro.dev/blog/kiro-and-the-future-of-software-development/). For Java services, I've also added a mutation testing gate using PIT — covered in [Wiring Mutation Testing into an Autonomous Coding Loop](/2026/04/19/pit-mutation-testing-ralph-loop/).*
