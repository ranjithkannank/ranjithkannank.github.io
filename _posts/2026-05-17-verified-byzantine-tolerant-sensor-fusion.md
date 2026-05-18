---
layout: post
title: "Verified Byzantine-Tolerant Sensor Fusion"
date: 2026-05-17
related:
  - /2026/04/10/multi-agent-tdd-loop/
  - /2026/05/10/verus-calibration-formal-verifier-loop/
---

Modern aircraft, drones, and increasingly autonomous vehicles all rely on multiple redundant sensors to know what's happening around them. Three accelerometers, two GPS receivers, a stack of inertial reference units. Each sensor's reading is uncertain, and any sensor can fail. Some failures are silent (the sensor stops reporting); some are deceptive (the sensor reports a value that's just plausible enough not to be flagged). The deceptive case is the harder one. Distributed-systems research has carried the name "Byzantine" for it since Lamport, Shostak, and Pease coined it in 1982.

The flight computer has to take these readings and produce a single trusted answer. "Three accelerometers say the aircraft is pitching up; one says it isn't" — what does the autopilot do? The textbook answer is fault-tolerant sensor fusion: an algorithm whose output is guaranteed safe as long as at most a known number of inputs are broken or lying.

This post is about two such algorithms, implemented in Rust with formal proofs of correctness, and three exercises that compose them into a verified Byzantine-tolerant sensor poll. All five were produced through an autonomous coding loop. One of the algorithm exercises surfaced a bug in our own specification that we wouldn't have caught without the loop's strict refusal to cheat. The third composition exercise was set up as a deliberate test of whether the loop can discover proofs rather than only execute pre-designed ones; it passed, and the result subsequently survived an audit re-run with the operator-authored witness file hidden. Those are the pieces worth reading for.

## The problem in plain English

Forget about distributed systems for a moment. You have three thermometers in a room. Two read 71°F; one reads 100°F. What's the temperature? Probably 71°F, and the third thermometer is broken. You made that call without thinking, because you carry an implicit fault-tolerance assumption: at most one of the three is wrong.

A flight computer makes the same kind of call, formally, dozens of times a second. Three altitude sensors, two GPS receivers, four inertial measurement units. Each sensor reports a value (or a range of values, allowing for its own uncertainty). The flight computer fuses them into a single trusted reading. Two requirements:

1. The fused reading is somewhere in the range any correct sensor would agree with.
2. If at most a known number of sensors are broken or lying, the first guarantee still holds.

The second requirement is what "Byzantine fault tolerance" means in this setting. A sensor can fail in any way at all: silent, stuck, drifting, actively reporting fictions to undermine the fusion, and the algorithm still has to produce a correct answer.

The minimum redundancy for tolerating `f` faulty sensors out of `n` total is `n ≥ 2f + 1`. With three sensors you can tolerate one fault. With five you can tolerate two. The arithmetic is independent of what the sensors are measuring.

What an algorithm of this shape looks like:

- **Fault-tolerant midpoint** (Schmid and Schossmaier, 2001). Take the readings, sort them, return the median. The median is guaranteed to lie inside the range that correct sensors agree on, given the redundancy assumption.
- **Marzullo's algorithm** (Marzullo, 1984). The same idea but for ranges instead of single numbers. Each sensor reports an interval `[lo, hi]` representing "I'm certain the true value is somewhere in here." The algorithm returns the smallest output interval whose interior contains a point that at least `n - f` input intervals also contain. By pigeonhole, at least one of those is a correct sensor's interval.

Both algorithms are old. Both are well-understood. The contribution isn't the algorithm; it's a publicly-verified implementation in a modern systems language, produced (mostly) by an autonomous coding loop.

## What "verified" buys you

Most production software is "trusted because it was tested." Run enough realistic inputs through it, check the outputs look right, ship it. That works for most code. It works less well for safety-critical code, because the inputs that break a fault-tolerant algorithm are exactly the unrealistic ones: a sensor that returns the maximum representable value, three sensors that all return the same wrong number, a sensor whose interval barely overlaps two others'. Test suites tend to miss these because nobody thinks to write the test.

A formal verifier replaces "trusted because tested" with "trusted because *proven*." We write down what the algorithm is supposed to do in a machine-checkable specification language, the verifier reads both the specification and the code, and either it produces a proof that the code satisfies the specification on every possible input, or it tells us which case it can't cover. The proof is checked by an SMT solver, the same kind of tool used in industrial chip design and in NASA's verified flight software.

We use [Verus](https://github.com/verus-lang/verus). It extends Rust with annotations for preconditions, postconditions, and proofs. The specification lives in the same file as the code; the verifier checks that the code body satisfies the postcondition under the preconditions. If it does, the file compiles. If it doesn't, we get a specific error pointing at the failing obligation.

This matters more than it sounds. A signed-off proof from a verifier is roughly the same kind of guarantee as a peer-reviewed mathematical theorem. It doesn't depend on the implementer being careful; it doesn't depend on the test author thinking of the right edge case; it doesn't depend on the language not introducing undefined behaviour. As long as the specification captures what we wanted, the implementation does it.

That last clause is doing a lot of work. We'll come back to it.

## What we're working with

The methodology was described in detail in a [previous post](https://ranjithkannan.com/2026/05/10/verus-calibration-formal-verifier-loop/). The short version: three agents, each implemented as a separate Claude Code subagent, talking through a bash script that drives a state machine. Each iteration of the loop spawns one fresh agent call, with the conversation context reset to empty. Memory between iterations lives entirely in files: the agent's written notes, the git history, the verifier output, the design.

The three roles:

- **Architect** (Opus 4.7). Reads the frozen specification, writes a design document with an ordered list of sub-tasks for the implementer to work through. Predicts the helper lemmas the proof will need.
- **Implementer** (Opus 4.7). One verifier attempt per iteration: edits the file to implement the next sub-task from the design, runs Verus, logs the result. Stops at the per-exercise iteration cap, or earlier if it can prove the spec is unprovable.
- **Reviewer** (Opus 4.7). Once verus accepts the file, audits the diff against the frozen baseline. Confirms no specifications were weakened, no verifier bypasses were introduced, no functions were trivialised. Returns APPROVE or REJECT.

Two structural rules make this load-bearing. The implementer cannot modify the frozen specification (any modification is a violation). And the implementer cannot use `assume(...)` or `#[verifier::external_body]` to bypass the verifier (these are on a banned-token list checked by a git pre-commit hook).

Code is at <https://github.com/ranjithkannank/verus-calibration> with the full per-attempt commit history.

## Two verified sensor-fusion artifacts

The sensor-fusion track builds on an earlier exercise, a verified Byzantine quorum certificate ([`exercises/quorum_cert.rs`][qc]), that proved the pigeonhole reasoning every Byzantine consensus protocol relies on. The two new artifacts apply the same machinery to sensor fusion.

**Fault-tolerant midpoint** ([`exercises/ft_midpoint.rs`][fm]) was the first. Input: a vector of `i64` sensor readings and a Byzantine bound `f`. Output: a single `i64` guaranteed to lie between the lowest and highest readings that honest sensors would have produced. The specification, in Verus:

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

The two postconditions say "there is a correct sensor whose reading is below the output, and a correct sensor whose reading is above." Together they bracket the output inside the range honest sensors agree on. The implementer's algorithm was a brute-force scan: for each candidate reading, count how many readings are `≤` it and how many are `≥` it; the first candidate with both counts at least `f + 1` is the answer. The proof of correctness used inclusion-exclusion over set cardinalities.

**Marzullo's algorithm** ([`exercises/marzullo.rs`][mz]) was the interval generalisation. Sensors report ranges instead of single values; the output is also a range. Same redundancy assumption.

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

The postcondition says: there's a point inside the output interval that at least `n - f` input intervals also contain. The `correct_intervals_overlap` precondition is the interesting one. It says all correct sensors' intervals share a common point, which is the standard assumption that honest sensors are all reporting bounds around the same underlying true value. We didn't include this precondition the first time we wrote the specification. That's the story below.

[qc]: https://github.com/ranjithkannank/verus-calibration/blob/main/exercises/quorum_cert.rs
[fm]: https://github.com/ranjithkannank/verus-calibration/blob/main/exercises/ft_midpoint.rs
[mz]: https://github.com/ranjithkannank/verus-calibration/blob/main/exercises/marzullo.rs

## When the loop refused to verify

The first run of marzullo failed. Not in a confusing way; in the most useful way we've seen the loop fail.

The implementer worked through the architect's sub-task list for six iterations, building proof scaffolding, defining helper lemmas, constructing a counting function. It got 8 of 9 verifier obligations to verify. The single failing assertion was a step in the safety lemma that required showing `intervals[i].lo ≤ intervals[j].hi` for two correct sensor indices `i` and `j`. Exactly the Helly-1D overlap condition that the algorithm's correctness depends on.

The verifier refused to accept that assertion. Correctly. Nothing in the specification said anything about correct sensors' intervals overlapping. As written, two "correct" sensors were allowed to report intervals like `[0, 0]` and `[10, 10]`, disjoint singletons satisfying every precondition we'd actually written. Under that allowed model the specification's postcondition was unsatisfiable: no point on the real line lies in both intervals, so no point can be in `n - f = 2` of the three intervals `[[0,0], [10,10], [20,20]]`.

The implementer noticed this. It produced a constructive counterexample, wrote a structured blocker report, and stopped. It did not try to weaken the specification. It did not silently fill in an `assume(intervals[i].lo <= intervals[j].hi)`. It did not mark the function as externally verified. It produced concrete evidence that the specification we'd authored was logically wrong, and it surfaced that as a clean signal rather than a soft failure.

The architect, re-invoked under the methodology's escalation path, read the blocker report and confirmed the diagnosis. Three separate times, in three independent revisions of the design document. Each revision came to the same conclusion: the specification is missing a precondition; the implementer should write a blocked.md, not try further algorithmic variants.

We (the operator) read all of this and realised the missing precondition was the Helly-1D condition. Honest sensors all observe the same underlying value, so their reported intervals all contain that value, so any two of them share at least one common point. It's the assumption that makes the algorithm meaningful in the first place, and we'd forgotten to write it down. One line of specification:

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

The three verified primitives — `quorum_cert`, `ft_midpoint`, `marzullo` — are uncomposed. A verified Byzantine-tolerant sensor poll that authenticates signed sensor reports via the quorum-style check and combines the authenticated readings via Marzullo would be the first end-to-end system on the path. We built three composition exercises to take it on, each adding one piece of what that real system would need. The first demonstrated the composition regime. The second threaded the cryptographic trust boundary through the contract. The third strengthened the postcondition with an honest-voter guarantee and ran as a deliberate test of the methodology itself. All three verified in one attempt each.

### `sensor_poll`

A new exercise in a multi-file directory layout:

```text
exercises/sensor_poll/
    main.rs      # mod fusion; mod auth; poll(reports, n, f)
    fusion.rs    # marzullo
    auth.rs      # SensorReport + distinct_sensors + check_distinct
    design.md
```

The `fusion` module is a verbatim port of the verified `marzullo` exercise: same types, same uninterp `correct_at` trust boundary, same `open spec fn` definitions, same algorithm, same proof helpers. The `auth` module defines a `SensorReport` carrying a `sensor_id` and an `interval`, plus a `distinct_sensors` predicate and a `check_distinct` exec function that decides it. The implementation is the bitmap-backed single-pass pattern from `quorum_cert::verify_qc_structure`, simplified to one vector and one threshold.

The `main` module defines `poll(reports, n, f) -> Option<Interval>`. Call `check_distinct`; if false, return `None`. Otherwise project each `SensorReport`'s `interval` field into a fresh `Vec<Interval>`. Call `marzullo` on that. Use `choose` to extract the witness point from `marzullo`'s existential, then a one-line projection lemma to bridge the frame.

The projection lemma is the load-bearing piece. Marzullo's postcondition is stated in terms of `intervals_containing(intervals@, p)`, a set of indices into the projected `Seq<Interval>`. The caller of `poll` wants a fact about `reports_containing(reports@, p)`, a set of indices into the original `Seq<SensorReport>`. The two sets are extensionally equal because the projection just pulls out the `interval` field; the membership predicates align after substitution. Verus closes the equality with `=~=` alone:

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

That empty-body lemma is the entire composition seam. Everything else is plumbing. The agent verified the exercise on the first attempt: 16 verified, 0 errors. Reviewer APPROVE.

### `sensor_poll_signed`

The first exercise's `auth` module ported only the bitmap-distinct half of `quorum_cert`. "Distinct sensor IDs" is a much weaker property than "valid signatures over signed reports." The second exercise closed that gap at the contract layer.

`fusion.rs` and the structural half of `auth.rs` are unchanged. `auth.rs` gains the cryptographic trust boundary lifted from `quorum_cert`: `Hash`, `PubKey`, `Signature` type aliases; three uninterpreted spec predicates (`pk_of`, `signature_valid`, `report_msg`); two open spec predicates (`all_signatures_valid` and a `valid_report_bundle` conjunction of distinct-and-signed); and a `sig: Signature` field on `SensorReport`. `poll`'s precondition gains `all_signatures_valid(reports@)`; its `Some`-branch ensures gains `valid_report_bundle(reports@)`.

What the exec layer does not gain: a signature-verification function. `signature_valid` stays opaque, the same way `quorum_cert.rs` left it. A real deployment would connect it to a vetted external crypto library via a thin exec wrapper outside the repo. The trust boundary in this exercise lives entirely at the spec layer: the caller of `poll` is responsible for having verified signatures upstream, the way a real Byzantine protocol receives already-signed messages from its network stack. The composition theorem now states that `poll` only returns `Some` when the inputs constitute a valid signed bundle, which is a real strengthening even though no exec-layer signature check happens inside the exercise.

The implementer's new work was a one-line conjunction: `assert(valid_report_bundle(reports@));` after `check_distinct` returns true. `distinct_sensors(reports@)` is in scope from `check_distinct`'s ensures, and `all_signatures_valid(reports@)` is in scope from the precondition. Verus joins them. The rest of `poll`'s body is byte-equivalent to the first exercise's. One attempt, 16 verified, 0 errors, reviewer APPROVE.

### `sensor_poll_honest`

The third exercise pushed harder. The first two carried a caveat: the design note pre-named the load-bearing proof construct, so the agent executed a designed proof rather than discovering one. Methodology that handles execution of designed proofs is a narrower claim than methodology that supports discovery. This exercise was set up specifically to test the discovery half.

Its `fusion.rs` and `auth.rs` are byte-identical to `sensor_poll_signed`. Its `main.rs` adds one conjunct to `poll`'s `Some`-branch ensures:

```rust
&&& exists|p: Reading, k: int|
    interval.lo <= p && p <= interval.hi
    && 0 <= k < reports.len()
    && correct_at(k)
    && point_in_interval(p, reports[k].interval)
```

There exists a point `p` in the returned interval AND an index `k` such that sensor `k` is honest (not Byzantine) AND its reported interval contains `p`. This is the BFT-meaningful strengthening: the signature trust boundary is now load-bearing in the proof, not just threaded through the contract. The returned interval is provably backed by at least one honest sensor's report.

The design note for this exercise was deliberately incomplete. It stated the obligation. It stated the informal mathematical content: that `n - f` supporters and `n - f` correct sensors, both subsets of an `n`-element universe, must overlap, and that with `n ≥ 2f + 1` the overlap has at least one element. It did not name the supporting lemmas. It did not name the helper-set constructions. It did not name the trigger annotations. It did not name the sub-proof structure. The architect's playbook in `AGENTS.md` does name those constructs, including the inclusion-exclusion identity `lemma_set_intersect_union_lens` and the universe-finite bridge via `lemma_int_range` and `lemma_len_subset`, but those entries sit under `ft_midpoint`, a different exercise with a different proof obligation in the same proof family.

The agent verified in one attempt. Its proof introduced a new helper lemma `lemma_honest_supporter_exists`, establishing both index sets as subsets of `[0, n)` via `lemma_int_range` and `lemma_len_subset`, applying `lemma_set_intersect_union_lens` to get `|s ∪ c| + |s ∩ c| == |s| + |c|`, bounding `|s ∪ c| ≤ n`, concluding `|s ∩ c| ≥ 2(n − f) − n ≥ 1`, then using `axiom_is_empty_len0` / `axiom_is_empty` to extract the witness from the non-empty intersection. The lemma's name, signature, proof structure, choice of helper-set construction, and specific use of the inclusion-exclusion identity were all the agent's. The design note named none of them. The agent recognised that the proof family from `ft_midpoint`'s playbook entry applied to a new situation in a different exercise on a different obligation, and reused it.

One data point on one proof family. It moves the designed-vs-discovered axis from "untested, plausible caveat" to "tested once, supports discovery within an established family." The next move on the methodology axis, testing whether the loop can discover proofs on a family the playbook does not document at all, is a separate single-function exercise (`swap_multiset`) covered in the [calibration post](https://ranjithkannan.com/2026/05/10/verus-calibration-formal-verifier-loop/), which also tells the find-fix-audit story behind hardening the agent's witness-access permissions for these tests.

## What compounded across exercises

Each verified exercise added something to a shared playbook the next exercise could pick up.

Quorum_cert produced the pigeonhole-via-contradiction shape: when the goal is "some honest member exists in this set," wrap the negation in an `if !(exists ...) { ... assert(false); }` block, derive a subset relation from the universal that the negated existential gives you, and apply `lemma_len_subset` for the cardinality contradiction.

Ft_midpoint produced the inclusion-exclusion shape via `vstd::set_lib::lemma_set_intersect_union_lens`. Same pigeonhole target, but constructed forward through set arithmetic rather than backward through contradiction. The reviewer flagged both shapes as worth knowing across exercises.

Marzullo produced a third: argmax-plus-Helly-1D for constructive existence. When the spec admits a witness construction (via a geometric property like overlap), it's cleaner than either pigeonhole shape because the proof gives you the witness directly rather than inferring it from a contradiction or arithmetic. The reviewer's audit note recommended promoting this to the architect's playbook for future sensor-fusion exercises.

The pattern is more general than the specific lemmas. Each exercise's architect could draw on the previous exercises' designs and verified machinery. The implementer didn't need to re-derive proof shapes that had already been worked out elsewhere; it could lift them. After six exercises the architect's role file has a long enough playbook that the marzullo restart converged in one attempt, with the implementer reusing scaffolding from ft_midpoint and from the prior (blocked) marzullo run.

The discovery question the playbook raised, whether the agent could recognise a known proof family and apply it to a new obligation without the architect spelling out the construct, got a clearer answer through `sensor_poll_honest`. The one-attempt verification of the honest-voter clause introduced a new helper lemma using inclusion-exclusion recognised from `ft_midpoint`'s playbook entry. That was one data point on one proof family. A second, structurally different test on a different proof family (`counter_filler`, in the multi-module track covered in the [calibration post](https://ranjithkannan.com/2026/05/10/verus-calibration-formal-verifier-loop/)) produced the same shape of result: one attempt, a new invariant derived from a different playbook entry, no thrashing. Both were subsequently audited under a tool whitelist that explicitly denies the agent reading the operator-authored witness file, with each exercise's prior playbook summary stripped from `AGENTS.md`. Both audits passed in one attempt with solutions structurally identical to the originals. Two data points, two distinct proof families, audit-confirmed. The methodology supports discovery within an established proof family. Whether it supports *invention* on a family the playbook does not document at all is a different test, also covered in the calibration post.

## Honest limitations

Nine verified exercises is not a benchmark. The Verus vericoding research papers operate on hundreds to thousands of tasks; nine lets us observe failure modes, not measure success rates.

Every primitive-track exercise (binary_search through marzullo) is single-module. The composition track is multi-module via Rust's standard `mod` mechanism, three siblings inside one Verus crate. That covers cross-module reasoning within a single crate; it does not cover genuinely cross-crate dependencies (one Verus crate importing another's `pub` interface), which is the regime real systems live in. We expect new failure modes when we get there.

The cryptographic trust boundary in the quorum certificate exercise is uninterpreted. The implementer cannot provide a body; the verifier reasons over all possible meanings of "this signature is valid." A real deployment connects this to a vetted crypto library through a thin wrapper that supplies the assumed behaviour. We verified the BFT-layer reasoning, not the cryptography underneath. The composition exercises inherit this boundary unchanged.

Two of the six primitive-track exercises required operator intervention to fix specification bugs. Both were caught and fixed with one targeted edit, but the underlying issue, that operator-authored specifications can be wrong, is real. The methodology refinement that catches this class at operator time, pre-verifying that a specification admits a satisfying model via an operator-authored witness file, has since been added (see the [calibration post](https://ranjithkannan.com/2026/05/10/verus-calibration-formal-verifier-loop/) for the find-fix-audit story). None of the composition exercises needed it; their specs admitted models cleanly on first authoring.

One harness bug surfaced and was fixed: a stuck-state loop where the escalation marker file couldn't be deleted (the agent's tool whitelist denied `rm`) and the state classifier treated an empty file as still-escalated. The orchestrator now cleans up the marker explicitly and the state classifier uses a non-empty check. This is the kind of refinement the autonomous-loop literature doesn't typically discuss; the fix is local and small, but the failure mode was non-obvious and only became visible after the first escalation in any exercise.

The composition exercises do not import the verified primitives as crate dependencies. Each existing exercise compiles as its own Verus crate via `verus <file>.rs --crate-type=lib`; there is no `Cargo.toml`, no workspace, no dependency arrows between exercises. All three composition exercises *port* the primitives, re-implementing them as sibling modules inside their own crate. From a verification standpoint the proofs are real; from a "reuse the verified artifacts" standpoint we did not reuse them as artifacts, we copied their source. Restructuring the existing exercises as importable Verus crates is in `BACKLOG.md`.

The composition exercises also do not push signature verification into the exec layer. `sensor_poll_signed` and `sensor_poll_honest` carry the `signature_valid` uninterp predicate and reason about it at the spec layer. Neither exercise has an exec function that walks each report and calls a cryptographic verifier; that step is left to the caller, and ultimately to an external library connected via an `assume_specification` outside the repo, the way `quorum_cert.rs` is structured. Adding the exec-layer trust boundary is another item in BACKLOG.

The composition exercises do not use `ft_midpoint`. All three pick `marzullo` (intervals) over `ft_midpoint` (scalar readings). One choice; we made it for tractability. The verified `ft_midpoint` sits unused in the composition track.

And the three composition exercises are not "a small system." They are three verified end-to-end functions, each building on the last. A system would have multiple flows, a configuration surface, integration points beyond a single composition call, probably real I/O. What we have is a chain of three functions whose postconditions depend on two other functions' postconditions via a projection lemma and, in the third exercise, an inclusion-exclusion lemma. That is composition reasoning, demonstrated; it is not a system, claimed.

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
