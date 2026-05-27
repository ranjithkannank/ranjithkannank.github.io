---
layout: post
title: "An Oracle-Driven Loop at the System Layer"
date: 2026-05-25
related:
  - /2026/05/10/verus-calibration-formal-verifier-loop/
  - /2026/05/17/verified-byzantine-tolerant-sensor-fusion/
---

The [previous methodology post](https://ranjithkannan.com/2026/05/10/verus-calibration-formal-verifier-loop/) wired a formal verifier into an autonomous coding loop and ran fourteen exercises through it. That loop's feedback signal was verus: proof-based correctness at the component layer. This post is the second autonomous loop in the same series, with a different feedback signal at a different layer: a deterministic oracle, applied at the system above the components.

The architecture follows Marc Brooker's [hypothesis on agentic software development](https://brooker.co.za/blog/2026/05/20/hypothesis.html), which states one claim in three forms. The weak form: "any coding task for which a complete specification is available will become trivial." The strong form (deterministic oracle): "any coding task for which a deterministic oracle is available will become trivial." The strongest form (non-adversarial oracle): "any coding task for which a non-adversarial (*pythic?*) oracle exists will become trivial." Brooker's progression names what makes automation cheap. The strong form is the one that travels furthest in practice, because complete specifications are hard to write and deterministic oracles are cheap to evaluate.

We apply that at the system layer where component-level proof is not available. Verus is the leaf-level oracle for component correctness in `verus-calibration`. An explicit single-node faultless reference implementation is the composition-level oracle for system behaviour in `bft_autotune`, the new repo this post is about. The two layers compose:

```
   component layer (verus-calibration)
   ───────────────────────────────────────────────────
     architect ──► implementer ──► reviewer
            verus is the feedback signal
            proof or rejection
   ───────────────────────────────────────────────────
                       │
                       ▼  verified components feed into
                       │
   ───────────────────────────────────────────────────
   system layer (bft_autotune)
     system_designer ──► integration_implementer ──► runner_auditor
            cargo test against a deterministic oracle
            is the feedback signal
            APPROVE or REJECT
   ───────────────────────────────────────────────────
```

Same shape (three roles, fresh context per call, role-scoped tool whitelists, per-attempt commits); different feedback signal. The trust-ladder framing from the prior posts extends here. Each layer's feedback signal is what makes the loop honest at that layer.

Honest scope above the fold:

- **n=1 system regenerated.** One data point, not a sweep. The next system on the track is a different shape (a RocksDB-like KV store under fault injection), and that is the test of whether the methodology travels beyond sensor-fusion.
- **The verus-to-cargo binding is stubbed.** The SUT in `sensor_poll_v2` is a plain-Rust port of the verified `marzullo`, not the verified `.rlib` itself. Real consumption is the next iteration of the harness.
- **Multi-instance is in-process channels.** Real network is later. Real hardware is later still.
- **The oracle and the property bodies are operator territory.** The loop did not invent the spec — the operator wrote `oracle.rs` and the existing properties, and the loop respects them. The autonomy claim is "given an oracle and a property set, the loop produces a passing SUT," not "the loop invents the spec."

Everything described here lives at <https://github.com/ranjithkannank/bft_autotune>. The per-section links point at exact files.

## The system layer's feedback signal

Brooker's claim has a specific shape. When an agent producing code can compare its output to a deterministic reference bit-for-bit, the agent's job collapses to "match the oracle." There is no judgement call about correctness; the comparison is mechanical. The verifier-versus-tests distinction at the component layer is the same distinction made one layer up. A proof is a deterministic verdict, and so is "outputs match the oracle on the input set."

For `sensor_poll_v2` the oracle is a single-node, faultless reference. Given N sensor readings without Byzantine input, it produces the value the multi-node BFT system should produce on faultless operation: the intersection of all reported intervals, with `lo = max(r.lo)` and `hi = min(r.hi)`, returning `Some` iff `lo <= hi`. Deterministic, cheap, trusted. The implementer's job is to produce a SUT that matches the oracle bit-for-bit on faultless inputs, and that satisfies the verified-component invariants from `verus-calibration` on Byzantine inputs.

This is Brooker's strong form applied at the system layer. The non-adversarial form may travel further on harder problems, but for sensor-fusion-with-agreement the deterministic form is achievable: the faultless oracle is a 20-line Rust function, and it is the entire spec for the faultless-equivalence property.

## The three boundaries, at the system layer

The component-layer loop had three boundaries: a pre-commit hook on content, a tool whitelist on capability, and witness-file path scoping on operator territory. The system-layer loop has three boundaries with the same shape, scoped to the new layer's artifacts.

**Content boundary: the `system-frozen-<system>` tag.** The audit runs `git diff system-frozen-<system>..HEAD -- systems/<system>/` and verifies that two files are byte-identical to the frozen tag: `src/oracle.rs` (the spec at the system layer) and the existing property assertions in `src/proptest.rs`. New properties may be added; existing `prop_assert*` and `prop_assume!` lines may not change. Weakening a property assertion to make tests pass is the system-layer equivalent of weakening an `ensures` clause in verus.

**Capability boundary: a per-role tool whitelist.** Each `claude -p` invocation passes a role-scoped `--allowedTools` list and a universal `--disallowedTools` deny set. The deny set includes `WebFetch`, `WebSearch`, `Task` (no sub-agent fan-out), network and install patterns (`curl`, `wget`, `brew`, `pip`, `cargo install`), and the `*--no-verify*` patterns that would bypass the pre-commit hook. The `system_designer` role does not get `Edit` for `src/`; the `integration_implementer` does. The `runner_auditor` gets neither, since its job is git-diff-based.

**Operator-territory boundary: `oracle.rs` deny patterns.** The capability boundary adds `Edit(**/oracle.rs)` and `Write(**/oracle.rs)` to the deny set explicitly. The oracle is the spec; the agent cannot rewrite it under any role. The property bodies in `proptest.rs` are protected by the audit rather than the deny patterns, because `proptest.rs` is a single file the implementer may need to extend with new properties. The constraint "existing property assertions byte-identical" is enforced by the `runner_auditor` at audit time, the same way the verus loop enforced spec preservation by reviewer audit.

To state what these are not. Not process isolation. Not a network namespace. An adversarial agent could find paths around them. The agents in this experiment are not adversarial. The boundaries' job is to make it impossible for an honest worker to cheat accidentally: to silence a failing property with `#[ignore]`, to wrap a panicking branch in `catch_unwind`, or to swap the oracle for a tautology because the loop was about to hit its cap.

## The three roles

Three [Claude Code subagents](https://github.com/ranjithkannank/bft_autotune/tree/main/.claude/agents), all on Opus 4.7.

- **[system_designer](https://github.com/ranjithkannank/bft_autotune/blob/main/.claude/agents/system_designer.md).** Reads the operator-authored `oracle.rs`, `proptest.rs`, and `Cargo.toml`. Writes `SYSTEM.md`: architecture, what the SUT must do, what "deploy" means here, open questions, and a numbered sub-task list ordered easiest to hardest. Does not see cargo output on the first pass.
- **[integration_implementer](https://github.com/ranjithkannank/bft_autotune/blob/main/.claude/agents/integration_implementer.md).** Makes exactly one new attempt per call: edit `sut.rs` (and `faults.rs` if the adversary needs to grow), run `cargo test --quiet`, log the result, commit. Stops at the per-system iteration cap, or earlier on escalation.
- **[runner_auditor](https://github.com/ranjithkannank/bft_autotune/blob/main/.claude/agents/runner_auditor.md).** After cargo test passes, audits the diff against `system-frozen-<system>` using a five-point checklist. Does not check correctness, since the proptest sweep already did. Returns APPROVE or REJECT.

The three roles are wired together by a [Ralph-style outer loop in bash](https://github.com/ranjithkannank/bft_autotune/blob/main/orchestrator/run-system.sh), the same shape as the verus loop. The state machine reads filesystem artifacts (`SYSTEM.md`, `attempts.md`, `status`, `review.md`, `done.flag`) and fires one `claude -p` per iteration with fresh context. The state transitions: `START → THINK → WORK → REVIEW → APPROVED`, with `THINK_REVISE` on implementer escalation and `WORK_AFTER_REJECT` on auditor rejection.

A single per-iteration claude invocation looks like this — the same shape as the verus loop:

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

`--no-session-persistence` keeps the loop honest: each iteration starts with empty context. Memory between iterations lives in files only — `SYSTEM.md`, the design's sub-task list, `attempts.md`, the per-attempt cargo output under `logs/<system>/raw/`, and git history. Each implementer attempt is its own commit, the same pattern as the verus loop.

The system-layer loop scaffold — `AGENTS.md`, the three role definitions, the orchestrator, and the `sensor_poll_v2` regeneration target — landed in commit [`c4b37f1`](https://github.com/ranjithkannank/bft_autotune/commit/c4b37f1).

## What happened — sensor_poll_v2

The first system this loop tried to implement was `sensor_poll_v2`. The spec is byte-identical to an operator-authored reference, [`sensor_poll_v1`](https://github.com/ranjithkannank/bft_autotune/tree/main/systems/sensor_poll_v1), which composes the verified `marzullo` interval-intersection primitive with a distinct-sensor structural check into a `poll(reports, f) -> Option<Interval>` function. v1 was hand-written and verified to pass three properties. v2 starts from the same `oracle.rs` and `proptest.rs` as v1, but with the SUT bodies blanked: `sut::poll`, `faults::apply`, and `main` all contain `unimplemented!()`. The loop's job is to fill them in from scratch.

The run took three outer iterations.

**Iteration 1 — THINK** (commit [`7112719`](https://github.com/ranjithkannank/bft_autotune/commit/7112719)). The system_designer agent read `oracle.rs`, `proptest.rs`, and `Cargo.toml`, and produced [`SYSTEM.md`](https://github.com/ranjithkannank/bft_autotune/blob/main/systems/sensor_poll_v2/SYSTEM.md): a 200-line architecture note covering the SUT contract derived from the oracle, what "deploy" means in v2 (single in-process invocation; the multi-instance framing is forward-looking), open questions including overflow on `i64` arithmetic, and a seven-step sub-task list ordered easiest to hardest.

**Iteration 2 — WORK** (commit [`62052d8`](https://github.com/ranjithkannank/bft_autotune/commit/62052d8)). The integration_implementer agent's attempt-1 covered all seven sub-tasks in one pass. Its [`attempts.md`](https://github.com/ranjithkannank/bft_autotune/blob/main/logs/sensor_poll_v2/attempts.md) entry is concise:

> **Approach:** Filled in `main.rs` with a trivial `println!`, implemented `faults::apply` as clone + replace-or-append per the v1 contract, implemented `sut::poll` as distinct-sensor structural check followed by a marzullo-style intersection (`max lo`, `min hi`) with a minimum-N guard `n >= 2f + 1`, comparison-only arithmetic (no `hi - lo`), and `None` on empty input.
>
> **Test output:**
> ```
> test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured;
> 0 filtered out; finished in 0.02s
> ```

All three properties (`faultless_equivalence_n3`, `marzullo_invariant_n4_f1`, `no_panic_under_arbitrary_input`) passed. The implementer set status to `tests_passed` and stopped. The outer loop fired the runner_auditor.

**Iteration 3 — REVIEW** (commit [`54d3553`](https://github.com/ranjithkannank/bft_autotune/commit/54d3553)). The runner_auditor returned APPROVE on all five checklist points. The justification cites the diff boundary directly: only four paths changed (`SYSTEM.md`, `src/faults.rs`, `src/main.rs`, `src/sut.rs`); `src/oracle.rs` and `src/proptest.rs` are byte-identical to the frozen tag.

**APPROVED → DONE** (commit [`4719474`](https://github.com/ranjithkannank/bft_autotune/commit/4719474)). The orchestrator wrote `done.flag` and exited 0.

Three outer iterations from spec to DONE. One implementer attempt. All three properties passing.

The interesting part is the reviewer-notes section of [`review.md`](https://github.com/ranjithkannank/bft_autotune/blob/main/logs/sensor_poll_v2/review.md):

> The SUT shape is small and clean: a 10-line distinct-id check plus a 20-line comparison-only marzullo intersect, composed in a 7-line `poll`. The author explicitly justifies the comparison-only choice ("no arithmetic on bounds, so arbitrary i64 inputs cannot overflow") which is exactly the overflow concern flagged in SYSTEM.md's open-questions section. The `n < 2*f + 1` guard short-circuits before the reduction, so empty input at `f=1` returns `None` rather than silently producing `[i64::MIN, i64::MAX]` — a sharper choice than the spec strictly required.

Two design choices the audit calls out: comparison-only arithmetic in the `marzullo` intersect (no `hi - lo` subtraction, so arbitrary `i64` inputs cannot overflow) and a short-circuit on `n < 2f + 1` before the reduction (empty input at `f=1` returns `None` rather than ever reaching the loop). Both are sharper than the property bodies strictly demand. The properties pass on less careful implementations too; the agent picked the more careful path on the first attempt.

The operator-written v1 happened to make the same design choices. The difference is what each implementation says about why. v1's `marzullo` doc comment describes the algorithm and points at the verified spec; v2's says "Comparison-only — no arithmetic on bounds, so arbitrary i64 inputs cannot overflow." The agent wrote the reasoning down, both in the code comment and in `attempts.md`. A separate auditor on a fresh context — running only against the diff, not against v1 — noticed and called the rationale out. That is the methodology signal worth landing. The loop produced production-quality design on its first attempt, with the design rationale documented in places the operator's hand-written reference did not bother to write it down. Reproducible quality with stated reasoning is more useful than a one-off improvement, because reproducibility is what the methodology claim turns on.

## What the loop did not do

Restating the honest-scope items as a bulleted block, so they cannot be skimmed past:

- **n=1.** One system regenerated end-to-end. Not a sweep, not a benchmark, not evidence the methodology generalises. The next system on the list is a different shape (a RocksDB-like KV store under fault injection), and that is the one that will test whether the loop travels beyond sensor-fusion.
- **The verus-to-cargo binding is stubbed.** The SUT in `sensor_poll_v2` is a plain-Rust port of the verified `marzullo`, not the verified `.rlib` itself. Three binding options are listed in [`sensor_poll_v1/SYSTEM.md`](https://github.com/ranjithkannank/bft_autotune/blob/main/systems/sensor_poll_v1/SYSTEM.md); none are committed yet. The v2 of *this work* (confusingly: not to be confused with `sensor_poll_v2`, which is v2 of the *system*) is real consumption of `verus-calibration`'s outputs.
- **Multi-instance is in-process channels.** The `sensor_poll_v2` run exercises the composition pattern at the level of one in-process `poll(reports, f)` call, not at the level of N nodes communicating over a real network. The "multi-instance / channel-routed network" framing in `sensor_poll_v1/SYSTEM.md` is forward-looking; the properties this system passes only exercise the in-memory `Vec` interface. Real network and the eventual hardware target are later.
- **The oracle and the property bodies are operator territory.** The loop did not invent the spec — the operator wrote `oracle.rs` and the existing properties, and the loop respects them. The autonomy claim is "given an oracle and a property set, the loop produces a passing SUT," not "the loop invents the spec." This is the system-layer analog of the verus loop's "operator authors the frozen spec" rule.

## Where this fits

The trust-ladder progression from the rest of the series: plain tests, mutation testing, a separate auditor, integration contracts, a formal verifier wired into the component-layer loop. Each step closed a hole through which a wrong loop could still pass. This post adds one more rung: a deterministic oracle wired into a composition-layer loop. The verifier closes holes at the leaves; the oracle closes holes at the composition.

The component-layer and the composition-layer are orthogonal. AutoVerus and VeruSAGE work at the component layer (single-function and repository-scoped Verus tasks); the system layer is downstream of that work, not in competition with it. It consumes the verified components and adds a different kind of feedback signal where component-level proof is not available.

The next concrete step on this track is a different system: a RocksDB-like key-value store with deterministic-oracle property testing under fault injection. Different shape from sensor-fusion (state, not stateless; multi-operation history, not single-call), which is exactly why it is the right next test of whether the methodology travels. After that, real verus-binding — Cargo path-dependency on a wrapped `verus-calibration`, then real `.rlib` consumption. After that, real network deployment.

## Reproducing

```bash
git clone https://github.com/ranjithkannank/bft_autotune
cd bft_autotune

# v1: operator-authored reference. Smoke test.
cd systems/sensor_poll_v1 && cargo test

# v2: autonomously regenerated by the loop. Same three properties.
cd ../sensor_poll_v2 && cargo test

# Re-run the loop end-to-end (rewind v2 to its stubbed state first):
cd ../..
./orchestrator/run-system.sh sensor_poll_v2
```

The first two commands let a reader confirm both systems work locally in two minutes. The third is the loop run. Per-attempt commits, the design note, and the audit report are committed under `logs/sensor_poll_v2/` and `systems/sensor_poll_v2/`. Models are `claude-opus-4-7` for all three roles; the choice lives in `orchestrator/run-system.sh`. The `system-frozen-sensor_poll_v2` tag marks the spec baseline the auditor diffs against.
