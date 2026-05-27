---
layout: post
title: "Wiring a Formal Verifier into an Autonomous Coding Loop"
date: 2026-05-10
related:
  - /2026/04/10/multi-agent-tdd-loop/
  - /2026/05/17/verified-byzantine-tolerant-sensor-fusion/
---

An autonomous coding loop that cannot satisfy its feedback signal except by being correct turns out to be a useful construct. This post is what happens when we wire one to a formal verifier, with three boundaries in place to make accidental cheating impossible, and run fourteen exercises through it.

The previous four posts in this series tightened the feedback signal that an autonomous coding loop runs against. First plain tests. Then [mutation testing](https://ranjithkannan.com/2026/04/19/pit-mutation-testing-ralph-loop/), to check that the tests caught real bugs. Then a [separate auditor](https://ranjithkannan.com/2026/04/23/final-validator-ralph-loop/), so the loop could not grade itself. Then [integration contracts](https://ranjithkannan.com/2026/04/27/integration-test-contracts/), to close the gap between green tests and observable correctness. Each step closed a hole through which a wrong loop could still pass.

A formal verifier is the limit of that progression. It is a feedback signal the agent cannot satisfy except by either weakening the specification or actually being correct. The methodology here is the rule that closes the first path: no spec weakening, audited at three boundaries, on every commit. The empirical question is whether the loop can still produce verified code under that constraint, and whether the methodology holds up when probed for cheats, for proof discovery, for proof invention, and on tasks we did not design.

We picked Verus, because it operates on real Rust rather than on a custom language. The original calibration was three exercises: a sorted binary search, a fixed-capacity append-only log with a frame property, and a Byzantine quorum check whose spec talks about mathematical set cardinality but whose implementation has to walk a `Vec`. The methodology then carried through eleven more exercises across four extension tracks: BFT primitives, multi-module Verus, composing the primitives into a small system, and deliberate discovery and invention tests of the proof process itself. An external-validity probe against Microsoft's VeruSAGE-Bench landed last.

Everything described here lives at <https://github.com/ranjithkannank/verus-calibration>. The per-section links below point at the exact files.

## The setup

Three roles, three [Claude Code subagents](https://github.com/ranjithkannank/verus-calibration/tree/main/.claude/agents).

- **[Architect](https://github.com/ranjithkannank/verus-calibration/blob/main/.claude/agents/architect.md)** (Opus 4.7). Reads the frozen spec. Writes a design note. Does not see verifier output on the first pass.
- **[Implementer](https://github.com/ranjithkannank/verus-calibration/blob/main/.claude/agents/implementer.md)** (Opus 4.7, originally Sonnet 4.6). One attempt per call: edit the file, run verus, log the result.
- **[Reviewer](https://github.com/ranjithkannank/verus-calibration/blob/main/.claude/agents/reviewer.md)** (Opus 4.7). After verus passes, audits the diff against the frozen baseline. Returns `APPROVE` or `REJECT`.

The implementer ran on Sonnet 4.6 for the original three calibration exercises and handled them cleanly. For BFT-path exercises starting with `quorum_cert` the implementer was switched to Opus 4.7. Those proof obligations involve genuine cardinality and pigeonhole reasoning where the model needs to plan over many tokens of internal thinking before producing structured output. We aligned the hardest role to the strongest model, even when that role is the most expensive.

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

## The three boundaries

A loop that runs unattended needs boundaries. There are three. The first two protect the agent's working set from the agent's own shortcuts; the third protects operator-authored material from the agent.

**Content boundary: a pre-commit hook.** Every commit goes through [`scripts/git-hooks/pre-commit`][hook]. It rejects three things.

1. Any staged file outside a path whitelist (`exercises/`, `logs/`, `writeup/`, `ralph/`, `scripts/`, `.claude/`, the named top-level docs).
2. Any cheat token added in `exercises/*.rs`: `assume(`, `#[verifier::external_body]`, `unreachable!()`, `panic!(`, `assume_specification`.
3. Spec drift. Every line in the frozen baseline whose body is part of a `requires` or `ensures` block must appear verbatim in the staged file. New helper `spec fn`s are allowed. Modifying frozen ones is not.

The third check is the one that does the work. For each staged exercise file, the hook diffs against the `spec-frozen-<exercise>` git tag and walks the frozen file by indentation to extract the complete clause body, not just keyword lines. Cosmetic reformatting that touches a frozen clause is rejected as firmly as a semantic weakening. Intentionally strict.

[hook]: https://github.com/ranjithkannank/verus-calibration/blob/main/scripts/git-hooks/pre-commit

**Capability boundary: a Claude Code tool whitelist.** Each `claude -p` call passes a role-scoped `--allowedTools` list and a universal `--disallowedTools` deny set. The deny set includes `WebFetch`, `WebSearch`, `Task` (no sub-agent fan-out), bash patterns for `rm`, `git push/reset/rebase/checkout/restore/config`, `*--no-verify*` so the agent cannot bypass the hook, `curl/wget/ssh`, `brew/npm/pip`, `sudo`. The architect gets only the tools it needs to write a design note. The reviewer does not get `Edit`.

**Operator-territory boundary: witness-file path scoping.** A third boundary, added partway through the project after a corrective arc described in the discovery-and-invention section below. The agent cannot read the operator-authored `<name>_witness.rs` files. These contain reference implementations the operator uses to confirm a spec admits a satisfying model before tagging `spec-frozen-<name>`. They live in the same repo as the exercise files, so the path whitelist accepts them; the implementer's `--allowedTools` includes generic `Read`, so without an explicit deny the witness was readable. The deny patterns now scope generic reads: `Read(**/*_witness*)`, `Read(**/*_witness/**)`, the `Glob` / `Grep` equivalents, and `Bash(cat|grep|rg|head|tail|ls *_witness*)`. The boundary's job is structural separation of operator work product from the agent's working set.

To state what these are not: not process isolation, not a network namespace, and a deliberately adversarial agent could find paths around them. The agents in this experiment are not adversarial. The boundaries' job is to make it impossible for an honest worker to cheat accidentally. To silently weaken a spec because the verifier complained, to slip an `assume` past the reviewer because it was tired, or to crib from a reference implementation that wasn't meant to be visible. The first two boundaries held across all fourteen DONE exercises. The third boundary was the corrective response to the first invention-test run, and the run history is preserved on disk as honest evidence.

## What happened

Results at a glance:

| Exercise              | Status      | Attempts            | Track            |
|-----------------------|-------------|---------------------|------------------|
| binary_search         | DONE        | 1                   | Calibration      |
| bounded_log           | DONE        | 1 (post re-freeze)  | Calibration      |
| quorum_count          | DONE        | 2                   | Calibration      |
| quorum_cert           | DONE        | 6                   | BFT primitives   |
| ft_midpoint           | DONE        | 7                   | BFT primitives   |
| marzullo              | DONE        | 1 (post re-freeze)  | BFT primitives   |
| cross_module_counter  | DONE        | 1                   | Multi-module     |
| counter_multifile     | DONE        | 1                   | Multi-module     |
| counter_producer      | DONE        | 1                   | Multi-module     |
| sensor_poll           | DONE        | 1                   | Composition      |
| sensor_poll_signed    | DONE        | 1                   | Composition      |
| sensor_poll_honest    | DONE        | 1 (audit-confirmed) | Discovery test   |
| counter_filler        | DONE        | 1 (audit-confirmed) | Discovery test   |
| vec_swap              | INVALIDATED | 1                   | Invention test   |
| vec_swap_v2           | INVALIDATED | 1                   | Invention test   |
| swap_multiset         | DONE        | 1                   | Invention test   |

A caveat to keep in mind through the rest of this section. Every 1-attempt success after `marzullo` carried the same shape: the architect's design note pre-named the load-bearing proof construct (the loop invariant, the bridging lemma, the helper-set shape), so the agent executed a designed proof rather than discovering one. The deliberate discovery tests (`sensor_poll_honest`, `counter_filler`) and the invention tests (`swap_multiset`) were set up specifically to probe the discovery and invention halves of that caveat; they get their own subsection below.

### binary_search

The easiest. A verified binary search with a sortedness precondition and the standard found/not-found postconditions. The architect's design predicted every load-bearing piece: a half-open window `[lo, hi)`, an overflow-safe `mid = lo + (hi - lo) / 2`, and the five-conjunct loop invariant whose two `forall` exclusion ranges tile the index space on loop exit. The implementer wrote a matching body. First try, verus passed, reviewer approved.

What the run set up was cross-exercise memory. The implementer appended four discovered patterns to `AGENTS.md` on its own — the `decreases` requirement on every `while`, the `assert forall ... by { ... }` trigger pattern for sortedness, the half-open underflow note, and the five-conjunct invariant shape. The role file has an explicit escape hatch for findings, and the implementer used it. Future exercises read what this one wrote.

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

The hardest of the three calibration exercises, by design. The spec defines `distinct_count(voters) = voters.to_set().len()`, mathematical set cardinality. The implementation has to walk a `Vec<NodeId>` and count distinct elements with a concrete data structure; the proof has to bridge those worlds.

The architect proposed a bitmap-backed approach (`Vec<bool>` of length `n` plus a `u64` counter, one linear pass). The proof reduces to three helper lemmas: a prefix-step identity, a concrete-to-abstract bridge from `push` to `insert`, and an idempotence claim for re-insertion. On attempt 1 the implementer wrote about 330 lines covering the algorithm and all three architect-predicted lemmas plus two more it added on its own. Verus returned `5 verified, 2 errors`: the empty-subrange `to_set().len()` was not seen as 0 at loop entry, and the `count <= n` bound after increment needed a pigeonhole argument the proof did not supply.

Attempt 2 did something we had not predicted. Rather than guessing at lemma names, the implementer grepped the local `vstd` source for relevant helpers — `lemma_len_subset`, `lemma_int_range`, `axiom_set_insert_len`. It noticed the type mismatch between `Set<NodeId>` and `Set<int>` and wrote a new NodeId-analogue helper [`lemma_range_nodeid_len`][quorum-rs] by structural recursion on `u32`. The implementer also caught its own regression mid-attempt (a `u32 - 1` int/nat cast in the new proof code) and patched it in the same iteration. Final: `8 verified, 0 errors`. The agent did not have to do any of this. It could have left `assert(count <= n)` in place, weakened the invariant, or added an `assume` (and been caught by the hook). It wrote a recursive cardinality lemma instead.

[quorum-rs]: https://github.com/ranjithkannank/verus-calibration/blob/main/exercises/quorum_count.rs

The reviewer's audit took under a minute. Five-point checklist passed with specific line citations, plus a cross-exercise observation: the implementer leans heavily on `=~=` extensional equality and `choose` witnesses to push set/seq reasoning through the SMT solver, a pattern that recurred in `bounded_log`. That observation went into the playbook. The audit role doing more than gatekeeping.

### The extension tracks

After the three calibration exercises, the methodology carried into four extension tracks. The per-exercise narratives for each track live elsewhere.

**BFT primitives: `quorum_cert`, `ft_midpoint`, `marzullo`.** The first BFT-shaped exercise verified a quorum certificate library with a safety lemma. The architect proposed pigeonhole-via-contradiction and a bitmap-backed structural check; the implementer worked through six narrow iterations and landed `12 verified, 0 errors`. Three patterns from this run went into the playbook: pigeonhole-via-contradiction (wrap the negation in `if !(exists ...) { ... assert(false); }` and close the cardinality contradiction with `lemma_len_subset`), the `lemma_fundamental_div_mod` trick for threshold arithmetic Verus's `nonlinear_arith` discharge does not know, and a usage note on the finiteness-hypothesis direction of `lemma_len_subset`. The full proof is in [`exercises/quorum_cert.rs`](https://github.com/ranjithkannank/verus-calibration/blob/main/exercises/quorum_cert.rs); the playbook entries live in [`AGENTS.md`](https://github.com/ranjithkannank/verus-calibration/blob/main/AGENTS.md).

The two sensor-fusion exercises (`ft_midpoint`, the brute-force-scan algorithm with inclusion-exclusion over set cardinalities; `marzullo`, the interval generalisation) live in the [sensor-fusion post](https://ranjithkannan.com/2026/05/17/verified-byzantine-tolerant-sensor-fusion/). `marzullo` was the second operator-intervention case: the first frozen spec omitted the Helly-1D precondition (all correct sensors' intervals share at least one common point), the agent surfaced it via constructive counterexample, the operator re-froze, and the loop converged in one attempt.

**Multi-module Verus: `cross_module_counter`, `counter_multifile`, `counter_producer`.** Three exercises that stretched the harness from single-file to multi-module and multi-file layouts. `cross_module_counter` is two nested `mod` blocks inside one `verus!{}`. `counter_multifile` splits the same algorithm into `main.rs` + `counter.rs` in a directory layout; the pre-commit hook's path whitelist, the spec-preservation step's exercise-name derivation, and the orchestrator's edit-scope variable all had to learn the multi-file shape. `counter_producer` adds a third module whose loop invariant carries facts the counter module cannot expose. All three verified first-attempt under the same loop. Caveat noted in the introduction to this section: the architect's design pre-named the load-bearing loop invariants in each case.

**Composition of BFT primitives: `sensor_poll`, `sensor_poll_signed`, `sensor_poll_honest`.** Three exercises composing `quorum_cert`'s distinct-and-signed structural check with `marzullo`'s interval fusion, at increasing levels of trust-boundary integration. The first added the structural check; the second threaded the cryptographic trust boundary at the spec layer; the third strengthened the postcondition with an honest-voter guarantee and doubled as a deliberate discovery test for the methodology. The dedicated discussion, including the honest-scope items about composition (the primitives are *ported* as sibling modules, not imported as crates; signature verification stays at the spec layer; `ft_midpoint` is unused), lives in the [sensor-fusion post](https://ranjithkannan.com/2026/05/17/verified-byzantine-tolerant-sensor-fusion/).

### Discovery and invention tests

The architect-pre-named caveat is methodologically important. Methodology that only handles executing pre-designed proofs is a much narrower claim than methodology that supports discovery or invention. Two distinct tests probe the difference.

A **discovery test** gives the architect an obligation and informal mathematical content, but deliberately omits the supporting lemmas, the helper-set constructions, the trigger annotations, and the sub-proof structure. The agent has to recognise that a proof family from the playbook applies to a new obligation in a different exercise.

An **invention test** gives the architect a spec for a proof family the playbook does not document at all. The agent has to invent the supporting structure.

Two discovery tests have run.

`sensor_poll_honest`'s design note states the obligation (an honest-voter clause: there exists a correct sensor whose interval contains the agreed point) and the informal `n - f` plus `n - f` overlap argument, but does not name lemmas or helper-set constructions. The agent verified in one attempt and introduced a new helper lemma `lemma_honest_supporter_exists`, using inclusion-exclusion via `vstd::set_lib::lemma_set_intersect_union_lens` against a universe-finite bridge via `lemma_int_range` and `lemma_len_subset`. The proof family was the one `ft_midpoint`'s playbook entry already documented; the agent recognised it and applied it to a new obligation in a different exercise.

`counter_filler` is a `counter_producer`-shaped cross-module exercise whose design note states the obligation only and explicitly warns against copying the parent exercise's loop invariant. The agent verified in one attempt and produced a 4-conjunct target-bounded invariant in place of the 6-conjunct counter-bounded form the playbook documented. Different proof family, same shape of result.

Both discovery tests were subsequently re-audited. The audit reset each exercise to its spec-frozen scaffold, stripped its prior playbook summary from `AGENTS.md`, and re-ran the loop under the hardened whitelist described below. Both verified in one attempt with solutions structurally identical to the originals. Two data points, two distinct proof families, audit-confirmed.

The invention-test arc produced the operator-territory boundary, the third sandbox layer described above. The first invention test, `vec_swap` (a `Vec<T>::swap`-style exercise with a multiset-preservation postcondition the playbook did not anchor), passed `verus` in one attempt. The reviewer accepted. The implementer's iter-1 commit message read "port witness proof", and its self-authored playbook entry recorded:

> The architect-owned witness file was visible to the implementer and gave the proof skeleton verbatim. For pure invention measurement, future exercises should hide the witness.

That entry, written by the agent against its own run, is the cleanest possible flagging of the gap. We marked `vec_swap` INVALIDATED and kept it on disk as evidence rather than rewriting. The `DISALLOWED_TOOLS` list was then extended with the witness-deny patterns enumerated in the operator-territory boundary section. An empirical probe at [`scripts/probe-witness-deny.sh`](https://github.com/ranjithkannank/verus-calibration/blob/main/scripts/probe-witness-deny.sh) invokes `claude -p` with the same disallowed-tools array and asks the agent to read a witness file via six different paths; each is blocked, with the agent's own verdict: "I was unable to read any of the witness file's contents through any of the attempted methods."

A second attempt to re-run an invention test (`vec_swap_v2`) was also INVALIDATED, but for a different reason. The operator's `cp vec_swap.rs vec_swap_v2.rs` copied the agent's already-filled body from the first run as the scaffold, so the implementer (correctly) noted "no edits made" and exited. Honest history, kept on disk.

The third attempt (`swap_multiset`, same spec as `vec_swap` but a hand-typed scaffold and the hardened whitelist) verified in one attempt with a proof structurally different from the operator-authored witness. The agent did not have the witness available to crib from. It found a different proof path. Invention test passed, one data point on one proof family.

### External-validity probe

Two batches against tasks we did not design. The load-bearing result is batch 2: eight tasks attempted with deliberately neutral design notes, six verified, two distinct blocking findings documented. *Honest scope first:* this evidence says the methodology travels at small-to-medium proof-fn and exec sizes (171 B to 3.7 KB). It does not yet say it travels at VeruSAGE-Bench's full scale, which includes 7-24 KB examples we deliberately deferred to a later batch.

Batch 2's design notes state the obligation and a standard sub-task ordering only — no tooling-family names, no lemma names, no proof-structure suggestions, no cross-references to internal exercises by name. The eight tasks were drawn from `tasks-sampled-100/` across six source projects in Microsoft's [VeruSAGE-Bench](https://github.com/microsoft/verus-proof-synthesis): `AL` (anvil-library), `IR` (ironkv), `MA` (memory-allocator), `NR` (nrkernel, two tasks), `OS` (atmosphere, an OS kernel), and `NO` (node-replication). The mix is intentional: pure ghost proof fns over data structures, arithmetic chains, exec functions with loops, and one stress test on size.

Five tasks landed single-attempt: NR `set_of_first_n_nat_is_finite`, AL `submap_of_a_finite_map_is_finite`, MA `shift_is_div`, IR `lemma_if_everything_in_seq_satisfies_filter_then_filter_is_identity`, NR `lemma_maxphyaddr_facts` (after a harness fix described below), and NO `get_fresh_nat_not_in`. One landed two-attempt: IR `singleton_seq_to_set_is_singleton_set`. The two-attempt arc is the methodology-grade data point: attempt 1 tried the obvious `=~=` extensional equality, watched verus reject it, and recorded in its own notes that "`=~=` alone doesn't trigger the `to_set` axiom needed to expand `seq![x].to_set().contains(y)`"; attempt 2 closed via vstd's `lemma_push_to_set_commute` plus two extensional-equality bridges. The agent read its failure, named the missing axiom, found the vstd lemma family that supplied it, and applied it without operator intervention.

Two tasks blocked, with distinct findings.

The first, `NR__definitions_u__lemma_maxphyaddr_facts`, was initially blocked by a real gap in our own pre-commit hook. The upstream task ships with `#[verifier(external_body)]` on an opaque-constant declaration, the standard Verus pattern for declaring a trust-boundary value constrained by a separate `pub axiom fn`. Our hook's cheat-token detector treated every line of the brand-new scaffold file as "added" (no prior baseline to diff against) and rejected the `external_body` marker as agent-introduced cheating. The `check-spec.sh` witness check had the symmetric gap, refusing witnesses that mirrored the exercise file's baseline `external_body`. Both layers were fixed in a single commit ([`041df77`](https://github.com/ranjithkannank/verus-calibration/commit/041df77)). The pre-commit hook now diffs against `spec-frozen-<exercise>` when the tag exists and treats the scaffold commit as a baseline event; the witness check now counts cheat-token occurrences in the witness minus the count in the paired exercise file, flagging only excess. Both fixes preserve the original boundary intent: agents still cannot introduce new cheat tokens. The task ran cleanly under the fix and verified in one attempt. The blocked-then-fixed-then-verified arc is itself the empirical demonstration of the boundary refinement described in [What the loop got right](#what-the-loop-got-right) below.

The second, `OS__array__impl4__init2none`, was blocked by an upstream-downstream Verus version mismatch. The task's `ensures` clauses use pre-`final(self)` syntax that the current Verus build rejects; the witness fails to verify under our Verus, so the agent loop never ran. This is exactly the class of finding the witness mechanism is designed to catch. Pinning Verus to VeruSAGE-Bench's authored version is the right scientific next move and is queued.

Two findings of qualitatively different shape from one batch are useful. The first surfaced a real harness adaptation need that no prior run had stressed; the second confirms that the witness-based pre-spec check generalises to externally-authored specs as well as our own. Both are recorded as `logs/<task>/blocked.md` (with the first task's original blocked marker retained as `blocked.first-attempt.md` after the harness fix unblocked it).

Batch 1 ran earlier as historical context. Before batch 2 we ran a smaller probe to test that the harness accepted the upstream task shape at all: `MA__bin_sizes__mul_assoc` (a pure `proof fn` arithmetic identity) and `VE__utils__init_vec_u8` (an exec function from the vest verified-serializer's utilities). Both verified in one implementer attempt. Both shipped with operator-authored design notes that named the relevant Verus tooling family before the implementer ran, so batch 1's evidence was about harness adaptation only, not the methodology claim. The harness adaptation surface stayed tiny in batch 2: a per-batch handful of iteration-cap entries in `ralph/run-exercise.sh`'s per-exercise `case` block, plus the `AGENTS.md` section naming the external-validity track as separate from the numbered internal-exercise sequence.

Direct comparison to AutoVerus and VeruSAGE on these specific tasks is not yet possible: the upstream README states the per-task leaderboard is "TO COME". The AutoVerus and VeruSAGE papers report aggregate pass rates across the full benchmarks (Verus-Bench: 150 algorithmic tasks; VeruSAGE-Bench: 849 repository-level tasks); n=10 across both batches is still too small to set against either aggregate. The honest aggregate sentence is "this harness verified eight of ten tasks attempted from VeruSAGE-Bench under increasingly neutral design notes; more tasks are needed before 'comparable to AutoVerus / VeruSAGE' is a defensible phrasing."

The natural next batch is a larger size tier: five to ten tasks in the 2-10 KB range, mostly from `AC` (anvil controller, temporal logic) and the larger `ST` (storage) / `OS` files. Negative results from that run are more informative than positive ones; they tell us where the methodology actually stops travelling.

## What the loop got right

Nine claims we are willing to defend from this run.

**The no-spec-weakening rule held under pressure.** Both the mechanical hook and the semantic reviewer caught real violations. The hook caught a body-content modification on bounded_log that no other rule matched. The reviewer caught a spec-shape modification that the hook missed because its check was keyword-prefix only at the time. Two layers that fail differently are the boundary. One layer alone was not enough.

**Per-attempt commits made the methodology auditable.** Every implementer iteration is its own commit, with the verifier output captured under `logs/<ex>/raw/`. The reviewer's audits cite specific HEAD lines. The git log is a usable timeline for the failure taxonomy. Git was sufficient, once the orchestrator was disciplined about committing on each attempt.

**The role split bought what it advertised.** The architect produced substantive design notes that the implementers followed. The implementers self-diagnosed their own failures and named follow-up plans rather than thrashing. The reviewer audited the diff with checklist rigor and contributed cross-exercise pattern observations. A single Opus call could have done all three jobs, probably. Whether its review of its own work would have been as honest as a separate role's, we doubt.

**Per-iteration scoping plus an architect sub-task list kept iterations narrow.** Each implementer call is directed at the smallest unfinished sub-task from the architect's design, or at the specific failing function from the latest verifier output. The architect's design note ends in a numbered sub-task list ordered easiest to hardest, each small enough to land in one edit-verus-iterate cycle. The orchestrator does the iterating; the implementer does one thing well per call.

**Pre-spec witness verification catches operator-time spec bugs.** Before tagging `spec-frozen-<name>`, the operator runs `ralph/check-spec.sh` against a reference implementation at `<name>_witness.rs`. A verus-passing witness means the spec admits a model; a verus-rejecting witness means the spec or the witness is wrong, and the operator fixes one before the agent loop runs. Two calibration exercises (`bounded_log`, `marzullo`) burned agent cycles on spec bugs this check would have caught at operator time; none of the post-witness-check exercises did. An empirical negative test in [`scripts/test-witness-catches-bad-spec.sh`](https://github.com/ranjithkannank/verus-calibration/blob/main/scripts/test-witness-catches-bad-spec.sh) confirms verus rejects a marzullo witness with the Helly-1D precondition stripped.

**Deliberate discovery tests passed, audit-confirmed.** Two exercises set up specifically to test whether the agent can recognise a known proof family without the architect naming the construct (`sensor_poll_honest`, `counter_filler`) verified in one attempt each, then survived a re-audit under the hardened whitelist with each exercise's prior playbook summary stripped from `AGENTS.md`. Both audits passed in one attempt with solutions structurally identical to the originals. Two data points on two distinct proof families. The methodology supports discovery within an established proof family.

**Witness-access hardening turned the witness into a structural boundary, with an empirical probe to verify it.** The corrective arc from `vec_swap` (described in the discovery-and-invention section) produced the operator-territory boundary, the third sandbox layer. The deny patterns are verified continuously by the probe script. The first clean invention test under the hardened whitelist (`swap_multiset`) produced a proof structurally different from the operator-authored witness; the invention claim is one data point on one proof family.

**The methodology travels on tasks we did not design, at small-to-medium sizes.** *Caveat first:* the claim is bounded by size. Across two batches against VeruSAGE-Bench tasks (171 B to 3.7 KB), eight of ten attempts verified, and six of those eight passed under deliberately neutral design notes that did not name the Verus tooling family before the implementer ran. The methodology travels at this scale. It does not yet travel at VeruSAGE-Bench's full scale, which includes 7-24 KB examples deliberately deferred. The detailed batch 2 framing, including the two distinct blocking findings and the next experiment, is in the external-validity section above.

**The cheat-token boundary is now scaffold-aware.** The external-validity probe surfaced a real gap in the original hook design: it treated every line of a brand-new exercise file as "added", silently equating baseline scaffold content with agent introduction. For internally-authored exercises this was harmless because none of our scaffolds carried cheat tokens, but VeruSAGE-Bench tasks that legitimately declare opaque constants via `#[verifier(external_body)]` were rejected at scaffold time. Two surgical fixes in commit [`041df77`](https://github.com/ranjithkannank/verus-calibration/commit/041df77): `scripts/git-hooks/pre-commit` now diffs against `spec-frozen-<exercise>` when the tag exists and skips cheat-token detection on the scaffold commit (no prior baseline to compare against); `ralph/check-spec.sh` now counts cheat-token occurrences in the witness minus the count in the paired exercise file, flagging only excess. The original boundary intent is preserved exactly: agents cannot introduce new cheat tokens. The blocked-then-fixed-then-verified arc on `NR__definitions_u__lemma_maxphyaddr_facts` (commits [`e2a2cb6`](https://github.com/ranjithkannank/verus-calibration/commit/e2a2cb6) blocked, [`041df77`](https://github.com/ranjithkannank/verus-calibration/commit/041df77) fix, [`248a6cc`](https://github.com/ranjithkannank/verus-calibration/commit/248a6cc) DONE) is the empirical demonstration.

## What the loop got wrong

Four honest shortcomings worth flagging. Two have been fixed and are kept here as past-tense entries with links to the fix; one is the witness-read gap that produced the third sandbox layer; one is the irreducible "sample size is small" limit.

**The hook's spec-preservation check had a known gap.** It used to look for lines whose first token was `requires` or `ensures`, missing the body content underneath. The bounded_log REJECT was caught only by the reviewer because the diff touched body lines. The hook now walks the frozen file by indentation and extracts the complete clause body, not just keyword lines. The fix sits in [`scripts/git-hooks/pre-commit`](https://github.com/ranjithkannank/verus-calibration/blob/main/scripts/git-hooks/pre-commit) with tests under `scripts/test-hook-spec-preservation.sh`.

**The orchestrator used to treat every non-zero claude exit code as verus-failed.** A rate-limited response, a budget cap firing, a network blip, an invocation error: all produced the same exit code and the loop would try the next iteration against the same transient problem. The first `quorum_count` run burned 27 rate-limited iterations before its outer ceiling fired. The orchestrator now classifies failures by grepping the iteration log against known signatures and exits cleanly on infrastructure failures rather than churning. Tests in [`ralph/test-classify-failure.sh`](https://github.com/ranjithkannank/verus-calibration/blob/main/ralph/test-classify-failure.sh).

**The original implementer whitelist allowed witness reads.** The discovery-test and invention-test framings depend on the agent not reading the operator-authored witness file. The original tool whitelist had generic `Read` and `Glob` with no path qualifier, so the witness was visible. The `vec_swap` invention test made this concrete: the agent's iter-1 commit was titled "port witness proof", and the agent's own playbook entry recorded the gap honestly. We treat that self-flagging as the cleanest possible signal. The fix is the operator-territory boundary described above; the original gap is kept here as honest history, and the `vec_swap` / `vec_swap_v2` rows in the results table stay INVALIDATED as evidence rather than being rewritten.

**Fourteen exercises plus ten external-validity tasks is not a benchmark.** The vericoding paper has 12,504 specifications. We have twenty-four total. The generalisations we are comfortable drawing are at the level of "this failure mode exists" or "this rule survived this pressure," not "success rate equals X percent." The external-validity batch 2 result is meaningful within its declared scope (small-to-medium proof-fn and exec sizes); aggregate-level claims against AutoVerus or VeruSAGE need an aggregate-scale run.

## Where this fits

Three existing bodies of work touch pieces of this, but not the whole intersection.

The **Verus vericoding benchmark** (Schubert et al., 2025) measures LLM success rates on isolated single-function Verus tasks. One model, one shot, one function, no role split, no autonomous outer loop, no treatment of cheating. The benchmark tells us 44% first-try is the current floor for raw single-shot Verus. This experiment is about what surrounding scaffolding gets a real codebase past it.

**Microsoft Research's AutoVerus and VeruSAGE** ([microsoft/verus-proof-synthesis](https://github.com/microsoft/verus-proof-synthesis)) are doing pieces of the same problem from a different direction. AutoVerus (OOPSLA 2025, [arXiv:2409.13082](https://arxiv.org/abs/2409.13082)) targets small algorithmic code with a three-phase pipeline: inference, refinement, repair. VeruSAGE ([arXiv:2512.18436](https://arxiv.org/abs/2512.18436)) targets large system projects, multi-module code, and ships VeruSAGE-Bench, the 849-task benchmark we drew our external-validity probe from. Both systems are LLM-based on the OpenAI side. Their orchestration is a single-agent pipeline rather than a role split, and they do not appear to ship an explicit no-cheating boundary. The overlap with our work is real and we read their results as the baseline once an aggregate-scale run lets us compare.

**Huntley's [Ralph pattern](https://ghuntley.com/ralph/)** describes the outer-loop shape: fresh context per iteration, file-based memory, agent commits per attempt. It applies the pattern to unsafe-code domains (internal tools, CRUD apps) where the feedback signal is the test suite. Tests pass or fail. No verifier, no no-cheating rule, no separate audit role. The contribution here is keeping the loop shape and changing the feedback signal to a verifier, plus adding the structural pieces (the three boundaries, the audit role) that the harder signal requires.

**Karpathy's autoresearch demos** popularised the agent-runs-an-experiment-overnight framing, with a scalar metric (a loss, a benchmark score) and a single agent that proposes, implements, and evaluates in the same context. The metric here is binary (verus exits 0). The agent doing the work is not the agent doing the evaluation. The rules forbidding spec drift live outside the agent's reach. Same overnight-experiment shape, but the proposer, implementer, and auditor are three separate roles by construction, and the operator's reference implementation is structurally separated from the agent's working set.

After the Microsoft work in particular, the claim worth defending is narrower than "no prior public work in this intersection." It is: a separate audit role running on a different model, a mechanical commit-time enforcement layer (the pre-commit hook) running alongside the LLM audit, per-attempt commits as the unit of evaluation, and operator-authored witness files used both for pre-spec verification and as a structural boundary the agent cannot reach. The combination is what we have not seen in published systems.

There is also a second contribution worth naming: the operator-intervention cases on `bounded_log` and `marzullo`. Most published vericoding results either succeed silently or fail silently. The loop's behaviour on both of these specs, where the agent refused to cheat in either direction, articulated the constraint, named the empowered role, and stopped, is the kind of structured-failure output a trustworthy methodology should produce. A single-shot prompt would either silently weaken the postcondition or silently fail. The runs surfaced the conflict, blamed the right party (the operator), and waited. The pre-spec witness check has since removed that class of operator-time mistakes.

The two specific next experiments. First, a verified Byzantine agreement primitive (deferred for now; see [`BACKLOG.md`](https://github.com/ranjithkannank/verus-calibration/blob/main/BACKLOG.md) in the repo for why) and a hardware-deployed demonstration of the sensor-fusion algorithms running on dissimilar redundant boards under live fault injection. Second, the methodology external-validity run described above: five to ten more VeruSAGE-Bench tasks with deliberately neutral design notes, no tooling hints, just the obligation and the upstream context, and report whatever happens. Negative results from that run are more informative than positive ones, because they tell us where the methodology actually stops travelling.

## Reproducing

- Repo: <https://github.com/ranjithkannank/verus-calibration>
- Verus: `0.2026.05.13.fae8859`, arm64-macos binary release.
- Models: `claude-opus-4-7` for all three roles. The implementer was originally `claude-sonnet-4-6` and handled the three calibration exercises competently; switched to Opus for the BFT-path exercises starting with `quorum_cert`. The bash script in `ralph/run-exercise.sh` holds the model choices. Subagent frontmatter declares defaults; the outer loop overrides per call.
- All prompts are inlined in `ralph/run-exercise.sh`. All raw verifier output is committed under `logs/<ex>/raw/`. `AGENTS.md` is the rule book; the pre-commit hook in `scripts/git-hooks/pre-commit` is the enforcement; the witness-deny patterns are in the `DISALLOWED_TOOLS` array at the top of `ralph/run-exercise.sh`, and the probe at `scripts/probe-witness-deny.sh` verifies them.
