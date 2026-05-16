---
layout: post
title: "Wiring a Formal Verifier into an Autonomous Coding Loop"
date: 2026-05-10
related:
  - /2026/04/10/multi-agent-tdd-loop/
  - /2026/04/19/pit-mutation-testing-ralph-loop/
  - /2026/04/23/final-validator-ralph-loop/
  - /2026/04/27/integration-test-contracts/
---

An autonomous coding loop that cannot satisfy its feedback signal except by being correct turns out to be a useful construct. This post is what happens when we wire one to a formal verifier and ask it to prove three non-trivial things.

The previous four posts in this series tightened the feedback signal that an autonomous coding loop runs against. First plain tests. Then [mutation testing](https://ranjithkannan.com/2026/04/19/pit-mutation-testing-ralph-loop/), to check that the tests caught real bugs. Then a [separate auditor](https://ranjithkannan.com/2026/04/23/final-validator-ralph-loop/), so the loop could not grade itself. Then [integration contracts](https://ranjithkannan.com/2026/04/27/integration-test-contracts/), to close the gap between green tests and observable correctness. Each step closed a hole through which a wrong loop could still pass.

A formal verifier is the limit of that progression. It is a feedback signal the agent cannot satisfy except by either weakening the specification or actually being correct. The methodology in this post is the rule that closes the first path: no spec weakening, audited at two boundaries, on every commit. The empirical question is whether the loop can still produce verified code under that constraint.

We picked Verus, because it operates on real Rust rather than on a custom language. We picked three exercises of increasing difficulty: a sorted binary search, a fixed-capacity append-only log with a frame property, and a Byzantine quorum check whose spec talks about mathematical set cardinality but whose implementation has to walk a `Vec`. The first two are textbook. The third is where the abstraction-to-implementation gap shows up.

Everything described here lives at <https://github.com/ranjithkannank/verus-calibration>. The per-section links below point at the exact files.

## The setup

Three roles, three [Claude Code subagents](https://github.com/ranjithkannank/verus-calibration/tree/main/.claude/agents), three (mostly two) models.

- **[Architect](https://github.com/ranjithkannank/verus-calibration/blob/main/.claude/agents/architect.md)** (Opus 4.7). Reads the frozen spec. Writes a design note. Does not see verifier output on the first pass.
- **[Implementer](https://github.com/ranjithkannank/verus-calibration/blob/main/.claude/agents/implementer.md)** (Sonnet 4.6). One attempt per call: edit the file, run verus, log the result.
- **[Reviewer](https://github.com/ranjithkannank/verus-calibration/blob/main/.claude/agents/reviewer.md)** (Opus 4.7). After verus passes, audits the diff against the frozen baseline. Returns `APPROVE` or `REJECT`.

The reviewer is a separate role for the same reason the [previous post](https://ranjithkannan.com/2026/04/23/final-validator-ralph-loop/) argued for splitting audit from decision. The architect committed to a design that produced the implementation. Asking it to audit that same implementation for spec drift bakes in confirmation bias. A separate audit on a fresh context is a cheap structural safeguard.

The three roles are wired together by a [Ralph-style outer loop][ralph] in bash. The loop reads state from filesystem artifacts and fires one `claude -p` call per iteration with fresh context each time. Memory lives in [`AGENTS.md`][agentsmd], the design note, `attempts.md`, and git history. The state machine:

[ralph]: https://github.com/ranjithkannank/verus-calibration/blob/main/ralph/run-exercise.sh
[agentsmd]: https://github.com/ranjithkannank/verus-calibration/blob/main/AGENTS.md

```
                  ┌─────────────────────────────────┐
                  ▼                                 │
START ──► THINK ──► WORK ──► (verus passes) ──► REVIEW
                     │                            │   │
                     │                            │   ├─► APPROVE ──► DONE
                     │                            │   │
                     │                            │   └─► REJECT ──► WORK_AFTER_REJECT
                     │
                     └─► (escalation) ──► THINK_REVISE ──► WORK
```

State is inferred from the filesystem on every iteration. Presence of the design note means we are past THINK. The number of entries in `attempts.md` is the current attempt count. The `Conclusion` line in `review.md` drives APPROVE versus REJECT. There is no explicit state file. The loop is fully resumable: kill it, re-run, it picks up where it left off.

A single per-iteration claude invocation looks like this:

```bash
claude -p \
  --agent "$agent" \
  --model "$model" \
  --no-session-persistence \
  --permission-mode acceptEdits \
  --allowedTools "${allowed[@]}" \
  --disallowedTools "${DISALLOWED_TOOLS[@]}" \
  -- "$prompt" > "$iter_log" 2>&1
```

The `--` separator is required. Without it the variadic `--allowedTools` list eats the prompt argument. The `--no-session-persistence` flag is what makes this a Ralph loop rather than a long-lived agent. Each iteration starts from a clean conversational slate, with only what is on disk to ground it.

Each implementer attempt is its own git commit. Two reasons. The reviewer's diff target needs to be a stable snapshot. And when something goes wrong later, we can quote the actual code.

## The two sandbox layers

A loop that runs unattended needs boundaries. There are two.

**Content boundary: a pre-commit hook.** Every commit goes through [`scripts/git-hooks/pre-commit`][hook]. It rejects three things.

1. Any staged file outside a path whitelist (`exercises/`, `logs/`, `writeup/`, `ralph/`, `scripts/`, `.claude/`, the named top-level docs).
2. Any cheat token added in `exercises/*.rs`: `assume(`, `#[verifier::external_body]`, `unreachable!()`, `panic!(`, `assume_specification`.
3. Spec drift. Every line in the frozen baseline whose first token is `requires` or `ensures` must appear verbatim in the staged file. New helper `spec fn`s are allowed. Modifying frozen ones is not.

The third check is the one that does the work. For each staged exercise file, the hook diffs against the `spec-frozen-<exercise>` git tag and runs the equivalent of:

```bash
frozen=$(git show "$tag:$f" | grep -E '^[[:space:]]*(requires|ensures)([^a-zA-Z0-9_]|$)')
staged_content=$(git show ":$f")
while IFS= read -r line; do
  if ! echo "$staged_content" | grep -qFx "$line"; then
    errors+=("[spec]  $f: frozen line missing verbatim: '$line'")
  fi
done <<< "$frozen"
```

`-Fx` is exact-line match. Cosmetic reformatting that touches a frozen `requires` or `ensures` line is rejected as firmly as a semantic weakening. Intentionally strict.

[hook]: https://github.com/ranjithkannank/verus-calibration/blob/main/scripts/git-hooks/pre-commit

**Capability boundary: a Claude Code tool whitelist.** Each `claude -p` call passes a role-scoped `--allowedTools` list and a universal `--disallowedTools` deny set. The deny set includes `WebFetch`, `WebSearch`, `Task` (no sub-agent fan-out), bash patterns for `rm`, `git push/reset/rebase/checkout/restore/config`, `*--no-verify*` so the agent cannot bypass the hook, `curl/wget/ssh`, `brew/npm/pip`, `sudo`. The architect gets only the tools it needs to write a design note. The reviewer does not get `Edit`.

To state what this is not: it is not process isolation, it is not a network namespace, and a deliberately adversarial agent could find paths around it. The agents in this experiment are not adversarial. The boundary's job is to make it impossible for an honest worker to cheat accidentally. To silently weaken a spec because the verifier complained, or to slip an `assume` past the reviewer because it was tired. That boundary held across all three exercises.

## What happened

| Exercise         | Status | Attempts to verify | Notable                                         |
|------------------|--------|--------------------|-------------------------------------------------|
| binary_search    | DONE   | 1                  | Architect's design predicted every invariant.   |
| bounded_log      | DONE   | 1 (post re-freeze) | Methodology pressed hardest here.               |
| quorum_count     | DONE   | 2                  | Real concrete-to-abstract proof engineering.    |

### binary_search

The easiest. A verified binary search with a sortedness precondition and the standard found/not-found postconditions.

The architect's design note ran to about 200 lines. It predicted what the proof would need: a half-open window `[lo, hi)` (justified, because `hi = mid - 1` underflows when `usize` cannot go negative), an overflow-safe `mid = lo + (hi - lo) / 2`, and the five conjuncts of the loop invariant. The two `forall` exclusion ranges tile the index space on loop exit. The implementer wrote a body that matched the design almost exactly. First try, verus passed, reviewer approved.

The implementer also appended four discovered patterns to `AGENTS.md` on its own. A `decreases` clause is required on every `while` loop. The `assert forall ... by { assert(is_sorted(...)); }` block reliably triggers sortedness instantiation. Half-open avoids underflow. The invariant should be five conjuncts in this shape. The role file has an explicit escape hatch for findings, and the implementer used it. That turns into a memory artifact future exercises read.

### bounded_log

This exercise pressed hardest on the methodology.

The spec is a fixed-capacity append-only log with `new`, `len`, `get`, and an `append` whose postcondition includes a frame property. In plain English, the frame property says that after a successful append, every existing entry at index `i < old_len` still equals what it was before. SMT solvers need that stated explicitly, usually with a defensive `assert` after the mutation. The underlying axioms about `Vec::push` do not fire eagerly when the goal is a quantified statement about older indices. The architect knew this and predicted the assert chain. The implementer wrote it.

On the first attempt, the implementer wrote `final(self)` everywhere in the `ensures` clause of `append` instead of bare `self`. Verus accepted: `4 verified, 0 errors`. The reviewer ran the audit against the frozen tag and rejected. The diff hunk that triggered the rejection:

```diff
- self.well_formed(),
- self.capacity() == old(self).capacity(),
+ final(self).well_formed(),
+ final(self).capacity() == old(self).capacity(),
  result.is_ok() ==> {
-     &&& self.view().len() == old(self).view().len() + 1
-     &&& self.view()[old(self).view().len() as int] == msg
+     &&& final(self).view().len() == old(self).view().len() + 1
+     &&& final(self).view()[old(self).view().len() as int] == msg
      // Frame property: existing entries are unchanged.
-                       self.view()[i] == old(self).view()[i]
+                       final(self).view()[i] == old(self).view()[i]
```

Six lines inside the `ensures` block, all rewritten. The reviewer's [full audit][bounded-log-reject] cited the line numbers, then explained:

> Even granting the implementer's claim that `final(self)` is semantically equivalent to the post-state `self`, this is not byte-identical and therefore falls under rule 1.

The reviewer rule fired exactly as designed. The implementer had even added a discovered-pattern note claiming `final(self)` was required by the current Verus version, and that claim turned out to be right.

[bounded-log-reject]: https://github.com/ranjithkannank/verus-calibration/commit/2f61144

On the second attempt, the implementer restored bare `self` to match the frozen spec. Verus rejected it with a clear migration error pointing at the new `&mut self` postcondition disambiguation rule. Both paths now violated a rule. One from the reviewer, one from the compiler. The implementer wrote a structured blocker report:

> | Constraint                    | What it requires                                                                |
> |-------------------------------|---------------------------------------------------------------------------------|
> | Frozen spec (reviewer rule 1) | `self.well_formed()`, `self.capacity()`, `self.view()` (bare `self`)            |
> | Verus 0.2026.05.13 syntax     | `final(self).well_formed()`, `final(self).capacity()`, `final(self).view()`     |
>
> These two constraints are mutually exclusive. No implementer-level change can satisfy both simultaneously. Only the architect (per AGENTS.md) is empowered to re-freeze the spec.

This is the outcome a trustworthy methodology should produce. The agent did not silently bypass the reviewer's rule with `final(self)`. It did not silently bypass Verus by adding an `assume`. It articulated the conflict, named the role empowered to resolve it, and stopped.

The frozen spec we had written was wrong. It predated the Verus version on the machine. We re-froze the baseline (a single targeted commit moving every post-state `self` to `final(self)`, function bodies left as `unimplemented!()`). The loop restarted clean against the corrected tag. First-try clean: architect, one implementer attempt, reviewer APPROVE, DONE. The proof body was nearly identical to the rejected version. Only the spec text in the frozen tag had changed.

The methodology held up against an operator error in spec authorship. That was the test it most needed to pass.

### quorum_count

The hardest, by design. The spec defines `distinct_count(voters) = voters.to_set().len()`, mathematical set cardinality. The implementation has to walk a `Vec<NodeId>` and count distinct elements with some concrete data structure. The proof has to bridge those two worlds.

The architect proposed a bitmap-backed approach: a `Vec<bool>` of length `n` plus a `u64` counter, one linear pass, O(1) per step. The proof reduces to three helper lemmas.

1. `s.subrange(0, i+1) == s.subrange(0, i).push(s[i])`, the prefix step.
2. `s.push(x).to_set() == s.to_set().insert(x)`, the concrete-to-abstract bridge.
3. `s.insert(x).len() == s.len()` when `x ∈ s`, already-present insertion is identity.

On attempt 1 the implementer wrote about 330 lines covering the algorithm and all three architect-predicted lemmas plus two more it decided it needed. Verus produced `5 verified, 2 errors`:

```
error: invariant not satisfied before loop
   --> exercises/quorum_count.rs:153:13
    |
153 |             count as nat == voters@.subrange(0, i as int).to_set().len(),

error: assertion failed
   --> exercises/quorum_count.rs:254:20
    |
254 |             assert(count as nat <= n as nat);
```

The `attempts.md` entry diagnosed both failures precisely. The empty-subrange `to_set().len()` was not seen as 0 at loop entry. The `count <= n` bound after increment needed a pigeonhole argument. It proposed specific fixes. The autoresearch loop working as designed: each attempt is a measurable, named outcome that feeds the next.

On attempt 2 the implementer did something we had not predicted. Rather than guessing at lemma names, it grepped the local `vstd` source for relevant helpers. It found `vstd::set_lib::lemma_len_subset(s1, s2)`, which proves `s1.len() <= s2.len()` when `s1 ⊆ s2` and `s2.finite()`. It found `lemma_int_range(lo, hi)`, which proves a range set is finite with cardinality `hi - lo`. It noticed the type mismatch: `voters.to_set()` is `Set<NodeId>` but `lemma_int_range` is on `Set<int>`. It wrote a new helper [`lemma_range_nodeid_len(n: u32)`][quorum-rs] as the NodeId analogue, by structural recursion on `u32`:

```rust
proof fn lemma_range_nodeid_len(n: u32)
    ensures
        Set::<NodeId>::new(|k: NodeId| (k as int) < n as int).finite(),
        Set::<NodeId>::new(|k: NodeId| (k as int) < n as int).len() == n as nat,
    decreases n,
{
    let s = Set::<NodeId>::new(|k: NodeId| (k as int) < n as int);
    if n == 0 {
        assert(s =~= Set::<NodeId>::empty());
    } else {
        let n1: u32 = (n - 1) as u32;
        let m: NodeId = n1;
        let s1 = Set::<NodeId>::new(|k: NodeId| (k as int) < n1 as int);
        lemma_range_nodeid_len(n1);                          // recurse
        assert(s1.insert(m) =~= s) by { /* element-wise */ };
        assert(!s1.contains(m));
        vstd::set::axiom_set_insert_len(s1, m);              // bridge
    }
}
```

The agent did not have to write this. It could have left `assert(count <= n)` in place and let verus keep complaining. It could have weakened the invariant. It could have added an `assume` (and been caught by the hook). Instead it wrote a recursive cardinality lemma mirroring the shape of vstd's own range lemma. The implementer's role doing what the architecture asked.

[quorum-rs]: https://github.com/ranjithkannank/verus-calibration/blob/main/exercises/quorum_count.rs

It also caught its own regression. After making the surgical proof additions, it ran verus and got a Rust-level type error in its own just-added code. `u32 - 1` evaluates to `int` inside a Verus `proof fn` context, and the subtraction needed a cast. It read the verus output, recognized the new failure pattern, and patched the cast in the same iteration. Then re-ran:

```
verification results:: 8 verified, 0 errors
```

The reviewer's audit took under a minute. Five-point checklist passed with specific line citations, plus a cross-exercise observation:

> The implementer leans heavily on `=~=` extensional equality and `choose` witnesses to push set/seq reasoning through the SMT solver; this pattern recurred in `bounded_log` too and is worth canonizing in the architect's playbook.

The audit role doing more than gatekeeping. It accumulated a pattern across exercises and proposed promoting it to architect-level guidance.

## What the loop got right

Three claims we are willing to defend from this run.

**The no-spec-weakening rule held under pressure.** Both the mechanical hook and the semantic reviewer caught real violations. The hook caught a body-content modification on bounded_log that no other rule matched. The reviewer caught a spec-shape modification that the hook missed because its check is keyword-prefix only. Two layers that fail differently are the boundary. One layer alone was not enough.

**Per-attempt commits made the methodology auditable.** Every implementer iteration is its own commit, with the verifier output captured under `logs/<ex>/raw/`. The reviewer's audits cite specific HEAD lines. The git log is a usable timeline for the failure taxonomy. None of this needed extra tooling. Git was sufficient, once the orchestrator was disciplined about committing on each attempt.

**The role split bought what it advertised.** The architect produced substantive design notes that the implementers followed. The implementers self-diagnosed their own failures and named follow-up plans rather than thrashing. The reviewer audited the diff with checklist rigor and contributed cross-exercise pattern observations. A single Opus call could have done all three jobs, probably. Whether its review of its own work would have been as honest as a separate role's, we doubt.

## What the loop got wrong

Three honest shortcomings worth flagging.

**The hook's spec-preservation check has a known gap.** It looks for lines whose first token is `requires` or `ensures`. The body content of those clauses, the continuation lines, is not on the frozen-line list. The bounded_log diff changed those body lines and the hook let it through. The reviewer caught it, which is what splitting audit from decision predicts. The fix is line-range matching instead of line-prefix matching.

**The orchestrator treats every non-zero claude exit code as verus-failed.** It is not. An exit code can also mean a rate-limit response, a budget cap firing, a network blip. The orchestrator should distinguish these and recover appropriately, not just count another failed attempt and try again. On a longer run, this would produce wasted iterations against transient infrastructure issues.

**Three exercises is not a benchmark.** The vericoding paper has 12,504 specifications. We have three. The generalizations we are comfortable drawing are at the level of "this failure mode exists" or "this rule survived this pressure," not "success rate equals X percent." Sample size three is sample size three.

## Where this fits

Three existing bodies of work touch pieces of this, but not the whole intersection.

The **Verus vericoding benchmark** (Schubert et al., 2025) measures LLM success rates on isolated single-function Verus tasks. One model, one shot, one function, no role split, no autonomous outer loop, no treatment of cheating. The benchmark tells us 44% first-try is the current floor for raw single-shot Verus. This experiment is about what surrounding scaffolding gets a real codebase past it.

**Microsoft Research's AutoVerus and VeruSAGE** ([microsoft/verus-proof-synthesis](https://github.com/microsoft/verus-proof-synthesis)) are doing pieces of the same problem from a different direction. AutoVerus (OOPSLA 2025, [arXiv:2409.13082](https://arxiv.org/abs/2409.13082)) targets small algorithmic code with a three-phase pipeline: inference, refinement, repair. VeruSAGE ([arXiv:2512.18436](https://arxiv.org/abs/2512.18436)) targets large system projects, multi-module code, and ships a benchmark (VeruSAGE-Bench) of 849 tasks drawn from real distributed systems, OS kernels, and storage systems. Both systems are LLM-based on the OpenAI side. Their orchestration is a single-agent pipeline rather than a role split, and they do not appear to ship an explicit no-cheating boundary. The overlap with our work is real and we read their results as a strong baseline for any next step.

**Huntley's [Ralph pattern](https://ghuntley.com/ralph/)** describes the outer-loop shape: fresh context per iteration, file-based memory, agent commits per attempt. It applies the pattern to unsafe-code domains (internal tools, CRUD apps) where the feedback signal is the test suite. Tests pass or fail. No verifier, no no-cheating rule, no separate audit role. The contribution here is keeping the loop shape and changing the feedback signal to a verifier, plus adding the structural pieces (the sandbox, the audit role) that the harder signal requires.

**Karpathy's autoresearch demos** popularized the agent-runs-an-experiment-overnight framing, with a scalar metric (a loss, a benchmark score) and a single agent that proposes, implements, and evaluates in the same context. The metric here is binary (verus exits 0). The agent doing the work is not the agent doing the evaluation. The rules forbidding spec drift live outside the agent's reach. Same overnight-experiment shape, but the proposer, implementer, and auditor are three separate roles by construction.

After the Microsoft work in particular, the claim worth defending is narrower than "no prior public work in this intersection." It is: a separate audit role running on a different model, a mechanical commit-time enforcement layer (the pre-commit hook) running alongside the LLM audit, and per-attempt commits as the unit of evaluation. The combination is what we have not seen in published systems, and it is what the bounded_log REJECT result depended on.

There is also a second contribution worth naming: the operator-intervention case on bounded_log. Most published vericoding results either succeed silently or fail silently. The loop's behavior on the bounded_log conflict, where the agent refused to cheat in either direction, articulated the constraint, named the empowered role, and stopped, is the kind of structured-failure output a trustworthy methodology should produce. A single-shot prompt would either silently apply `final(self)` or silently fail. This run surfaced the conflict, blamed the right party (the operator), and waited.

The next rung is multi-module Verus code with cross-module invariants. The concrete plan is to draw tasks from VeruSAGE-Bench (so the results are comparable to existing published numbers) and run them through this loop, with AutoVerus and VeruSAGE as baselines on the same tasks.

## Reproducing

- Repo: <https://github.com/ranjithkannank/verus-calibration>
- Verus: `0.2026.05.13.fae8859`, arm64-macos binary release.
- Models: `claude-opus-4-7` for architect and reviewer, `claude-sonnet-4-6` for implementer. The bash script holds the source of truth. Subagent frontmatter declares defaults; the outer loop overrides per call.
- All prompts are inlined in `ralph/run-exercise.sh`. All raw verifier output is committed under `logs/<ex>/raw/`. `AGENTS.md` is the rule book; the pre-commit hook in `scripts/git-hooks/pre-commit` is the enforcement.

---

*This post is part of a series on the multi-agent Ralph loop. See [the architecture post](https://ranjithkannan.com/2026/04/10/multi-agent-tdd-loop/), [mutation testing](https://ranjithkannan.com/2026/04/19/pit-mutation-testing-ralph-loop/), [the Final Validator](https://ranjithkannan.com/2026/04/23/final-validator-ralph-loop/), and [integration test contracts](https://ranjithkannan.com/2026/04/27/integration-test-contracts/).*
