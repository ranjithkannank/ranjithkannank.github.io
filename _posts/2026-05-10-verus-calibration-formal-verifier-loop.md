---
layout: post
title: "Wiring a Formal Verifier into an Autonomous Coding Loop"
date: 2026-05-10
related:
  - /2026/04/10/multi-agent-tdd-loop/
  - /2026/04/19/pit-mutation-testing-ralph-loop/
  - /2026/04/23/final-validator-ralph-loop/
  - /2026/04/27/integration-test-contracts/
  - /2026/05/16/verified-byzantine-quorum-certificate/
  - /2026/05/17/verified-byzantine-tolerant-sensor-fusion/
---

An autonomous coding loop that cannot satisfy its feedback signal except by being correct turns out to be a useful construct. This post is what happens when we wire one to a formal verifier and ask it to prove twelve non-trivial things.

The previous four posts in this series tightened the feedback signal that an autonomous coding loop runs against. First plain tests. Then [mutation testing](https://ranjithkannan.com/2026/04/19/pit-mutation-testing-ralph-loop/), to check that the tests caught real bugs. Then a [separate auditor](https://ranjithkannan.com/2026/04/23/final-validator-ralph-loop/), so the loop could not grade itself. Then [integration contracts](https://ranjithkannan.com/2026/04/27/integration-test-contracts/), to close the gap between green tests and observable correctness. Each step closed a hole through which a wrong loop could still pass.

A formal verifier is the limit of that progression. It is a feedback signal the agent cannot satisfy except by either weakening the specification or actually being correct. The methodology in this post is the rule that closes the first path: no spec weakening, audited at two boundaries, on every commit. The empirical question is whether the loop can still produce verified code under that constraint.

We picked Verus, because it operates on real Rust rather than on a custom language. We picked twelve exercises of increasing difficulty. The first six are a calibration set of standalone algorithmic exercises: a sorted binary search, a fixed-capacity append-only log with a frame property, a Byzantine quorum check whose spec talks about mathematical set cardinality but whose implementation has to walk a `Vec`, a single-module quorum certificate with a safety lemma, and two Byzantine-tolerant sensor-fusion algorithms (fault-tolerant midpoint and Marzullo's interval algorithm). The next three stress the harness on multi-module Verus. The last three compose the BFT primitives at increasing levels of trust-boundary integration, with the final exercise doubling as a deliberate test of whether the methodology supports proof discovery and not only proof execution.

Everything described here lives at <https://github.com/ranjithkannank/verus-calibration>. The per-section links below point at the exact files.

## The setup

Three roles, three [Claude Code subagents](https://github.com/ranjithkannank/verus-calibration/tree/main/.claude/agents), one model now (originally two).

- **[Architect](https://github.com/ranjithkannank/verus-calibration/blob/main/.claude/agents/architect.md)** (Opus 4.7). Reads the frozen spec. Writes a design note. Does not see verifier output on the first pass.
- **[Implementer](https://github.com/ranjithkannank/verus-calibration/blob/main/.claude/agents/implementer.md)** (Opus 4.7, originally Sonnet 4.6). One attempt per call: edit the file, run verus, log the result.
- **[Reviewer](https://github.com/ranjithkannank/verus-calibration/blob/main/.claude/agents/reviewer.md)** (Opus 4.7). After verus passes, audits the diff against the frozen baseline. Returns `APPROVE` or `REJECT`.

The implementer ran on Sonnet 4.6 for the original three exercises and handled them cleanly. For BFT-path exercises starting with quorum_cert the implementer was switched to Opus 4.7. Those proof obligations involve genuine cardinality and pigeonhole reasoning where the model needs to plan over many tokens of internal thinking before producing structured output. Aligning the hardest role to the strongest model, even when that role is the most expensive.

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

Each implementer attempt is also scoped narrowly. The prompt directs the implementer to either pick the next unfinished sub-task from the architect's design or scope its edits to the specific failing function from the latest verifier output. Small, surgical edits per iteration, not file-wide rewrites. The orchestrator iterates; the implementer does not need to fix everything in one call.

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

To state what this is not: it is not process isolation, it is not a network namespace, and a deliberately adversarial agent could find paths around it. The agents in this experiment are not adversarial. The boundary's job is to make it impossible for an honest worker to cheat accidentally. To silently weaken a spec because the verifier complained, or to slip an `assume` past the reviewer because it was tired. That boundary held across all twelve exercises.

## What happened

| Exercise         | Status | Attempts to verify | Notable                                         |
|------------------|--------|--------------------|-------------------------------------------------|
| binary_search    | DONE   | 1                  | Architect's design predicted every invariant.   |
| bounded_log      | DONE   | 1 (post re-freeze) | Methodology pressed hardest here.               |
| quorum_count     | DONE   | 2                  | Real concrete-to-abstract proof engineering.    |
| quorum_cert      | DONE   | 6                  | First BFT-shaped exercise. Six narrow iterations through the architect's sub-task list. Pigeonhole-via-contradiction proof of a quorum certificate's honest-voter guarantee. |
| ft_midpoint      | DONE   | 7                  | First sensor-fusion artifact. Brute-force scan with inclusion-exclusion proof. |
| marzullo         | DONE   | 1 (post re-freeze) | Interval generalization. Operator-intervention case: the loop refused to verify and surfaced a missing Helly-1D precondition. |
| cross_module_counter | DONE | 1               | First multi-module exercise. Nested `mod` blocks inside one `verus!{}`. Closed-spec vocabulary across the module boundary. |
| counter_multifile    | DONE | 1               | First multi-file exercise. Same algorithm, sibling-file layout. Tooling test for path whitelist and exercise-name derivation. |
| counter_producer     | DONE | 1               | First cross-module composition. Producer's loop invariant carries facts the counter doesn't expose. |
| sensor_poll          | DONE | 1               | First composition of BFT primitives. One-line projection lemma bridges marzullo's frame to the caller's frame. |
| sensor_poll_signed   | DONE | 1               | Adds the cryptographic trust boundary from `quorum_cert` at the spec layer. |
| sensor_poll_honest   | DONE | 1               | Deliberate discovery test. Design note omits lemma names; agent reuses inclusion-exclusion from the `ft_midpoint` playbook entry. |

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

### quorum_cert

The first BFT-shaped exercise, after the calibration. The spec asks for two obligations: a structural runtime check that a quorum certificate has the right number of distinct in-range voters, and a safety lemma that any valid certificate over `n` nodes with fewer than `n/3` Byzantine voters contains at least one honest voter. The cryptographic predicates are uninterpreted, declared with `pub uninterp spec fn` and no body. The verified module proves the BFT-layer reasoning; a real deployment connects signature checks via a wrapper outside.

The architect proposed a bitmap-backed structural check (the same pattern that worked for `quorum_count`) and a pigeonhole-via-contradiction proof of the safety lemma. The design note ended in a nine-item sub-task list, ordered easiest to hardest. The implementer worked through it in six narrow iterations: skeleton, cursor bounds, bitmap abstraction invariant, pairwise distinctness, helpers lifted from `quorum_count`, then the safety lemma. Each attempt scoped to one sub-task. The final attempt landed `12 verified, 0 errors` with the pigeonhole step expressed as `if !(exists honest h) { assert(false) }`: the negated existential turns into a universal, which implies a subset relation between the certificate's voters and the Byzantine set, which closes against the threshold by cardinality.

The reviewer's APPROVE noted the same pattern recurring across exercises and flagged it for the architect's playbook. Cross-exercise memory in `AGENTS.md` already contained the universe-size lemma pattern from `quorum_count`; the architect for `quorum_cert` lifted it directly. Each exercise's findings make the next one cheaper. The full per-attempt history and reviewer audit live in [`logs/quorum_cert/`](https://github.com/ranjithkannank/verus-calibration/tree/main/logs/quorum_cert), and the verified module sits at [`exercises/quorum_cert.rs`](https://github.com/ranjithkannank/verus-calibration/blob/main/exercises/quorum_cert.rs). A [dedicated follow-up post](https://ranjithkannan.com/2026/05/16/verified-byzantine-quorum-certificate/) covers the proof structure, the pigeonhole-by-contradiction pattern, and the BFT-path roadmap in more depth.

### ft_midpoint

The first sensor-fusion artifact. Input: a vector of `i64` sensor readings and a Byzantine bound `f`. Output: a single `i64` guaranteed to lie between the lowest and highest readings that honest sensors would have produced. The implementer's algorithm was a brute-force scan: for each candidate reading, count how many readings are `≤` it and how many are `≥` it; the first candidate with both counts at least `f + 1` is the answer.

The proof reduces to inclusion-exclusion over set cardinalities. Two prefix-set helpers (`count_le`, `count_ge`) with five-conjunct invariants, two pigeonhole lemmas (`lemma_pigeonhole_le`, `lemma_pigeonhole_ge`) that lift the `≥ f+1` counts into existence of a correct sensor on each side, and a closing existence lemma (`lemma_exists_midpoint`) that handles the post-loop case. The architect lifted the universe-size and pigeonhole patterns from `quorum_count` and `quorum_cert`. Seven attempts to converge, each scoped to one sub-task from the design. The reviewer's audit flagged the inclusion-exclusion forward construction as a methodological alternative to pigeonhole-by-contradiction, worth knowing across exercises.

The full proof and per-attempt history are at [`exercises/ft_midpoint.rs`](https://github.com/ranjithkannank/verus-calibration/blob/main/exercises/ft_midpoint.rs) and in [`logs/ft_midpoint/`](https://github.com/ranjithkannank/verus-calibration/tree/main/logs/ft_midpoint). A [dedicated follow-up post](https://ranjithkannan.com/2026/05/17/verified-byzantine-tolerant-sensor-fusion/) covers the sensor-fusion track in depth.

### marzullo

Marzullo's algorithm, the interval generalization of ft_midpoint. Sensors report ranges instead of single values; the algorithm returns the smallest output interval whose interior contains a point that at least `n - f` input intervals also contain.

The first run failed in a methodology-grade way. The implementer worked through five iterations and got 8 of 9 verifier obligations to verify. The failing assertion required `intervals[i].lo ≤ intervals[j].hi` for two correct sensor indices, the Helly-1D overlap condition the algorithm depends on. Nothing in the specification said anything about correct sensors' intervals overlapping. As written, two "correct" sensors were allowed to report `[0, 0]` and `[10, 10]`, disjoint singletons that satisfied every precondition we'd actually written.

The implementer produced a constructive counterexample, wrote a structured blocker report, and stopped. It did not weaken the spec, did not slip in an `assume`, did not try further algorithmic variants. The architect, re-invoked under the escalation path, confirmed the diagnosis three separate times in three independent revisions of the design. The operator (us) had forgotten to write down the Helly-1D precondition. After adding it and re-freezing the spec, the loop converged in a single attempt by lifting scaffolding from the prior blocked run.

This was the second methodology-grade signal in the same shape as bounded_log: the loop's refusal to cheat surfaced an operator-authored specification bug as a clean, structured report. Two out of six exercises required operator intervention to fix specification bugs; that is a meaningful rate, and exactly the rate you'd want from a methodology whose purpose is to surface the specification-authoring work the operator skipped. See [the sensor-fusion post](https://ranjithkannan.com/2026/05/17/verified-byzantine-tolerant-sensor-fusion/) for the full case, including the constructive counterexample preserved in git history.

## What the loop got right

Seven claims we are willing to defend from this run.

**The no-spec-weakening rule held under pressure.** Both the mechanical hook and the semantic reviewer caught real violations. The hook caught a body-content modification on bounded_log that no other rule matched. The reviewer caught a spec-shape modification that the hook missed because its check is keyword-prefix only. Two layers that fail differently are the boundary. One layer alone was not enough.

**Per-attempt commits made the methodology auditable.** Every implementer iteration is its own commit, with the verifier output captured under `logs/<ex>/raw/`. The reviewer's audits cite specific HEAD lines. The git log is a usable timeline for the failure taxonomy. None of this needed extra tooling. Git was sufficient, once the orchestrator was disciplined about committing on each attempt.

**The role split bought what it advertised.** The architect produced substantive design notes that the implementers followed. The implementers self-diagnosed their own failures and named follow-up plans rather than thrashing. The reviewer audited the diff with checklist rigor and contributed cross-exercise pattern observations. A single Opus call could have done all three jobs, probably. Whether its review of its own work would have been as honest as a separate role's, we doubt.

**Per-iteration scoping in the implementer prompt.** Each implementer call is directed at the smallest unfinished sub-task from the architect's design, or at the specific failing function from the latest verifier output. Without this, an implementer attempt could rewrite the whole file and still pass the "one-attempt-per-call" rule; with it, attempts are narrow and the next iteration picks up where the previous left off. The orchestrator does the iterating; the implementer does one thing well per call.

**Architect produces a sub-task list as part of the design.** Every design note ends with a numbered list of sub-tasks, ordered easiest to hardest, each small enough to land in one edit-verus-iterate cycle. The implementer reads this list and works through it in order. The list is what makes the per-call scoping concrete: "scope to the smallest unfinished sub-task" means something specific because the architect listed them.

**Pre-spec verification via operator-authored witness files.** Before freezing a spec, the operator writes a reference implementation in `<name>_witness.rs` carrying the same spec block, then runs `ralph/check-spec.sh`. If the witness verifies under Verus with no cheat tokens, the spec admits a model and the freeze is safe; if verus rejects it, the operator fixes the spec (or the witness) before the agent loop ever starts. Two of the calibration exercises required mid-run re-freezes: `bounded_log` for a Verus syntax migration, `marzullo` for the missing Helly-1D precondition. Both held the methodology but burned agent cycles on bugs that were ultimately in operator-authored input. The witness check catches that class at operator time. An empirical negative test in [`scripts/test-witness-catches-bad-spec.sh`](https://github.com/ranjithkannank/verus-calibration/blob/main/scripts/test-witness-catches-bad-spec.sh) strips the Helly-1D precondition from a copy of the marzullo witness and confirms verus rejects it.

**A deliberate discovery test on the architect-execution caveat.** Every 1-attempt success since `marzullo` carried the same caveat: the design note pre-named the load-bearing proof construct, so the agent executed a designed proof rather than discovering one. Methodology that only handles executing pre-designed proofs is a much narrower claim than methodology that supports discovery. `sensor_poll_honest` was set up specifically to test the discovery half. Its design note states the proof obligation and the informal mathematical content but deliberately omits the supporting lemmas, the helper-set constructions, the trigger annotations, and the sub-proof structure. The agent verified in one attempt and introduced a new helper lemma using inclusion-exclusion that it recognised from `ft_midpoint`'s playbook entry, a different exercise with a different proof obligation but the same proof family. One data point on one proof family. It moves the designed-vs-discovered axis from "untested, plausible caveat" to "tested once, supports discovery within an established family." Whether the methodology supports discovery on a proof family the playbook does not already document is a separate test on a different exercise.

## What the loop got wrong

Three honest shortcomings worth flagging.

**The hook's spec-preservation check had a known gap.** It used to look for lines whose first token was `requires` or `ensures`, missing the body content underneath. The bounded_log REJECT was caught only by the reviewer because the diff touched body lines. The hook now walks the frozen file by indentation and extracts the complete clause body, not just keyword lines. The fix sits in [`scripts/git-hooks/pre-commit`](https://github.com/ranjithkannank/verus-calibration/blob/main/scripts/git-hooks/pre-commit) with tests under `scripts/test-hook-spec-preservation.sh`.

**The orchestrator used to treat every non-zero claude exit code as verus-failed.** A rate-limited response, a budget cap firing, a network blip, an invocation error: all produced the same exit code and the loop would try the next iteration against the same transient problem. The first quorum_count run burned 27 rate-limited iterations before its outer ceiling fired. The orchestrator now classifies failures by grepping the iteration log against known signatures and exits cleanly on infrastructure failures rather than churning. Tests in [`ralph/test-classify-failure.sh`](https://github.com/ranjithkannank/verus-calibration/blob/main/ralph/test-classify-failure.sh).

**Twelve exercises is not a benchmark.** The vericoding paper has 12,504 specifications. We have twelve. The generalizations we are comfortable drawing are at the level of "this failure mode exists" or "this rule survived this pressure," not "success rate equals X percent." Sample size twelve is sample size twelve.

## Where this fits

Three existing bodies of work touch pieces of this, but not the whole intersection.

The **Verus vericoding benchmark** (Schubert et al., 2025) measures LLM success rates on isolated single-function Verus tasks. One model, one shot, one function, no role split, no autonomous outer loop, no treatment of cheating. The benchmark tells us 44% first-try is the current floor for raw single-shot Verus. This experiment is about what surrounding scaffolding gets a real codebase past it.

**Microsoft Research's AutoVerus and VeruSAGE** ([microsoft/verus-proof-synthesis](https://github.com/microsoft/verus-proof-synthesis)) are doing pieces of the same problem from a different direction. AutoVerus (OOPSLA 2025, [arXiv:2409.13082](https://arxiv.org/abs/2409.13082)) targets small algorithmic code with a three-phase pipeline: inference, refinement, repair. VeruSAGE ([arXiv:2512.18436](https://arxiv.org/abs/2512.18436)) targets large system projects, multi-module code, and ships a benchmark (VeruSAGE-Bench) of 849 tasks drawn from real distributed systems, OS kernels, and storage systems. Both systems are LLM-based on the OpenAI side. Their orchestration is a single-agent pipeline rather than a role split, and they do not appear to ship an explicit no-cheating boundary. The overlap with our work is real and we read their results as a strong baseline for any next step.

**Huntley's [Ralph pattern](https://ghuntley.com/ralph/)** describes the outer-loop shape: fresh context per iteration, file-based memory, agent commits per attempt. It applies the pattern to unsafe-code domains (internal tools, CRUD apps) where the feedback signal is the test suite. Tests pass or fail. No verifier, no no-cheating rule, no separate audit role. The contribution here is keeping the loop shape and changing the feedback signal to a verifier, plus adding the structural pieces (the sandbox, the audit role) that the harder signal requires.

**Karpathy's autoresearch demos** popularized the agent-runs-an-experiment-overnight framing, with a scalar metric (a loss, a benchmark score) and a single agent that proposes, implements, and evaluates in the same context. The metric here is binary (verus exits 0). The agent doing the work is not the agent doing the evaluation. The rules forbidding spec drift live outside the agent's reach. Same overnight-experiment shape, but the proposer, implementer, and auditor are three separate roles by construction.

After the Microsoft work in particular, the claim worth defending is narrower than "no prior public work in this intersection." It is: a separate audit role running on a different model, a mechanical commit-time enforcement layer (the pre-commit hook) running alongside the LLM audit, and per-attempt commits as the unit of evaluation. The combination is what we have not seen in published systems, and it is what the bounded_log REJECT result depended on.

There is also a second contribution worth naming: the operator-intervention case on bounded_log. Most published vericoding results either succeed silently or fail silently. The loop's behavior on the bounded_log conflict, where the agent refused to cheat in either direction, articulated the constraint, named the empowered role, and stopped, is the kind of structured-failure output a trustworthy methodology should produce. A single-shot prompt would either silently apply `final(self)` or silently fail. This run surfaced the conflict, blamed the right party (the operator), and waited.

The multi-module step is now done at the harness level: three exercises (`cross_module_counter`, `counter_multifile`, `counter_producer`) verified first-attempt under the same loop, with small extensions to the path whitelist and the spec-preservation derivation. Three further exercises (`sensor_poll`, `sensor_poll_signed`, `sensor_poll_honest`) compose the BFT primitives at increasing levels of trust-boundary integration; the last is the deliberate discovery test described above. None of the composition exercises import the primitives as crates yet, they re-implement them as sibling modules, and `BACKLOG.md` carries the restructure-as-importable-crates item. Drawing tasks from VeruSAGE-Bench so the results are comparable to existing published numbers, with AutoVerus and VeruSAGE as baselines, is still the obvious benchmark direction at the multi-module scale.

## Reproducing

- Repo: <https://github.com/ranjithkannank/verus-calibration>
- Verus: `0.2026.05.13.fae8859`, arm64-macos binary release.
- Models: `claude-opus-4-7` for all three roles. The implementer was originally `claude-sonnet-4-6` and handled the three calibration exercises competently; switched for the BFT-path exercises starting with `quorum_cert`. The bash script in `ralph/run-exercise.sh` holds the model choices. Subagent frontmatter declares defaults; the outer loop overrides per call.
- All prompts are inlined in `ralph/run-exercise.sh`. All raw verifier output is committed under `logs/<ex>/raw/`. `AGENTS.md` is the rule book; the pre-commit hook in `scripts/git-hooks/pre-commit` is the enforcement.

---

*This post is part of a series on the multi-agent Ralph loop. See [the architecture post](https://ranjithkannan.com/2026/04/10/multi-agent-tdd-loop/), [mutation testing](https://ranjithkannan.com/2026/04/19/pit-mutation-testing-ralph-loop/), [the Final Validator](https://ranjithkannan.com/2026/04/23/final-validator-ralph-loop/), and [integration test contracts](https://ranjithkannan.com/2026/04/27/integration-test-contracts/).*
