---
layout: post
title: "Integration Test Contracts: When Green Doesn't Mean Correct"
date: 2026-04-27
related:
  - /2026/04/10/multi-agent-tdd-loop/
  - /2026/04/23/final-validator-ralph-loop/
---

The [multi-agent Ralph loop](https://ranjithkannan.com/2026/04/10/multi-agent-tdd-loop/) treats integration tests as deterministic proof that the deployed system satisfies requirements. This post is about what happens when that proof is hollow: tests that go green without testing anything, and a contract mechanism that prevents it.

## The Problem

Integration tests can go green without testing anything. The tests that look most like work done sometimes aren't. Patterns we keep finding:

```java
@Test
void testFullPipeline() {
    var response = pipeline.run(request);
    if (!response.isSuccess()) {
        return; // silent pass when the deployed service is flaky
    }
    assertEquals(expected, response.getResult());
}
```

JaCoCo shows coverage. JUnit shows green. No assertion ran.

```java
@Test
void testBackfillResume() {
    var response = client.resume(executionId);
    assertNotNull(response); // the mock returns non-null. passes forever.
}
```

Technically an assertion. Practically meaningless.

```java
@Test
void testQueryTruncation() {
    var response = client.query(largePayload);
    assertTrue(response.isSuccess());
    // doesn't check the truncation field, which was the whole point.
}
```

Assertion on the wrong thing.

A single LLM reviewing its own integration tests tends to miss these. The review step rationalizes what it wrote five minutes earlier. We needed a stricter mechanism.

## What We Added

A per-spec contract file plus enforcement across two agents. The contract is the source of truth for what the integration tests must do. If a spec includes a `.integ-test-contract.json`, the loop's integ-tester and final-validator both honor it.

## The Contract File

Written by the engineer up front, either by hand or by asking the spec-writer agent to produce one from the requirements. Shape:

```json
{
  "spec": "order-processing-sf",
  "conventions": {
    "forbidEarlyReturn": true,
    "forbidEarlyReturnDescription": "No test may contain return; inside an if (!response.isSuccess()) block. Tests that expect success MUST call assertTrue(response.isSuccess(), ...) unconditionally.",
    "assertionPatterns": ["assertTrue", "assertEquals", "assertNotNull", "verify", "assertThat"],
    "sourceRoot": "src/OrderServiceTests/src/main/java/..."
  },
  "tests": {
    "OrderApiIntegrationTest.testDryRun_SetupConfig": {
      "task": "29.INTEG",
      "mustPass": true,
      "minAssertions": 2,
      "mustAssertFields": ["success", "dryRun"],
      "forbidEarlyReturn": true,
      "description": "SETUP_CONFIG with dryRun=true must return {success: true, dryRun: true}"
    }
  }
}
```

Each row names a test method and specifies:

- **`task`**: which `.INTEG` task this row belongs to
- **`mustPass`**: the test must actually pass when run, not just exist
- **`minAssertions`**: minimum count of assertion/verify calls in the method body
- **`mustAssertFields`**: response fields or state properties the test must check
- **`forbidEarlyReturn`**: no `if (!x) return;` guard patterns
- **`description`**: human-readable scenario

The pattern is a decision table. Each test method is a row. The contract is an upfront declaration of what the test proves. The integ-tester implements it. The final-validator audits it.

## How It Flows Through the Loop

Three touchpoints.

**The engineer writes the contract before the loop runs.** Either before the spec begins or alongside the `.INTEG` tasks in `tasks.md`. The engineer decides the scenarios, assertion floor, and required fields based on the requirements. This is the hardest step and it stays a human judgment call. The contract encodes what "correct behavior observed" means, which is a requirements question, not a code question.

**The integ-tester agent implements to the contract.** When the loop routes a `.INTEG` task to the integ-tester, the agent checks for `.integ-test-contract.json`. If it exists, the agent switches from creative-writer mode to contract-implementer mode. The prompt says, paraphrased:

> You are NOT a creative test writer for this task. You are a contract implementer. For each contract row in scope for this INTEG task, create a `@Test` method with the exact method name, at least `minAssertions` assertions, coverage for every field in `mustAssertFields`, and no early-return guards. You may add adversarial tests on top. You may not omit contract rows.

The agent can add tests beyond the contract (race conditions, adversarial scenarios it spots in the requirements), but it cannot skip a contract row or rename one.

**The final-validator audits the contract at FINAL time.** The validator runs after all `.INTEG` tasks are checked. When the contract file is present, the validator runs a six-check audit per row:

1. **Exists**: the method named in the contract is declared in a test file under `sourceRoot`
2. **Annotated**: the method has `@Test`
3. **Assertion count**: the method body contains at least `minAssertions` calls matching the `assertionPatterns`
4. **Required fields**: every field in `mustAssertFields` is referenced in the method's assertions
5. **No early return**: when `forbidEarlyReturn: true`, the method body has no `return;` above the final assertion
6. **Runtime status**: when `mustPass: true` and JUnit XML results are available, the test is present in results and neither skipped nor failed

Each row scores PASS only if all six checks succeed. FAIL on any failing check, with evidence cited as file and line number. UNKNOWN if the method isn't found in source at all.

The validator emits a fourth table in `validation-report.md`:

```
| Contract ID | Test method               | Exists | @Test | Asserts (>=min) | Fields          | No early return | Runtime | Status | Evidence             |
|-------------|---------------------------|--------|-------|-----------------|-----------------|-----------------|---------|--------|----------------------|
| C-01        | FooIT.testBackfillResume   | YES    | YES   | 4/3             | all             | YES             | PASS    | PASS   | FooIT.java:42        |
| C-07        | BarIT.testQueryTruncation  | YES    | YES   | 2/6             | missing errorCode | YES           | n/a     | FAIL   | BarIT.java:88        |
```

The final gate counts FAIL and UNKNOWN rows in that table alongside the existing three tables. Gaps block COMPLETE and produce COMPLETE-WITH-GAPS for manual triage.

## Why the Contract Is Separate From the Agent Prompt

The agent prompts are shared across every spec in the repo. Hardcoding "test X must assert field Y" in a prompt would contaminate every unrelated spec. The contract file is spec-local, so the universal prompt stays lean and each spec brings its own rules.

A second benefit: the contract is machine-readable. The final-validator's six checks can run deterministically against the JSON. A future shell gate could skip the LLM call entirely for most of the audit.

## Why We Haven't Built a Shell Gate Yet

The same six checks look shell-scriptable and we considered it. We decided to wait because:

Five of the six checks are easy (grep, regex, count). The field-reference check is hard. `assertThat(response).hasFieldOrPropertyWithValue("details.errorCode", ...)` versus `assertEquals(expected.getDetails().getErrorCode(), actual.getDetails().getErrorCode())` versus `var e = actual.getDetails(); assertEquals("X", e.getErrorCode())` — the LLM matches these intentionally; a substring grep matches or mis-matches depending on the code style. False negatives on this one check undermine the whole gate.

We have one spec with a contract so far. A shell gate is worth building once the pattern repeats. Two more contracts and we'd have concrete data on which match patterns actually show up.

The existing validator plus final gate already produce COMPLETE-WITH-GAPS correctly for contract violations. The shell gate would save LLM tokens, not add a capability.

## What a Contract Row Feels Like to Write

Start from the requirement. If the requirement says "dry-run SETUP_CONFIG returns success=true, dryRun=true without mutating any resource," your row is:

```json
"OrderApiIntegrationTest.testDryRun_SetupConfig": {
  "task": "29.INTEG",
  "mustPass": true,
  "minAssertions": 2,
  "mustAssertFields": ["success", "dryRun"],
  "forbidEarlyReturn": true,
  "description": "SETUP_CONFIG with dryRun=true must return {success: true, dryRun: true} without side effects"
}
```

Two assertions minimum: one for `success`, one for `dryRun`. You might add a third that reads the database record and confirms nothing changed. The integ-tester will do that if the requirement makes it obvious, but the contract enforces the minimum floor so the agent can't stop at `assertTrue(response.isSuccess())` and call it done.

Three heuristics that have worked:

**One assertion per field you care about.** If the requirement names three output fields, `minAssertions` is at least three.

**If there's a side effect, name it.** "Must insert a row in table X" becomes an extra assertion pulling the row back out.

**`mustPass: true` unless there's a reason not to.** `mustPass: false` is for tests that are only expected to compile right now, for example against a service that isn't deployed until a later task.

If a spec doesn't need this level of rigor, skip the contract file. The integ-tester falls back to its default creative-writer mode. The final-validator's three existing tables still apply.

---

*This post is part of a series on the multi-agent Ralph loop. See [the architecture](https://ranjithkannan.com/2026/04/10/multi-agent-tdd-loop/) and [the Final Validator](https://ranjithkannan.com/2026/04/23/final-validator-ralph-loop/).*
