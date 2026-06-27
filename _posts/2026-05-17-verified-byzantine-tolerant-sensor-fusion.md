---
layout: post
title: "Verified Byzantine-Tolerant Value Agreement"
date: 2026-05-17
related:
  - /2026/04/10/multi-agent-tdd-loop/
  - /2026/05/10/verus-calibration-formal-verifier-loop/
---

Marzullo's algorithm comes from a 1983 PhD thesis on distributed clock synchronization. Keith Marzullo wanted to keep a set of computer clocks in sync when some clock sources might be unreliable or lying — silent, drifting, or actively reporting fictional times. Each source reports an interval `[lo, hi]` containing what it believes to be the true time; the algorithm fuses these intervals into a single trusted interval, tolerating up to `f` faulty sources out of `n` provided `n ≥ 2f + 1`. The algorithm became foundational to NTP, and to a broader family of Byzantine-tolerant value-agreement primitives: replica voting, source agreement, any setting where multiple noisy reports on a single underlying value have to be fused and some reports may be lying.

Distributed-systems research has carried the name "Byzantine" for the deceptive-failure case since Lamport, Shostak, and Pease coined it in 1982. The silent case is easier — a source that returns nothing is one a protocol can plan for. A source that returns a value plausible enough not to be flagged is harder, and that case is the one Byzantine fault tolerance is named after.

This post is about two such algorithms, implemented in Rust with formal proofs of correctness, and three exercises that compose them into a verified Byzantine-tolerant value aggregator. All five were produced through an autonomous coding loop. One of the algorithm exercises surfaced a bug in our own specification that we wouldn't have caught without the loop's strict refusal to cheat. The third composition exercise was set up as a deliberate test of whether the loop can discover proofs rather than only execute pre-designed ones; it passed, and the result subsequently survived an audit re-run with the operator-authored witness file hidden. Those are the pieces worth reading for.

## The problem in plain English

Forget about distributed systems for a moment. You have three thermometers in a room. Two read 71°F; one reads 100°F. What's the temperature? Probably 71°F, and the third thermometer is broken. You made that call without thinking, because you carry an implicit fault-tolerance assumption: at most one of the three is wrong.

A distributed consensus protocol makes the same kind of call on every commit, formally. Each source reports a value (or a range of values, allowing for its own uncertainty). The protocol fuses them into a single trusted reading. Two requirements:

1. The fused reading is somewhere in the range any correct source would agree with.
2. If at most a known number of sources are broken or lying, the first guarantee still holds.

The second requirement is what "Byzantine fault tolerance" means in this setting. A source can fail in any way at all: silent, stuck, drifting, actively reporting fictions to undermine the fusion, and the algorithm still has to produce a correct answer.

The minimum redundancy for tolerating `f` faulty sources out of `n` total is `n ≥ 2f + 1`. With three sources you can tolerate one fault. With five you can tolerate two. The arithmetic is independent of what the sources are reporting.

What an algorithm of this shape looks like:

- **Fault-tolerant midpoint** (Schmid and Schossmaier, 2001). Take the readings, sort them, return the median. The median is guaranteed to lie inside the range that correct sources agree on, given the redundancy assumption.
- **Marzullo's algorithm** (Marzullo, 1984). The same idea but for ranges instead of single numbers. Each source reports an interval `[lo, hi]` representing "I'm certain the true value is somewhere in here." The algorithm returns the smallest output interval whose interior contains a point that at least `n - f` input intervals also contain. By pigeonhole, at least one of those is a correct source's interval.

Both algorithms are old. Both are well-understood. The contribution isn't the algorithm; it's a publicly-verified implementation in a modern systems language, produced (mostly) by an autonomous coding loop.

## Two verified value-agreement primitives

The methodology — three-role autonomous loop, three boundaries, fresh context per attempt — is the subject of the [calibration post](https://ranjithkannan.com/2026/05/10/verus-calibration-formal-verifier-loop/). Code is at <https://github.com/ranjithkannank/verus-calibration>.

The value-agreement track builds on an earlier exercise, a verified Byzantine quorum certificate ([`exercises/quorum_cert.rs`][qc]), that proved the pigeonhole reasoning every Byzantine consensus protocol relies on. The two new artifacts apply the same machinery to fusing multiple noisy reports of an underlying value.

**Fault-tolerant midpoint** ([`exercises/ft_midpoint.rs`][fm]) was the first. Input: a vector of `i64` source readings and a Byzantine bound `f`. Output: a single `i64` guaranteed to lie between the lowest and highest readings that honest sources would have produced. The specification, in Verus:

```rust
pub fn ft_midpoint(readings: &Vec<Reading>, f: u32) -> (result: Reading)
    requires
        readings.len() as nat >= 2 * (f as nat) + 1,
        correct_indices(readings.len() as nat).len()
            >= readings.len() as nat - f as nat,
    ensures
        some_correct_le(readings@, result),
        some_correct_ge(readings@, result),
```

The two postconditions say "there is a correct source whose reading is below the output, and a correct source whose reading is above." Together they bracket the output inside the range honest sources agree on. The implementer's algorithm was a brute-force scan: for each candidate reading, count how many readings are `≤` it and how many are `≥` it; the first candidate with both counts at least `f + 1` is the answer. The proof of correctness used inclusion-exclusion over set cardinalities.

**Marzullo's algorithm** ([`exercises/marzullo.rs`][mz]) was the interval generalisation. Sources report ranges instead of single values; the output is also a range. Same redundancy assumption.

```rust
pub fn marzullo(intervals: &Vec<Interval>, f: u32) -> (result: Interval)
    requires
        intervals.len() as nat >= 2 * (f as nat) + 1,
        well_formed(intervals@),
        correct_indices(intervals.len() as nat).len()
            >= intervals.len() as nat - f as nat,
        correct_intervals_overlap(intervals@),
    ensures
        result.lo <= result.hi,
        exists|p: Reading|
            result.lo <= p && p <= result.hi
                && intervals_containing(intervals@, p).len()
                   >= intervals.len() as nat - f as nat,
```

The postcondition says: there's a point inside the output interval that at least `n - f` input intervals also contain. The `correct_intervals_overlap` precondition is the interesting one. It says all correct sources' intervals share a common point, which is the standard assumption that honest sources are all reporting bounds around the same underlying true value. We didn't include this precondition the first time we wrote the specification. That's the story below.

[qc]: https://github.com/ranjithkannank/verus-calibration/blob/main/exercises/quorum_cert.rs
[fm]: https://github.com/ranjithkannank/verus-calibration/blob/main/exercises/ft_midpoint.rs
[mz]: https://github.com/ranjithkannank/verus-calibration/blob/main/exercises/marzullo.rs

## When the loop refused to verify

The first run of marzullo failed. Not in a confusing way; in the most useful way we've seen the loop fail.

The implementer worked through the architect's sub-task list for six iterations, building proof scaffolding, defining helper lemmas, constructing a counting function. It got 8 of 9 verifier obligations to verify. The single failing assertion was a step in the safety lemma that required showing `intervals[i].lo ≤ intervals[j].hi` for two correct source indices `i` and `j`. Exactly the Helly-1D overlap condition that the algorithm's correctness depends on.

The verifier refused to accept that assertion. Correctly. Nothing in the specification said anything about correct sources' intervals overlapping. As written, two "correct" sources were allowed to report intervals like `[0, 0]` and `[10, 10]`, disjoint singletons satisfying every precondition we'd actually written. Under that allowed model the specification's postcondition was unsatisfiable: no point on the real line lies in both intervals, so no point can be in `n - f = 2` of the three intervals `[[0,0], [10,10], [20,20]]`.

The implementer noticed this. It produced a constructive counterexample, wrote a structured blocker report, and stopped. It did not try to weaken the specification. It did not silently fill in an `assume(intervals[i].lo <= intervals[j].hi)`. It did not mark the function as externally verified. It produced concrete evidence that the specification we'd authored was logically wrong, and it surfaced that as a clean signal rather than a soft failure.

The architect, re-invoked under the methodology's escalation path, read the blocker report and confirmed the diagnosis. Three separate times, in three independent revisions of the design document. Each revision came to the same conclusion: the specification is missing a precondition; the implementer should write a blocked.md, not try further algorithmic variants.

We (the operator) read all of this and realised the missing precondition was the Helly-1D condition. Honest sources all observe the same underlying value, so their reported intervals all contain that value, so any two of them share at least one common point. It's the assumption that makes the algorithm meaningful in the first place, and we'd forgotten to write it down. One line of specification:

```rust
pub open spec fn correct_intervals_overlap(intervals: Seq<Interval>) -> bool {
    forall|i: int, j: int|
        0 <= i < intervals.len() && 0 <= j < intervals.len()
        && correct_at(i) && correct_at(j)
            ==> intervals[i].lo <= intervals[j].hi
}
```

We added it as a precondition, force-moved the frozen-specification tag to the corrected version, and restarted the loop. The implementer verified the file in a single attempt by lifting the proof scaffolding from the prior (blocked) run and plugging in the new precondition at the one failing line.

The point of this story isn't that the agent caught our bug. The point is that the strict no-cheating rule is what made the agent's behaviour useful. Without the rule, the loop would have either (a) silently weakened the postcondition until it could be proved trivially, (b) reached for `assume` to paper over the failing step, or (c) just produced a confused output we'd have to interpret. With the rule, we got a structured report containing a constructive counterexample, a diagnosis of which precondition was missing, and a suggested amendment. The kind of output you actually want from a trusted methodology.

This was the second time the methodology surfaced an operator-authored specification bug as a clean signal. The first was on an earlier exercise where we'd used Verus syntax that the current compiler version had deprecated. Same shape: the implementer articulated the conflict precisely, the architect confirmed it, the operator fixed the specification with a targeted edit, the loop resumed and converged in one attempt.

In a sample of six verified exercises, two of them required operator intervention to fix specification bugs that surfaced through the loop's refusal to cheat. That's a meaningful rate. It's also, on reflection, exactly the rate you'd want: the loop is doing the specification-authoring work the operator skipped, by trying to verify it.

## Composing the primitives

The three verified primitives — `quorum_cert`, `ft_midpoint`, `marzullo` — are uncomposed. A verified Byzantine-tolerant value aggregator that authenticates signed source reports via the quorum-style check and combines the authenticated readings via Marzullo would be the first end-to-end system on the path. We built three composition exercises to take it on, each adding one piece of what that real system would need. The first demonstrated the composition regime. The second threaded the cryptographic trust boundary through the contract. The third strengthened the postcondition with an honest-voter guarantee and ran as a deliberate test of the methodology itself. All three verified in one attempt each.

### `sensor_poll`

A multi-file directory layout: `main.rs` (`mod fusion; mod auth; poll(reports, n, f)`), `fusion.rs` (verbatim port of verified `marzullo`), `auth.rs` (`SensorReport` + `distinct_sensors` + `check_distinct` from `quorum_cert::verify_qc_structure`).

The `main` module's `poll(reports, n, f) -> Option<Interval>` calls `check_distinct`, projects each `SensorReport`'s `interval` field into a fresh `Vec<Interval>`, calls `marzullo`, uses `choose` to extract the witness point, and uses a one-line projection lemma to bridge the frame. The projection lemma is the load-bearing piece. Marzullo's postcondition is stated in terms of `intervals_containing(intervals@, p)`, a set of indices into the projected `Seq<Interval>`. The caller wants a fact about `reports_containing(reports@, p)`. The two sets are extensionally equal:

```rust
proof fn lemma_reports_eq_intervals_containing(
    reports: Seq<SensorReport>, p: Reading)
    ensures
        reports_containing(reports, p)
            =~= intervals_containing(project_intervals(reports), p),
{
    // body intentionally empty
}
```

That empty-body lemma is the entire composition seam. First attempt: 16 verified, 0 errors. Reviewer APPROVE.

### `sensor_poll_signed`

The second exercise added the cryptographic trust boundary at the spec layer. `auth.rs` gains `Hash`, `PubKey`, `Signature` type aliases; uninterpreted spec predicates `pk_of`, `signature_valid`, `report_msg`; open spec predicates `all_signatures_valid` and `valid_report_bundle`; a `sig: Signature` field on `SensorReport`. `poll`'s precondition gains `all_signatures_valid(reports@)`; its `Some`-branch ensures gains `valid_report_bundle(reports@)`.

The exec layer does not gain a signature-verification function. `signature_valid` stays opaque; the caller is responsible for having verified signatures upstream. The implementer's new work was a one-line conjunction: `assert(valid_report_bundle(reports@));` after `check_distinct` returns true. One attempt, 16 verified, reviewer APPROVE.

### `sensor_poll_honest`

The third exercise was set up specifically as a discovery test. `fusion.rs` and `auth.rs` byte-identical to `sensor_poll_signed`; `main.rs` adds one conjunct to `poll`'s `Some`-branch ensures:

```rust
&&& exists|p: Reading, k: int|
    interval.lo <= p && p <= interval.hi
    && 0 <= k < reports.len()
    && correct_at(k)
    && point_in_interval(p, reports[k].interval)
```

There exists a point `p` in the returned interval and an index `k` such that source `k` is honest and its reported interval contains `p`. The signature trust boundary is now load-bearing in the proof, not just threaded through the contract.

The design note was deliberately incomplete: stated the obligation and the informal `n - f` supporters + `n - f` correct sources overlap argument, but named no lemmas, helper-set constructions, trigger annotations, or sub-proof structure. The architect's playbook does name those constructs under `ft_midpoint`, a different exercise.

The agent verified in one attempt. Its proof introduced `lemma_honest_supporter_exists`, establishing both index sets as subsets of `[0, n)` via `lemma_int_range` and `lemma_len_subset`, applying `lemma_set_intersect_union_lens` to get `|s ∪ c| + |s ∩ c| == |s| + |c|`, concluding `|s ∩ c| ≥ 2(n − f) − n ≥ 1`, then extracting the witness from the non-empty intersection. The lemma's name, signature, proof structure, and specific use of the identity were all the agent's. The agent recognised that the proof family from `ft_midpoint`'s playbook entry applied to a new obligation. One data point on one proof family; the invention-test story (where the playbook documents nothing of the family) is in the [calibration post](https://ranjithkannan.com/2026/05/10/verus-calibration-formal-verifier-loop/).

## What compounded across exercises

Each verified exercise added something to a shared playbook the next exercise could pick up. Quorum_cert produced pigeonhole-via-contradiction (`if !(exists ...) { ... assert(false); }`, derive a subset relation, apply `lemma_len_subset`). Ft_midpoint produced the inclusion-exclusion shape via `lemma_set_intersect_union_lens` — forward through set arithmetic rather than backward through contradiction. Marzullo produced a third: argmax-plus-Helly-1D for constructive existence, cleaner than either pigeonhole shape because the proof gives the witness directly.

After six exercises the playbook was long enough that the marzullo restart converged in one attempt, the implementer reusing scaffolding from ft_midpoint and the prior blocked run. The discovery question — whether the agent could recognise a proof family and apply it to a new obligation without the architect spelling out the construct — got a clearer answer through `sensor_poll_honest` and `counter_filler`: two data points on two distinct proof families, both audit-confirmed under the hardened witness-deny whitelist with prior playbook summaries stripped. The invention question is a different test, covered in the [calibration post](https://ranjithkannan.com/2026/05/10/verus-calibration-formal-verifier-loop/).

## Honest limitations

Nine verified exercises is not a benchmark. The Verus vericoding research papers operate on hundreds to thousands of tasks.

The composition exercises do not import the verified primitives as crate dependencies. Each existing exercise compiles as its own Verus crate; there is no `Cargo.toml`, no workspace, no dependency arrows. All three composition exercises *port* the primitives as sibling modules. From a verification standpoint the proofs are real; from a "reuse the verified artifacts" standpoint we copied source. Restructuring as importable Verus crates is in `BACKLOG.md`.

The composition exercises also do not push signature verification into the exec layer. `signature_valid` stays opaque; the caller is responsible. Adding the exec-layer trust boundary is another `BACKLOG.md` item.

The composition exercises do not use `ft_midpoint`. All three pick `marzullo` (intervals) over `ft_midpoint` (scalar readings). The verified `ft_midpoint` sits unused.

And the three composition exercises are not "a small system." They are three verified end-to-end functions, each building on the last. A system would have multiple flows, a configuration surface, real I/O. What we have is composition reasoning, demonstrated; not a system, claimed.

## Where to find everything

- Repo: <https://github.com/ranjithkannank/verus-calibration>
- The verified primitives:
  - [`exercises/quorum_cert.rs`](https://github.com/ranjithkannank/verus-calibration/blob/main/exercises/quorum_cert.rs)
  - [`exercises/ft_midpoint.rs`](https://github.com/ranjithkannank/verus-calibration/blob/main/exercises/ft_midpoint.rs)
  - [`exercises/marzullo.rs`](https://github.com/ranjithkannank/verus-calibration/blob/main/exercises/marzullo.rs)
- The composition exercises:
  - [`exercises/sensor_poll/`](https://github.com/ranjithkannank/verus-calibration/tree/main/exercises/sensor_poll)
  - [`exercises/sensor_poll_signed/`](https://github.com/ranjithkannank/verus-calibration/tree/main/exercises/sensor_poll_signed)
  - [`exercises/sensor_poll_honest/`](https://github.com/ranjithkannank/verus-calibration/tree/main/exercises/sensor_poll_honest)
- The methodology, witness-access hardening, and find-fix-audit story: <https://ranjithkannan.com/2026/05/10/verus-calibration-formal-verifier-loop/>
- The marzullo operator-intervention case in full detail, including the constructive counterexample and the architect's three revisions: the prior frozen tag's `logs/marzullo/blocked.md`, preserved in git history at commit [`c859e6f`](https://github.com/ranjithkannank/verus-calibration/commit/c859e6f).
