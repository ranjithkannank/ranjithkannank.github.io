---
layout: post
title: "A Verified Byzantine Quorum Certificate"
date: 2026-05-16
related:
  - /2026/04/10/multi-agent-tdd-loop/
  - /2026/04/19/pit-mutation-testing-ralph-loop/
  - /2026/04/23/final-validator-ralph-loop/
  - /2026/04/27/integration-test-contracts/
  - /2026/05/10/verus-calibration-formal-verifier-loop/
  - /2026/05/17/verified-byzantine-tolerant-sensor-fusion/
---

The [previous post in this series](https://ranjithkannan.com/2026/05/10/verus-calibration-formal-verifier-loop/) used Verus and an autonomous coding loop to verify three calibration exercises. The exercises were chosen because we needed to know whether the methodology survived pressure before staking real work on it. This post is the first piece of real work: a verified Byzantine quorum certificate library, with a safety lemma proven, produced end to end by the same loop. The infrastructure that the calibration tested now does what it was built to do.

A quorum certificate is the building block every Byzantine fault tolerant consensus protocol uses. PBFT, HotStuff, Tendermint, BFT-SMaRt, the lot of them, all carry around bundles of signed votes witnessing agreement on some proposal. The structural and cryptographic checks on these bundles are well-understood, but a clean, publicly-verified implementation in Verus does not exist as of this writing. Building one is the smallest concrete artifact on the path to formally verified safety-critical Byzantine systems, which is the wider problem this work is heading toward.

## What the exercise verified

The frozen spec sits at [`exercises/quorum_cert.rs`](https://github.com/ranjithkannank/verus-calibration/blob/main/exercises/quorum_cert.rs) and asks for two distinct obligations.

**Obligation 1: structural runtime check.** Given a `QuorumCert` and total node count `n`, return `true` iff the certificate has distinct voters, all voters fall in range `[0, n)`, and the count meets the Byzantine threshold `2n/3 + 1`. The cryptographic validity of signatures is treated as an uninterpreted predicate. That choice is deliberate. Real deployments connect signature verification to a vetted crypto library outside the verified module, and what we want to verify here is the BFT-layer reasoning that sits on top.

**Obligation 2: safety lemma.** If a QC is valid over `n` nodes and the Byzantine set has size strictly less than `n/3`, then the QC's voter set contains at least one voter that is not in the Byzantine set. This is the pigeonhole step that makes quorum certificates useful at all. Two valid QCs intersect in at least one honest voter, and that intersection is what prevents a Byzantine quorum of liars from producing conflicting certificates.

The spec uses `pub uninterp spec fn` for the cryptographic predicates, with no body. The implementer is forbidden from providing one. A consumer of the library connects them to a real implementation via a wrapper outside the repo:

```rust
pub uninterp spec fn pk_of(node: NodeId) -> PubKey;
pub uninterp spec fn signature_valid(pk: PubKey, msg: Hash, sig: Signature) -> bool;
```

## How it ran

The architect produced a 17 KB design note covering the representation choice (bitmap-backed structural check, the same pattern that worked for `quorum_count`), the proof shape (pigeonhole via contradiction, helper lemmas lifted from earlier exercises), and an ordered list of nine sub-tasks ranked easiest to hardest.

The implementer then made six attempts, each scoped to the next unfinished sub-task. The shape of the progression:

1. Skeleton: bitmap, early returns, threshold compare, no invariants.
2. Cursor bounds and in-range invariant.
3. Bitmap abstraction invariant (`seen[k]` iff the prefix contains `k`).
4. Pairwise distinctness invariant via the bitmap.
5. Helper lemmas lifted from `quorum_count`, threshold step verified.
6. Safety lemma via pigeonhole-by-contradiction, `12 verified, 0 errors`.

The reviewer's audit landed `APPROVE` with line-cited evidence against the frozen spec. The [full review](https://github.com/ranjithkannank/verus-calibration/blob/main/logs/quorum_cert/review.md) is in the repo.

## Patterns the run added to the playbook

Three patterns surfaced during the run that the implementer wrote back into [`AGENTS.md`](https://github.com/ranjithkannank/verus-calibration/blob/main/AGENTS.md)'s discovered-patterns section. All three are reusable for future BFT-shaped exercises.

**Pigeonhole-via-contradiction.** A proof of the form "there exists an honest voter" is most cleanly written as:

```rust
if !(exists|h: NodeId| voters(qc).contains(h) && !byzantine.contains(h)) {
    // negation gives a universal: forall h. !(P(h))
    // turn it into a subset relation between voters(qc) and byzantine
    // apply lemma_len_subset to get |voters(qc)| <= |byzantine|
    // arithmetic contradiction with has_quorum and the n > 3f assumption
    assert(false);
}
```

The negated existential gives a `forall`, which an `assert forall ... implies ... by { }` block can convert into a subset relation. Combined with `vstd::set_lib::lemma_len_subset`, the cardinality contradiction closes. This pattern recurs in any proof shape where the conclusion is "some element exists with property P" and the hypothesis bounds cardinalities.

**`lemma_fundamental_div_mod` for threshold arithmetic.** Verus's `nonlinear_arith` discharge does not know the basic euclidean identity `x == d * (x / d) + (x % d)`. The library lemma `vstd::arithmetic::div_mod::lemma_fundamental_div_mod(x, d)` provides it as an explicit primitive. Any reasoning that needs to bridge `(2*n)/3 + 1` to `n - byzantine.len()` style arguments will reach for this. The trick: pass `int`-typed arguments, then bridge to `nat`. The remainder bound `0 <= r < 3` is known to the solver; the identity is not.

**`lemma_len_subset` requires the superset finite, not the subset.** The signature is `lemma_len_subset(s1, s2) requires s1.subset_of(s2) && s2.finite() ensures s1.finite() && s1.len() <= s2.len()`. The lemma both lifts finiteness from a known-finite universe set down to an abstract `voters(qc)` set, and provides the cardinality bound. Reading the signature carefully matters: putting the finiteness hypothesis on the wrong set is the most common usage mistake.

The architect playbook in [`.claude/agents/architect.md`](https://github.com/ranjithkannank/verus-calibration/blob/main/.claude/agents/architect.md) is updated to mention these patterns directly. The reviewer's audit notes also explicitly recommended their promotion, which is the audit role doing its other job: noticing what works across exercises and pushing it upstream into the design phase.

## What the run is evidence of

Four signals worth naming.

**The methodology refinements applied since the original post pay off.** The per-iteration scoping change directs the implementer to the next unfinished sub-task. Six attempts, each making narrow progress, no thrashing. Without it, an attempt might rewrite the file trying to land both obligations at once and stall on composition complexity.

**The audit role does more than gatekeep.** The reviewer's report not only confirmed `APPROVE` against the five-point checklist, it flagged two patterns worth promoting to the architect's playbook (the pigeonhole-via-contradiction shape, the bitmap-abstraction + `Vec::set` framing). The audit role accumulates cross-exercise knowledge and feeds it back into design.

**Cross-exercise memory in `AGENTS.md` compounds.** The discovered-patterns section now contains entries from binary_search, bounded_log, quorum_count, and quorum_cert. The architect for quorum_cert was able to lift the universe-size lemma pattern verbatim from quorum_count because both the playbook and the file-based memory pointed at it. Each exercise's findings make the next one cheaper.

**The trust boundary stayed honest.** The cryptographic predicates in the spec are uninterpreted. The implementer did not add a body, did not add an `assume_specification`, did not introduce `external_body`. The reviewer's grep confirmed zero bypass tokens. The safety lemma works under the assumed cryptographic axioms; a real deployment supplies them.

## What this is not

The verified library is single-module. It verifies the BFT-layer structural and pigeonhole reasoning on top of cryptographic abstractions; it does not verify a complete consensus protocol. The next concrete artifact on the path, a verified Byzantine agreement primitive, needs multi-round messaging and multi-module reasoning, which the calibration's harness has not yet been stressed on.

The library is also not, on its own, a contribution to consensus algorithm design. The structural and pigeonhole arguments are classical. The contribution is the formally verified implementation in Rust and Verus, with the proof exposed as a reusable artifact, produced by an autonomous loop that refused to cheat.

## Where to find it

- Repo: <https://github.com/ranjithkannank/verus-calibration>
- The exercise: [`exercises/quorum_cert.rs`](https://github.com/ranjithkannank/verus-calibration/blob/main/exercises/quorum_cert.rs)
- The architect's design: [`exercises/quorum_cert.design.md`](https://github.com/ranjithkannank/verus-calibration/blob/main/exercises/quorum_cert.design.md)
- Per-attempt history: [`logs/quorum_cert/attempts.md`](https://github.com/ranjithkannank/verus-calibration/blob/main/logs/quorum_cert/attempts.md)
- Reviewer's audit: [`logs/quorum_cert/review.md`](https://github.com/ranjithkannank/verus-calibration/blob/main/logs/quorum_cert/review.md)
- Final state in git: commit [`5a2a87d`](https://github.com/ranjithkannank/verus-calibration/commit/5a2a87d)

## What's next on this path

A verified Byzantine agreement primitive (the textbook one-shot Lamport-Shostak-Pease problem) is the next step. After that, a verified sensor fusion algorithm where `2f+1` Byzantine-tolerant sensors produce an agreed-upon reading whose safety property constrains it to the convex hull of correct sensor outputs. After that, a hardware-deployed demonstration on dissimilar redundant boards.

Each step ships as it's done.

---

*This post is part of a series on the multi-agent Ralph loop. See [the architecture post](https://ranjithkannan.com/2026/04/10/multi-agent-tdd-loop/), [mutation testing](https://ranjithkannan.com/2026/04/19/pit-mutation-testing-ralph-loop/), [the Final Validator](https://ranjithkannan.com/2026/04/23/final-validator-ralph-loop/), [integration test contracts](https://ranjithkannan.com/2026/04/27/integration-test-contracts/), and [the formal verifier calibration](https://ranjithkannan.com/2026/05/10/verus-calibration-formal-verifier-loop/).*
