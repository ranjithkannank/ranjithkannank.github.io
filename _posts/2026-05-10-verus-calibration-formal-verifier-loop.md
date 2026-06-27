---
layout: post
title: "Wiring a Formal Verifier into an Autonomous Coding Loop"
date: 2026-05-10
related:
  - /2026/04/10/multi-agent-tdd-loop/
  - /2026/05/17/verified-byzantine-tolerant-sensor-fusion/
---

An autonomous coding loop that cannot satisfy its feedback signal except by being correct turns out to be a useful construct. This post is what happens when we wire one to a formal verifier, with three boundaries in place to make accidental cheating impossible, and run fourteen exercises through it.

The previous four posts in this series tightened the feedback signal that an autonomous coding loop runs against. First plain tests. Then [mutation testing](https://ranjithkannan.com/2026/04/19/pit-mutation-testing-ralph-loop/). Then a [separate auditor](https://ranjithkannan.com/2026/04/23/final-validator-ralph-loop/). Then [integration contracts](https://ranjithkannan.com/2026/04/27/integration-test-contracts/). Each step closed a hole through which a wrong loop could still pass.

A formal verifier is the limit of that progression. It is a feedback signal the agent cannot satisfy except by either weakening the specification or actually being correct. The methodology here is the rule that closes the first path: no spec weakening, audited at three boundaries, on every commit. The empirical question is whether the loop can still produce verified code under that constraint, and whether the methodology holds up when probed for cheats, for proof discovery, for proof invention, and on tasks we did not design.

We picked Verus because it operates on real Rust. The original calibration was three exercises: a sorted binary search, a fixed-capacity append-only log with a frame property, and a Byzantine quorum check whose spec talks about mathematical set cardinality but whose implementation has to walk a `Vec`. The methodology then carried through eleven more exercises across four extension tracks (BFT primitives, multi-module Verus, composing the primitives, discovery and invention tests). An external-validity probe against Microsoft's VeruSAGE-Bench landed last.

## The setup

Three roles, three Claude Code subagents.

- **Architect** (Opus 4.7). Reads the frozen spec. Writes a design note. Does not see verifier output on the first pass.
- **Implementer** (Opus 4.7, originally Sonnet 4.6). One attempt per call: edit the file, run verus, log the result.
- **Reviewer** (Opus 4.7). After verus passes, audits the diff against the frozen baseline. Returns `APPROVE` or `REJECT`.

The reviewer is a separate role for the same reason the [previous post](https://ranjithkannan.com/2026/04/23/final-validator-ralph-loop/) argued for splitting audit from decision: a fresh-context audit catches what the author cannot.

The three roles are wired together by a Ralph-style outer loop in bash. The loop reads state from filesystem artifacts and fires one `claude -p` call per iteration with fresh context each time. Memory lives in `AGENTS.md`, the design note, `attempts.md`, and git history. The state machine:

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

State is inferred from the filesystem on every iteration. The loop is fully resumable: kill it, re-run, it picks up where it left off.

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

The `--no-session-persistence` flag is what makes this a Ralph loop. Each iteration starts from a clean conversational slate, with only what is on disk to ground it. Each implementer attempt is its own git commit, scoped narrowly to the next unfinished sub-task from the architect's design or the specific failing function from the latest verifier output. The orchestrator iterates; the implementer does one thing per call.

## The three boundaries

A loop that runs unattended needs boundaries. There are three. The first two protect the agent's working set from the agent's own shortcuts; the third protects operator-authored material from the agent.

**Content boundary: a pre-commit hook.** Every commit goes through a `scripts/git-hooks/pre-commit` hook. It rejects three things.

1. Any staged file outside a path whitelist (`exercises/`, `logs/`, `writeup/`, `ralph/`, `scripts/`, `.claude/`, the named top-level docs).
2. Any cheat token added in `exercises/*.rs`: `assume(`, `#[verifier::external_body]`, `unreachable!()`, `panic!(`, `assume_specification`.
3. Spec drift. Every line in the frozen baseline whose body is part of a `requires` or `ensures` block must appear verbatim in the staged file. For each staged exercise file, the hook diffs against the `spec-frozen-<exercise>` git tag and walks the frozen file by indentation to extract the complete clause body. Cosmetic reformatting that touches a frozen clause is rejected as firmly as a semantic weakening.

**Capability boundary: a Claude Code tool whitelist.** Each `claude -p` call passes a role-scoped `--allowedTools` list and a universal `--disallowedTools` deny set. The deny set includes `WebFetch`, `WebSearch`, `Task` (no sub-agent fan-out), `*--no-verify*` so the agent cannot bypass the hook, plus the usual network and install patterns. The architect gets only the tools it needs to write a design note. The reviewer does not get `Edit`.

**Operator-territory boundary: witness-file path scoping.** A third boundary, added partway through the project after a corrective arc described in the discovery-and-invention section below. The agent cannot read the operator-authored `<name>_witness.rs` files (reference implementations used to confirm a spec admits a satisfying model). The deny patterns: `Read(**/*_witness*)`, `Read(**/*_witness/**)`, the `Glob` / `Grep` equivalents, and `Bash(cat|grep|rg|head|tail|ls *_witness*)`.

None of this is process isolation; an adversarial agent could find paths around it. The boundaries' job is to make it impossible for an honest worker to cheat accidentally. The first two boundaries held across all fourteen DONE exercises. The third was the corrective response to the first invention-test run.

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

A caveat: every 1-attempt success after `marzullo` carried the same shape. The architect's design note pre-named the load-bearing proof construct, so the agent executed a designed proof rather than discovering one. The discovery tests (`sensor_poll_honest`, `counter_filler`) and the invention tests (`swap_multiset`) probe the discovery and invention halves below.

### binary_search

The easiest. A verified binary search with a sortedness precondition and the standard found/not-found postconditions. The architect's design predicted every load-bearing piece: a half-open window `[lo, hi)`, an overflow-safe `mid = lo + (hi - lo) / 2`, and the five-conjunct loop invariant whose two `forall` exclusion ranges tile the index space on loop exit. First try, verus passed, reviewer approved.

What this run set up was cross-exercise memory: the implementer appended four discovered patterns to `AGENTS.md` on its own (the `decreases` requirement on every `while`, the `assert forall ... by { ... }` trigger pattern for sortedness, the half-open underflow note, the five-conjunct invariant shape). Future exercises read what this one wrote.

### bounded_log

This exercise pressed hardest on the methodology. The spec is a fixed-capacity append-only log; `append`'s postcondition includes a frame property.

On the first attempt, the implementer wrote `final(self)` everywhere in the `ensures` clause of `append` instead of bare `self`. Verus accepted: `4 verified, 0 errors`. The reviewer audited against the frozen tag and rejected. The diff hunk that triggered the rejection:

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

The full audit:

> Even granting the implementer's claim that `final(self)` is semantically equivalent to the post-state `self`, this is not byte-identical and therefore falls under rule 1.

On the second attempt, the implementer restored bare `self`. Verus rejected it with a `&mut self` postcondition disambiguation error. Both paths now violated a rule. The implementer wrote a structured blocker report:

> | Constraint                    | What it requires                                                                |
> |-------------------------------|---------------------------------------------------------------------------------|
> | Frozen spec (reviewer rule 1) | `self.well_formed()`, `self.capacity()`, `self.view()` (bare `self`)            |
> | Verus 0.2026.05.13 syntax     | `final(self).well_formed()`, `final(self).capacity()`, `final(self).view()`     |
>
> These two constraints are mutually exclusive. No implementer-level change can satisfy both simultaneously. Only the architect (per AGENTS.md) is empowered to re-freeze the spec.

The agent articulated the conflict, named the role empowered to resolve it, and stopped. The frozen spec was wrong; it predated the Verus version on the machine. We re-froze the baseline and the loop converged in one attempt. The methodology held up against operator error in spec authorship.

### quorum_count

The hardest of the three calibration exercises. The spec defines `distinct_count(voters) = voters.to_set().len()`, mathematical set cardinality; the implementation walks a `Vec<NodeId>` with a bitmap. The architect proposed a `Vec<bool>` of length `n` plus a `u64` counter, with three helper lemmas (prefix-step, push-to-insert bridge, idempotent re-insertion).

Attempt 1 returned `5 verified, 2 errors`: the empty-subrange `to_set().len()` was not seen as 0 at loop entry, and the `count <= n` bound after increment needed a pigeonhole argument.

Attempt 2 did something we had not predicted. Rather than guessing at lemma names, the implementer grepped the local `vstd` source for relevant helpers: `lemma_len_subset`, `lemma_int_range`, `axiom_set_insert_len`. It noticed the type mismatch between `Set<NodeId>` and `Set<int>` and wrote a new NodeId-analogue helper `lemma_range_nodeid_len` by structural recursion on `u32`. Final: `8 verified, 0 errors`. The agent could have left `assert(count <= n)` in place, weakened the invariant, or added an `assume` (and been caught by the hook). It wrote a recursive cardinality lemma instead.

The reviewer added a cross-exercise observation: the implementer leans heavily on `=~=` extensional equality and `choose` witnesses, a pattern that recurred in `bounded_log`. That observation went into the playbook.

### The extension tracks

After the three calibration exercises, the methodology carried into four extension tracks.

**BFT primitives** (`quorum_cert`, `ft_midpoint`, `marzullo`). The quorum certificate verified in six narrow iterations and landed `12 verified, 0 errors`. Three patterns from this run went into the playbook: pigeonhole-via-contradiction (wrap the negation in `if !(exists ...) { ... assert(false); }` and close with `lemma_len_subset`), the `lemma_fundamental_div_mod` trick for threshold arithmetic, and a usage note on the finiteness-hypothesis direction of `lemma_len_subset`. The two sensor-fusion exercises live in the [sensor-fusion post](https://ranjithkannan.com/2026/05/17/verified-byzantine-tolerant-sensor-fusion/); `marzullo` was the second operator-intervention case (the first frozen spec omitted the Helly-1D precondition, the agent surfaced it via constructive counterexample, the operator re-froze, the loop converged in one attempt).

**Multi-module Verus** (`cross_module_counter`, `counter_multifile`, `counter_producer`). Three exercises that stretched the harness from single-file to multi-module and multi-file layouts. The pre-commit hook's path whitelist, the spec-preservation step's exercise-name derivation, and the orchestrator's edit-scope variable all had to learn the multi-file shape. All three verified first-attempt; the architect's design pre-named the load-bearing loop invariants in each case.

**Composition of BFT primitives** (`sensor_poll`, `sensor_poll_signed`, `sensor_poll_honest`). Three exercises composing `quorum_cert`'s distinct-and-signed structural check with `marzullo`'s interval fusion, at increasing levels of trust-boundary integration. The third strengthened the postcondition with an honest-voter guarantee and doubled as a deliberate discovery test. Full discussion, including the honest-scope items about composition, lives in the [sensor-fusion post](https://ranjithkannan.com/2026/05/17/verified-byzantine-tolerant-sensor-fusion/).

### Discovery and invention tests

A **discovery test** gives the architect an obligation and informal mathematical content but deliberately omits the supporting lemmas. An **invention test** gives the architect a spec for a proof family the playbook does not document at all.

Two discovery tests ran: `sensor_poll_honest` (introduced `lemma_honest_supporter_exists` using inclusion-exclusion via `lemma_set_intersect_union_lens`, recognising the family from `ft_midpoint`'s playbook entry) and `counter_filler` (produced a 4-conjunct target-bounded invariant in place of the 6-conjunct counter-bounded form the playbook documented). Both re-audited under the hardened whitelist with their prior playbook summaries stripped from `AGENTS.md`; both verified in one attempt, structurally identical to the originals.

The invention-test arc produced the operator-territory boundary. The first invention test, `vec_swap` (a `Vec<T>::swap`-style exercise with a multiset-preservation postcondition the playbook did not anchor), passed `verus` in one attempt. The implementer's iter-1 commit message read "port witness proof", and its self-authored playbook entry recorded:

> The architect-owned witness file was visible to the implementer and gave the proof skeleton verbatim. For pure invention measurement, future exercises should hide the witness.

The agent flagged its own corruption in its own logs. We marked `vec_swap` INVALIDATED and extended `DISALLOWED_TOOLS` with the witness-deny patterns. An empirical probe confirms each of six attack vectors gets blocked, with the agent's own verdict: "I was unable to read any of the witness file's contents through any of the attempted methods."

A second attempt (`vec_swap_v2`) was also INVALIDATED because the operator's `cp vec_swap.rs vec_swap_v2.rs` copied the agent's already-filled body as the scaffold. The third attempt (`swap_multiset`, same spec as `vec_swap` but a hand-typed scaffold under the hardened whitelist) verified in one attempt with a proof structurally different from the operator-authored witness.

### External-validity probe

Two batches against tasks we did not design from Microsoft's [VeruSAGE-Bench](https://github.com/microsoft/verus-proof-synthesis). The load-bearing result is batch 2: eight tasks attempted with deliberately neutral design notes (no tooling-family names, no lemma names, no proof-structure suggestions), six verified, two distinct blocking findings. *Honest scope first:* the evidence says the methodology travels at small-to-medium proof-fn and exec sizes (171 B to 3.7 KB); it does not yet say it travels at VeruSAGE-Bench's full scale (7-24 KB examples deliberately deferred). Batch 1 was a smaller historical probe (n=2) with operator-authored design notes that named the tooling family, so its evidence was about harness adaptation only.

Six tasks verified: five single-attempt, one two-attempt (IR `singleton_seq_to_set_is_singleton_set`). The two-attempt arc is the methodology-grade data point: attempt 1 tried the obvious `=~=` extensional equality, watched verus reject it, and recorded in its own notes that "`=~=` alone doesn't trigger the `to_set` axiom needed to expand `seq![x].to_set().contains(y)`"; attempt 2 closed via vstd's `lemma_push_to_set_commute`. The agent read its failure, named the missing axiom, found the vstd lemma family that supplied it.

Two tasks blocked, with distinct findings.

The first, `NR__definitions_u__lemma_maxphyaddr_facts`, was blocked by a real gap in our own pre-commit hook. The upstream task ships with `#[verifier(external_body)]` on an opaque-constant declaration. Our hook's cheat-token detector treated every line of the brand-new scaffold file as "added" and rejected the `external_body` marker as agent-introduced cheating. Both layers were fixed: the pre-commit hook now diffs against `spec-frozen-<exercise>` when the tag exists and treats the scaffold commit as a baseline event; the witness check counts cheat-token occurrences in the witness minus the count in the paired exercise file. The task ran cleanly under the fix and verified in one attempt. The blocked-then-fixed-then-verified arc is itself the empirical demonstration of the boundary refinement.

The second, `OS__array__impl4__init2none`, was blocked by an upstream-downstream Verus version mismatch. The task's `ensures` clauses use pre-`final(self)` syntax that the current Verus build rejects; the witness fails to verify under our Verus, so the agent loop never ran. This is exactly the class of finding the witness mechanism is designed to catch. Pinning Verus to VeruSAGE-Bench's authored version is queued.

n=10 across both batches is too small to set against AutoVerus or VeruSAGE aggregates (Verus-Bench: 150 tasks; VeruSAGE-Bench: 849). The next batch is a larger size tier (2-10 KB), from `AC` and the larger `ST` / `OS` files.

## What the loop got right

Nine claims we are willing to defend from this run.

**The no-spec-weakening rule held under pressure.** Both the mechanical hook and the semantic reviewer caught real violations. The hook caught a body-content modification on bounded_log; the reviewer caught a spec-shape modification the hook missed. Two layers that fail differently are the boundary.

**Per-attempt commits made the methodology auditable.** Every implementer iteration is its own commit, with verifier output under `logs/<ex>/raw/` and reviewer audits citing specific HEAD lines. Git was sufficient.

**The role split bought what it advertised.** The architect produced substantive design notes; the implementers self-diagnosed their own failures rather than thrashing; the reviewer contributed cross-exercise pattern observations. A single Opus call could have done all three jobs. Whether its review of its own work would have been as honest as a separate role's, we doubt.

**Per-iteration scoping plus an architect sub-task list kept iterations narrow.** Each implementer call is directed at the smallest unfinished sub-task from the architect's design or at the specific failing function from the latest verifier output. The orchestrator iterates; the implementer does one thing per call.

**Pre-spec witness verification catches operator-time spec bugs.** Before tagging `spec-frozen-<name>`, the operator runs `ralph/check-spec.sh` against a reference implementation. A verus-passing witness means the spec admits a model. Two calibration exercises (`bounded_log`, `marzullo`) burned agent cycles on spec bugs this check would have caught; none of the post-witness-check exercises did. An empirical negative test confirms verus rejects a marzullo witness with the Helly-1D precondition stripped.

**Deliberate discovery tests passed, audit-confirmed.** `sensor_poll_honest` and `counter_filler` verified in one attempt each, then survived a re-audit under the hardened whitelist with their prior playbook summaries stripped from `AGENTS.md`. Two data points on two distinct proof families.

**Witness-access hardening turned the witness into a structural boundary, with an empirical probe to verify it.** The corrective arc from `vec_swap` produced the operator-territory boundary. The first clean invention test under the hardened whitelist (`swap_multiset`) produced a proof structurally different from the operator-authored witness.

**The methodology travels on tasks we did not design, at small-to-medium sizes.** *Caveat first:* the claim is bounded by size. Across two batches against VeruSAGE-Bench tasks (171 B to 3.7 KB), eight of ten attempts verified, and six of those eight passed under deliberately neutral design notes. It does not yet travel at VeruSAGE-Bench's full scale (7-24 KB examples deliberately deferred).

**The cheat-token boundary is now scaffold-aware.** Two surgical fixes: `scripts/git-hooks/pre-commit` diffs against `spec-frozen-<exercise>` and skips cheat-token detection on the scaffold commit; `ralph/check-spec.sh` counts cheat-token occurrences in the witness minus the count in the paired exercise file. The blocked-then-fixed-then-verified arc on `NR__definitions_u__lemma_maxphyaddr_facts` is the empirical demonstration.

## What the loop got wrong

Four honest shortcomings worth flagging.

**The hook's spec-preservation check had a known gap.** It used to look for lines whose first token was `requires` or `ensures`, missing the body content underneath. The bounded_log REJECT was caught only by the reviewer. The hook now walks the frozen file by indentation and extracts the complete clause body.

**The orchestrator used to treat every non-zero claude exit code as verus-failed.** A rate-limited response, a budget cap firing, a network blip all produced the same exit code. The first `quorum_count` run burned 27 rate-limited iterations before its outer ceiling fired. The orchestrator now classifies failures by grepping the iteration log against known signatures.

**The original implementer whitelist allowed witness reads.** The `vec_swap` invention test made this concrete: the agent's iter-1 commit was titled "port witness proof", and its own playbook entry recorded the gap honestly. The fix is the operator-territory boundary described above; the `vec_swap` / `vec_swap_v2` rows stay INVALIDATED as evidence.

**Fourteen exercises plus ten external-validity tasks is not a benchmark.** The vericoding paper has 12,504 specifications. We have twenty-four. The generalisations we are comfortable drawing are at the level of "this failure mode exists" or "this rule survived this pressure," not "success rate equals X percent."

## Where this fits

Three existing bodies of work touch pieces of this, but not the whole intersection.

The **Verus vericoding benchmark** (Schubert et al., 2025) measures LLM success rates on isolated single-function Verus tasks. One model, one shot, one function, no role split, no autonomous outer loop, no treatment of cheating. 44% first-try is the current floor for raw single-shot Verus. This experiment is about what surrounding scaffolding gets a real codebase past it.

**Microsoft Research's AutoVerus and VeruSAGE** ([microsoft/verus-proof-synthesis](https://github.com/microsoft/verus-proof-synthesis)) are doing pieces of the same problem from a different direction. AutoVerus (OOPSLA 2025, [arXiv:2409.13082](https://arxiv.org/abs/2409.13082)) targets small algorithmic code with a three-phase pipeline. VeruSAGE ([arXiv:2512.18436](https://arxiv.org/abs/2512.18436)) targets large system projects and ships VeruSAGE-Bench. Their orchestration is a single-agent pipeline rather than a role split, and they do not appear to ship an explicit no-cheating boundary.

**Huntley's [Ralph pattern](https://ghuntley.com/ralph/)** describes the outer-loop shape: fresh context per iteration, file-based memory, agent commits per attempt. It applies to unsafe-code domains where the feedback signal is the test suite. No verifier, no no-cheating rule, no separate audit role. The contribution here is keeping the loop shape and changing the feedback signal to a verifier, plus adding the three boundaries and the audit role that the harder signal requires.

**Karpathy's autoresearch demos** popularised the overnight-experiment framing with a scalar metric and a single agent that proposes, implements, and evaluates in the same context. Here the metric is binary, the proposer/implementer/auditor are three separate roles, and the operator's reference implementation is structurally separated from the agent's working set.

After the Microsoft work in particular, the claim worth defending is narrower than "no prior public work in this intersection." It is: a separate audit role running on a different model, a mechanical commit-time enforcement layer running alongside the LLM audit, per-attempt commits as the unit of evaluation, and operator-authored witness files used both for pre-spec verification and as a structural boundary the agent cannot reach.

A second contribution worth naming: the operator-intervention cases on `bounded_log` and `marzullo`. Most published vericoding results either succeed silently or fail silently. The loop's behaviour on both (the agent refused to cheat, articulated the constraint, named the empowered role, and stopped) is the kind of structured-failure output a trustworthy methodology should produce. The pre-spec witness check has since removed that class of operator-time mistakes.

Two next experiments. A verified Byzantine agreement primitive plus hardware-deployed sensor-fusion under live fault injection (deferred). And five to ten more VeruSAGE-Bench tasks at larger sizes with neutral design notes. Negative results from that run are more informative than positive ones; they tell us where the methodology stops travelling.

