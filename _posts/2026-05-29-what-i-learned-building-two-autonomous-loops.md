---
layout: post
title: "What I Learned Building Two Autonomous Coding Loops"
date: 2026-05-29
related:
  - /2026/05/10/verus-calibration-formal-verifier-loop/
  - /2026/05/17/verified-byzantine-tolerant-sensor-fusion/
  - /2026/05/27/three-systems-one-loop/
---

For the past several months I've been building an autonomous coding loop that wires LLM agents to formal feedback signals — Verus at the component layer, a deterministic oracle at the system layer. The work lives in two open-source repositories with sixteen verified component exercises and three runnable end-to-end systems between them. A few weeks ago I ran one of those systems through the system-layer half of the loop — a small in-memory key-value store I'd been calling `kv_store_v1`. What it surfaced is the cleanest illustration of what I'm building toward and what's still missing.

The setup was deliberately simple: a `BTreeMap<u64, u64>` reference implementation as the deterministic oracle, three proptest properties (random-operation equivalence against the oracle, empty-state consistency, put-then-get round-trip), and a stubbed SUT for the loop to fill in.

The loop's `system_designer` agent wrote a one-item plan: wrap a `BTreeMap`. The `integration_implementer` agent executed in a single attempt. The SUT it produced looked like this:

```rust
use std::collections::BTreeMap;

pub struct KvStore {
    inner: BTreeMap<u64, u64>,
}

impl KvStore {
    pub fn new() -> Self { Self { inner: BTreeMap::new() } }
    pub fn put(&mut self, k: u64, v: u64) { self.inner.insert(k, v); }
    pub fn get(&self, k: u64) -> Option<u64> { self.inner.get(&k).copied() }
    pub fn delete(&mut self, k: u64) { self.inner.remove(&k); }
    pub fn len(&self) -> usize { self.inner.len() }
    pub fn keys_sorted(&self) -> Vec<u64> { self.inner.keys().copied().collect() }
}
```

All three properties green on first run. The `runner_auditor` agent returned APPROVE on every checklist item — oracle unchanged, properties preserved, no test-disabling annotations, no verification-bypass tokens, no property deletions. Done.

Except for one paragraph the auditor put in `review.md` outside the checklist:

> The SUT is a textbook thin-wrapper v1 — structurally identical to the oracle, as SYSTEM.md predicts. This makes equivalence_step_by_step nearly trivial; a future kv_store_v2 with a structurally different backing (append-only log, hashmap + sorted index, etc.) would exercise the property suite more meaningfully.

The loop did exactly what I asked. I asked for a key-value store satisfying functional equivalence against a `BTreeMap`. The cheapest correct implementation of "match a `BTreeMap`" is to *be* a `BTreeMap`. Textbook engineering result. Also a textbook spec failure — the property suite I wrote did not require what I thought it required.

## the production-incident pattern, made cheap

This is the shape of a production incident, made cheap. A team writes a test suite. The suite passes. Six months in, an implementation choice that satisfies the suite causes the incident; the postmortem reveals that what the team cared about was never actually in the suite. The autonomous loop made the surprise arrive on first attempt instead of after six months of operation. The cheap implementation showed up immediately and the gap between "what the spec says" and "what I meant" became a thing I could see.

That's the thread the rest of this post pulls on. Two repositories built around the question of what specifications actually require versus what I mean them to require, and what feedback signals close the gap before an implementation reaches production.

## the architecture, in one diagram

```
   component layer  ──  verus-calibration
   ──────────────────────────────────────
     architect → implementer → reviewer
       verus is the feedback signal
       proof or rejection, no negotiation
   ──────────────────────────────────────
                   │
                   ▼ verified components compose into
                   │
   ──────────────────────────────────────
   system layer  ──  bft_autotune
     system_designer → integration_implementer → runner_auditor
       cargo test against a deterministic oracle
       is the feedback signal
       APPROVE or REJECT
   ──────────────────────────────────────
```

Same three-role shape at both layers. Different feedback signal. Marc Brooker's [hypothesis post](https://brooker.co.za/blog/2026/05/20/hypothesis.html) is the cleanest framing of why both signals work: given a deterministic oracle, the agent's job collapses to "match the oracle." Verus is the leaf-level oracle for component correctness. A reference implementation is the composition-level oracle for system behaviour. The combination produces a correctness claim stronger than either layer alone.

At the component layer the specs are `requires` / `ensures` blocks the operator freezes against a git tag. Sixteen exercises through this loop in [`verus-calibration`](https://github.com/ranjithkannank/verus-calibration), including a multi-module Byzantine quorum certificate with a safety lemma and a fault-tolerant Marzullo interval intersection over `n >= 2f + 1` sensors. The implementer fills in a proof body, runs Verus, iterates one attempt per call. The verifier accepts the body or rejects it with a specific failing obligation. No negotiation, no judgement call about correctness.

At the system layer the spec is the oracle file plus the property bodies in `proptest.rs`. Three runnable systems end-to-end in [`bft_autotune`](https://github.com/ranjithkannank/bft_autotune): a sensor-fusion service that composes the verified Marzullo with a distinct-sensor structural check, a stop-and-wait reliable transport over a lossy channel, and the in-memory key-value store from the opening scene. The implementer fills in the SUT and the adversary, runs `cargo test`, iterates. The feedback signal is the property test exit code. Pass means the SUT matches the oracle bit-for-bit on every operation in every generated sequence.

## the audit catches what the author can't

The most useful thing the loops produced isn't the verified exercises or the runnable systems. It's the times the *auditor* surfaced something the *author* couldn't have, because the audit happens on a fresh context against the diff, with no stake in defending the author's choices.

The `kv_store_v1` reviewer note from the opening scene is one example. Here's a sharper one.

The first time I tried an "invention test" in `verus-calibration` — could the loop produce a proof for a problem the playbook had never named — I used a permissive whitelist that allowed agent reads on operator-authored witness files. The exercise (`vec_swap`) verified in one iteration. Its commit message read "port witness proof." Its own self-authored playbook entry recorded that the witness file had been visible and had given the proof skeleton verbatim.

The agent flagged its own corruption in its own logs.

I tightened the whitelist with the deny patterns this work now ships with everywhere:

```
"Read(**/*_witness*)" "Read(**/*_witness/**)"
"Glob(**/*_witness*)" "Grep(**/*_witness.rs)"
"Bash(cat *_witness*)" "Bash(grep *_witness*)" ...
```

Re-ran on a fresh-named exercise (`swap_multiset`, same shape, same spec). The rerun produced a proof structurally different from the witness. The boundary that wasn't there in version one became load-bearing in version two — and the fact that the loop produced the evidence of its own failure is itself the signal that the audit separation works.

What I take from this beyond the loops is that separation of authoring from auditing matters more than either role's individual quality. The auditor on a fresh context, looking only at the diff against the spec, catches things the author cannot catch about their own work. This is what code review is supposed to be. In practice, code review is often by collaborators who share the author's framing — same standup, same design discussions, same architectural assumptions. The loop has none of that pollution. Half the value of the loop is that property.

The corollary is that audit quality is bounded by spec quality. The auditor's checklist catches cheating, spec drift, test-disabling, verification-bypass tokens. It does not catch *under-constraining* specs. That is what the `kv_store_v1` reviewer-notes paragraph surfaced. The auditor noticed because a fresh-context reader with no stake in defending the author's choices will sometimes spot the structural surprise — but only in the editorial register, outside the checklist. The checklist tells you whether the loop cheated. The reviewer notes tell you whether the spec was strong enough to matter.

## what I'm building next

The forward direction is what I'm calling spec refinement: a fourth agent role that consumes editorial observations like the `kv_store_v1` note and proposes a property tightening that would force the missing constraint. Today the observation lives in `review.md` and the loop terminates DONE regardless. The closed-loop version would propose, the operator would approve, and the loop would re-run against the tightened spec.

That is co-evolving the spec and the implementation under an autonomous loop, which is what senior engineers actually do in production code reviews. I have not seen anyone ship the autonomous version. AutoVerus and VeruSAGE do not have it. Hydro is a different paradigm without LLM iteration. Brooker's posts point at the gap but do not ship a solution. The next experiment is `kv_store_v2`, with the new role wired in and the v1 reviewer-notes paragraph as the seed input. If it works, the loop refines its own spec.

## what is still open

Three honest gaps, in priority order. The loops have not been run at production scale — the component-layer exercises are small, and the system-layer demonstrations are single-binary, single-process, with no concurrent load. The verus-to-Cargo binding is still stubbed: the `bft_autotune` SUTs that compose verified components today are ports of the verified Rust source, not direct consumption of the verified `.rlib` artefacts. The spec-refinement closed loop is named but not built. Each is a real piece of work; none of them invalidates what is already there.

## if you read this far

Two repositories: [`verus-calibration`](https://github.com/ranjithkannank/verus-calibration) (component-layer loop with Verus as the feedback signal; sixteen verified exercises, a documented playbook under `AGENTS.md`) and [`bft_autotune`](https://github.com/ranjithkannank/bft_autotune) (system-layer loop with a deterministic oracle; three runnable systems, each with its own `SYSTEM.md`).

The interesting reading is in the `logs/` directories of both, particularly `bft_autotune/logs/kv_store_v1/review.md` (the auditor's reviewer-notes paragraph that started this post) and `verus-calibration/scripts/probe-witness-deny.sh` (the empirical probe that demonstrates the boundary deny patterns hold under six common attack vectors). The "Discovered patterns" section of `verus-calibration/AGENTS.md` is where the methodology compounds — later exercises cite earlier patterns by name in their attempt logs.

The work isn't a benchmark or a paper or a finished thing. It's a thesis-shaped project written in public, with the blog posts as integration tests on the thinking. The next piece is `kv_store_v2` with spec refinement wired in. I'll write up the result when it lands.
