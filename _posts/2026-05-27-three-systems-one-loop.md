---
layout: post
title: "Three Systems, One Loop"
date: 2026-05-27
related:
  - /2026/05/10/verus-calibration-formal-verifier-loop/
  - /2026/05/17/verified-byzantine-tolerant-sensor-fusion/
---

The [previous methodology post](https://ranjithkannan.com/2026/05/10/verus-calibration-formal-verifier-loop/) wired a formal verifier into an autonomous coding loop at the component layer. The [next one](https://ranjithkannan.com/2026/05/17/verified-byzantine-tolerant-sensor-fusion/) showed that loop producing verified Byzantine-fault-tolerant primitives. This post is what happens when we move one layer up — a different autonomous loop, a different feedback signal — and ask it to compose those primitives into runnable systems. Three runs, three structurally different system shapes, three different lessons.

The conceptual anchor is Marc Brooker's [hypothesis post](https://brooker.co.za/blog/2026/05/20/hypothesis.html). Brooker states one claim in three forms. The weak form: "any coding task for which a complete specification is available will become trivial." The strong form (deterministic oracle): "any coding task for which a deterministic oracle is available will become trivial." The strongest form (non-adversarial oracle): "any coding task for which a non-adversarial (*pythic?*) oracle exists will become trivial." Brooker's progression names what makes automation cheap. The strong form is the one that travels furthest in practice — complete specs are hard to write; deterministic oracles are cheap to evaluate and hard to argue with.

We apply that at the system layer where component-level proof is unavailable. Verus is the leaf-level oracle for component correctness in `verus-calibration`. An explicit single-node faultless reference is the composition-level oracle for system behaviour in `bft_autotune`, the repo this post is about. Without an oracle, "the system works" becomes a vibe check; with one, it's a property test the agent's output must satisfy bit-for-bit.

This post's claim is the aggregate: the system-layer loop works on three structurally different systems. None of the three alone would be enough. **`sensor_poll_v2`** (sensor fusion with agreement, algorithmic fault tolerance over `n >= 2f + 1` reports) converged in one implementer attempt, and the auditor's review identified two points where the loop's output was sharper than the operator's spec required. **`tcp_v1`** (stop-and-wait reliable transport over a lossy channel, state machine plus adversary) converged in five attempts, and along the way revealed that the system_designer's sub-task list is a suggested decomposition rather than a contract. **`kv_store_v1`** (in-memory key-value store, storage with state-machine semantics) converged in one attempt, and the auditor's notes flagged that the spec under-constrained the SUT shape — pointing at a methodology direction that becomes the next post's headline.

Honest scope above the fold:

- **n=3 systems, three different shapes.** Sensor fusion, network protocol, key-value store. Three is meaningfully better than one but is still a small sample.
- **The verus binding is stubbed across all three.** `sensor_poll_v2`'s SUT is a plain-Rust port of the verified `marzullo`, not the verified `.rlib`. `tcp_v1` and `kv_store_v1` don't consume verified components at all. Real `.rlib` consumption is a subsequent iteration's first concern.
- **Multi-instance is in-process channels.** For `sensor_poll_v2` and `tcp_v1`, the "network" is in-process channel routing controlled by the fault adversary. `kv_store_v1` is single-threaded with no network at all. Real network is later. Real hardware is later still.
- **`kv_store_v1` picked the cheapest correct implementation.** The system_designer chose a `BTreeMap`-wrapper because the oracle is also `BTreeMap`-backed, which makes the equivalence property close to trivial. This is not a failure of the loop. It is the loop revealing that the property suite under-constrains the SUT shape. The fix sits on the spec side and is the subject of the next post.

Everything described here lives at <https://github.com/ranjithkannank/bft_autotune>. The per-section links point at exact files.

## The system layer's feedback signal

Brooker's claim has a specific shape. When an agent producing code can compare its output to a deterministic reference bit-for-bit, the agent's job collapses to "match the oracle." There is no judgement call about correctness; the comparison is mechanical. The verifier-versus-tests distinction at the component layer is the same distinction made one layer up. A proof is a deterministic verdict, and so is "outputs match the oracle on the input set."

The system-layer oracle in `bft_autotune` is a single-node, faultless reference implementation: trivially correct, cheap to evaluate, written by the operator before the loop runs. For sensor fusion the oracle is a 20-line interval intersection. For stop-and-wait transport it is a perfect-channel `OracleSender` / `OracleReceiver` pair that records every `send(msg)` and returns the messages verbatim in order. For the key-value store it is a `BTreeMap<u64, u64>` wrapper. The SUT must match the oracle on faultless inputs and satisfy verified-component invariants on adversarially-perturbed inputs.

This is Brooker's strong form applied at the system layer. The non-adversarial form may travel further on harder problems; for the three runs here, the deterministic form is achievable.

## The two-tier architecture

The system-layer loop has the same shape as the component-layer loop, with a different feedback signal:

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

Same three-role structure, fresh context per call, role-scoped tool whitelists, per-attempt commits. The framework scaffold (`AGENTS.md`, the three role definitions, the orchestrator, the first regen target) landed in commit [`c4b37f1`](https://github.com/ranjithkannank/bft_autotune/commit/c4b37f1).

## The three boundaries, at the system layer

Three boundaries with the same shape as the component-layer loop, scoped to the new layer's artifacts.

**Content boundary.** The `runner_auditor`'s five-point checklist audits the diff against a `system-frozen-<system>` git tag. It confirms `src/oracle.rs` unchanged, existing property bodies in `src/proptest.rs` preserved verbatim, no test-disabling annotations (`#[ignore]`, `#[should_panic]` on existing tests, `catch_unwind` wrappers), no verification-bypass tokens (`assume!`, `external_body`, `assume_specification`, `unimplemented!()` in reachable positions), no property deletions.

**Capability boundary.** Each `claude -p` call passes a role-scoped `--allowedTools` list and a universal `--disallowedTools` deny set. The `integration_implementer` can edit and `cargo test`; the `system_designer` cannot run cargo; the `runner_auditor` cannot edit code at all (its job is git-diff-based).

**Operator-territory boundary.** `src/oracle.rs` and the existing property bodies in `src/proptest.rs` are operator territory. The orchestrator's `DISALLOWED_TOOLS` denies any agent `Edit(**/oracle.rs)` or `Write(**/oracle.rs)`; the property-body constraint is enforced by the auditor at audit time. `SYSTEM.md` is dynamic — written by the `system_designer` in THINK, revised on escalation — and is not part of the frozen spec.

Honest: none of this is a sandbox. A motivated adversarial agent could find paths around it. As with the component-layer loop, the boundaries' job is to make it impossible for an honest worker to cheat accidentally.

## The three roles

Three [Claude Code subagents](https://github.com/ranjithkannank/bft_autotune/tree/main/.claude/agents), all on Opus 4.7.

- **[system_designer](https://github.com/ranjithkannank/bft_autotune/blob/main/.claude/agents/system_designer.md).** Reads the operator-authored `oracle.rs`, `proptest.rs`, and `Cargo.toml`. Writes `SYSTEM.md`: architecture, what the SUT must do, what "deploy" means, open questions, and a numbered sub-task list ordered easiest to hardest. Does not see cargo output on the first pass.
- **[integration_implementer](https://github.com/ranjithkannank/bft_autotune/blob/main/.claude/agents/integration_implementer.md).** One new attempt per call: edit `sut.rs` (and `faults.rs` if needed), run `cargo test --quiet`, log the result, commit. Escalates after three consecutive failures on the same property.
- **[runner_auditor](https://github.com/ranjithkannank/bft_autotune/blob/main/.claude/agents/runner_auditor.md).** After cargo test exits 0, audits the diff against `system-frozen-<system>` using the five-point checklist. Does not check correctness; the proptest sweep already did.

## First run — sensor_poll_v2

The first system was a regeneration target. `sensor_poll_v1` was operator-authored — a distinct-sensor structural check composed with a marzullo-style interval intersection, validated against a single-node faultless oracle under three proptest properties (faultless equivalence on N=3, marzullo invariant on N=4 with f=1, no-panic under arbitrary input). `sensor_poll_v2` shares the operator-frozen `oracle.rs` and `proptest.rs` byte-for-byte, with the SUT bodies blanked. The loop's job: fill in `sut.rs`, `faults.rs`, and `main.rs` from scratch.

Three outer iterations from spec to DONE. The `system_designer` (commit [`7112719`](https://github.com/ranjithkannank/bft_autotune/commit/7112719)) wrote `SYSTEM.md` from `oracle.rs` and the property bodies. The `integration_implementer`'s single attempt (commit [`62052d8`](https://github.com/ranjithkannank/bft_autotune/commit/62052d8)) covered all seven `system_designer` sub-tasks in one pass and got `test result: ok. 3 passed; 0 failed`. The `runner_auditor` (commit [`54d3553`](https://github.com/ranjithkannank/bft_autotune/commit/54d3553)) returned APPROVE on all five checklist points, then APPROVED → DONE in commit [`4719474`](https://github.com/ranjithkannank/bft_autotune/commit/4719474).

The interesting part is the reviewer-notes section of [`review.md`](https://github.com/ranjithkannank/bft_autotune/blob/main/logs/sensor_poll_v2/review.md):

> The author explicitly justifies the comparison-only choice ("no arithmetic on bounds, so arbitrary i64 inputs cannot overflow") which is exactly the overflow concern flagged in SYSTEM.md's open-questions section. The `n < 2*f + 1` guard short-circuits before the reduction, so empty input at `f=1` returns `None` rather than silently producing `[i64::MIN, i64::MAX]` — a sharper choice than the spec strictly required.

Two design choices the audit calls out: comparison-only arithmetic in the `marzullo` intersect (no `hi - lo` subtraction, no overflow risk on arbitrary `i64` inputs) and a short-circuit on `n < 2f + 1` before the reduction. Both are sharper than the property bodies strictly demand. The properties pass on less careful implementations too; the agent picked the more careful path on the first attempt.

The operator-written v1 made the same design choices. What v2 added was the rationale: v1's `marzullo` doc comment says nothing about overflow safety, v2's says "Comparison-only — no arithmetic on bounds, so arbitrary i64 inputs cannot overflow." The agent wrote the reasoning down — in the code comment and in `attempts.md`. A separate auditor on a fresh context, running only against the diff, noticed and called the rationale out. Reproducible quality with stated reasoning is a useful methodology signal: not "the loop catches what the operator missed" but "the loop produces work indistinguishable from a careful operator's, with the design rationale documented in places the operator's hand-written reference did not bother to write it down."

## Harness change between sensor_poll_v2 and tcp_v1

Before the second system we made one harness change. The orchestrator now scopes each WORK iteration to exactly one sub-task from `SYSTEM.md`. After THINK, the `system_designer`'s sub-task list is parsed into `logs/<system>/subtasks.md` as a checklist; each WORK prompt names exactly one unchecked sub-task as the implementer's scope; the implementer marks it complete when (and only when) it is actually done. Multi-sub-task work in a single iteration is forbidden — fresh context per sub-task is the whole point.

The motivation was anticipated rather than discovered. `sensor_poll_v2` was small enough that the implementer could land all seven sub-tasks in a single pass. Anything stateful and adversary-driven would not be. With per-sub-task scoping, each iteration is bounded by the size of one sub-task, and the implementer's context window stops being a hidden constraint on system complexity. The change is itself a methodology contribution — the way to scale the system-layer loop to larger systems is to insist on one bite at a time. It landed in commit [`5c00a06`](https://github.com/ranjithkannank/bft_autotune/commit/5c00a06).

## Second run — tcp_v1

The second system tests a different shape: stateful Sender / Receiver pair, network protocol rather than algorithmic fault tolerance. Operator-authored spec: a perfect-channel oracle (`OracleSender` records every `send(msg)`, `OracleReceiver::delivered` returns those messages verbatim in order), a `WireAction` adversary in `faults.rs` (`Deliver`, `Drop`, `Duplicate`), and three properties in `proptest.rs` — faultless delivery matches the oracle, eventual delivery under drops, receiver dedups under duplication.

The `system_designer` produced a seven-item sub-task list ordered easiest to hardest: `faults::apply` dispatcher → trivial `main.rs` → sender/receiver state + constructors → faultless happy path → retransmission → receiver dedupe → full green and polish. The `integration_implementer` worked through it one sub-task at a time under the new harness. Five implementer attempts in total ([`f735167`](https://github.com/ranjithkannank/bft_autotune/commit/f735167), [`2736835`](https://github.com/ranjithkannank/bft_autotune/commit/2736835), [`841bec1`](https://github.com/ranjithkannank/bft_autotune/commit/841bec1), [`db3771c`](https://github.com/ranjithkannank/bft_autotune/commit/db3771c), [`eae8326`](https://github.com/ranjithkannank/bft_autotune/commit/eae8326)). Sub-tasks 1 through 4 took one attempt each. Sub-task 5 (retransmission via `Sender::on_timeout`) took two outer iterations — the same sub-task selected twice from the checklist, fresh context each time.

After sub-task 5's attempt landed, all three properties passed. The orchestrator transitioned to REVIEW. The `runner_auditor` returned APPROVE on the checklist ([`4a58f67`](https://github.com/ranjithkannank/bft_autotune/commit/4a58f67)), then DONE ([`f142ac4`](https://github.com/ranjithkannank/bft_autotune/commit/f142ac4)).

The interesting part is what happened to sub-tasks 6 and 7 — the planned receiver-dedup logic and the "full green and polish" pass. They were never explicitly worked. They were satisfied incidentally by the choices the implementer made in earlier sub-tasks. A duplicate `Data` packet whose `seq < expected_seq` falls through `Receiver::on_packet`'s `_ => Vec::new()` no-op branch: silently dropped, no re-deliver, no ACK. The sender already got the ACK from the first copy and moved on. The dedup property passes without explicit dedup handling, because the stop-and-wait + retransmission design already encodes the right behaviour at the `seq` level. The implementer's own attempts.md flagged this honestly:

> Sub-task 6 (receiver dedupe — Property 3) is already implicitly green because `Receiver::on_packet` returns `Vec::new()` on stale Data and proptest happens not to need an Ack on that branch yet. However, the sub-task explicitly requires returning an Ack on `seq < expected_seq` (so a sender retransmitting after a lost Ack can make progress). Even though all tests currently pass, sub-task 6 should still implement that behavior to match SYSTEM.md's contract — fresh-context iteration will handle it.

But the orchestrator's success condition is `cargo test --quiet` exit 0, not "all sub-tasks marked complete." The moment the properties green, the loop transitions to REVIEW regardless of how many sub-tasks remain on the checklist. The `system_designer`'s sub-task list is a *suggested* decomposition for the implementer's context-window budget, not a contract the loop must satisfy. tcp_v1 converged faster than its sub-task list predicted, because the protocol design naturally subsumes some of the planned work. That is a feature: the list provides scaffolding, the loop's success condition is the property test.

## Third run — kv_store_v1

The third system tests another shape: storage with state-machine semantics. Operator-authored spec: a `BTreeMap<u64, u64>` oracle exposing `put` / `get` / `delete` / `len` / `keys_sorted`, a SUT stub with the same interface, three proptest properties — step-by-step equivalence under a random op sequence (keys biased to a small range so `Get` and `Delete` actually hit existing entries), empty-state consistency, put-then-get round-trip. No `faults.rs` — single-threaded, no adversary in v1.

The `system_designer` wrote a two-item sub-task list. Sub-task 1: replace `KvStore`'s backing store with a `BTreeMap<u64, u64>` and implement all five methods as direct delegations. Sub-task 2: replace `main.rs`'s `unimplemented!()` with a trivial body. The `integration_implementer`'s attempt-1 (commit [`151cd29`](https://github.com/ranjithkannank/bft_autotune/commit/151cd29)) implemented sub-task 1 exactly as the system_designer specified. From `attempts.md`:

> Rewrote `src/sut.rs` so `KvStore` wraps a `std::collections::BTreeMap<u64, u64>`. `new` constructs an empty map; `put` calls `insert`; `get` calls `get` + `copied`; `delete` calls `remove`; `len` delegates; `keys_sorted` collects `inner.keys().copied()`.

All three properties green on first run. The `runner_auditor` returned APPROVE on the checklist ([`71edc2a`](https://github.com/ranjithkannank/bft_autotune/commit/71edc2a)), then DONE ([`e8fcd28`](https://github.com/ranjithkannank/bft_autotune/commit/e8fcd28)). Three outer iterations, one implementer attempt, all properties passing.

The auditor's checklist behaved as expected. The auditor's reviewer notes did something the checklist alone does not. Quoting [`review.md`](https://github.com/ranjithkannank/bft_autotune/blob/main/logs/kv_store_v1/review.md) verbatim:

> The SUT is a textbook thin-wrapper v1 — structurally identical to the oracle, as `SYSTEM.md` predicts. This makes `equivalence_step_by_step` nearly trivial; a future `kv_store_v2` with a structurally different backing (append-only log, hashmap + sorted index, etc.) would exercise the property suite more meaningfully. Sub-task 2 (replacing the `unimplemented!` in `main.rs`) remains undone according to `subtasks.md`/the diff; the orchestrator's one-sub-task-per-iteration rule explains this and it is not an audit concern (main is not on the cargo-test path).

The auditor's role widened, from "did the agent cheat" to "does the spec actually require what we thought it required." The system_designer picked a `BTreeMap`-wrapper because the oracle is also `BTreeMap`-backed and that satisfies `equivalence_step_by_step` trivially. The implementer executed. The audit passed. And the auditor explicitly noted that the property suite is under-constraining the SUT shape — that a structurally different backing would exercise the proptest harness more meaningfully than this one does.

In the current loop this observation is passive. The auditor writes it in `review.md` and the orchestrator transitions DONE regardless. Making the observation active — turning auditor notes into inputs the loop refines the spec against — is the direction the next post will develop. We are not developing it here.

## Where this fits

The trust-ladder progression from the rest of the series: plain tests, then mutation testing, then a separate auditor, then integration contracts, then a formal verifier wired into the component-layer loop. Each step closed a hole through which a wrong loop could still pass. This post adds a different layer rather than a tighter feedback signal at the same layer. The system layer has its own three boundaries, its own three roles, its own feedback signal (oracle + cargo test). The component layer continues to operate underneath it; the system-layer SUTs consume the verified components (currently via ported plain-Rust copies; real `.rlib` consumption is later).

The system layer is orthogonal to AutoVerus and VeruSAGE. Those systems work at the component layer — single-function and repository-scoped Verus tasks. The system layer is downstream of that work, not in competition with it. It consumes the verified components and adds a different kind of feedback signal where component-level proof is not available.

Three threads on what comes next. The first is the spec-refinement direction — the `kv_store_v1` auditor's notes are the empirical opening for a `spec_critic` role that proposes property tightenings when the auditor surfaces a spec gap, and a closed loop where the methodology refines its own spec. That is the next post, with `kv_store_v2` as the case study. The second is real consumption of `verus-calibration`'s outputs as Cargo dependencies rather than ported source; substantively engineering, strengthens the "verified components" claim of the methodology. The third is the long-arc aerospace direction the project carries from `verus-calibration`'s sensor-fusion track: in-process channels → multi-process → multi-board dissimilar redundant triads. Multi-week per step, off the post's path, worth naming as the target.

## Reproducing

```bash
git clone https://github.com/ranjithkannank/bft_autotune
cd bft_autotune

# Run each system's property test sweep:
cd systems/sensor_poll_v1 && cargo test --quiet && cd ../..
cd systems/sensor_poll_v2 && cargo test --quiet && cd ../..
cd systems/tcp_v1         && cargo test --quiet && cd ../..
cd systems/kv_store_v1    && cargo test --quiet && cd ../..

# Re-run the loop on any of the regen targets (operator-side
# plumbing — rewinding the system's src/ to its stubbed state and
# retagging — omitted here):
./orchestrator/run-system.sh sensor_poll_v2
./orchestrator/run-system.sh tcp_v1
./orchestrator/run-system.sh kv_store_v1
```

The four `cargo test` commands let a reader confirm all four systems work locally in under a minute. Per-attempt commits, the design notes, and the audit reports are committed under `logs/<system>/` and `systems/<system>/`. Models are `claude-opus-4-7` for all three roles; the choice lives in `orchestrator/run-system.sh`. Each `system-frozen-<system>` tag marks the spec baseline the auditor diffs against.
