---
name: review-and-test
description: Validate completed code changes with targeted and broader tests, type checking, lint/static analysis, builds, and explicit edge-case reasoning. Review test quality, identify missing verification, investigate failures, and report PASS/FAIL/NOT RUN honestly. Use after implementation or code review and before commit, PR, or merge.
---

# Review and Test

Validate that completed changes behave as intended and are ready for integration.

This Skill owns the executable verification loop:

```text
understand changed behavior
→ discover repository checks
→ assess test quality / gaps
→ run targeted checks
→ run broader checks when appropriate
→ investigate failures
→ report verified and unverified behavior
```

Architecture and implementation review belongs to `backend-code-review`. Git-history restructuring belongs to `rework-commits`.

Passing tests are evidence, not proof. A reliable result combines multiple independent signals when the project provides them.

---

## 1. Boundaries and Permissions

- Read the applicable `AGENTS.md` first.
- Prefer repository-defined commands and existing test infrastructure.
- A request only to "review and test" does not automatically authorize unrelated refactoring, dependency upgrades, snapshot rewrites, or weakening checks.
- If this Skill is invoked inside an already-authorized implementation/fix task, fixes may be made only within that task's scope.
- Do not rewrite Git history; use `rework-commits` only when explicitly requested.
- Never report a command as passed unless it was actually executed successfully in the current relevant state.

---

## 2. Understand the Changed Behavior

Before running commands, determine:

- what behavior was added or changed;
- what bug or requirement the change addresses;
- what existing behavior could regress;
- which components and contracts are affected.

Use the diff, requirements, nearby implementation, existing tests, and relevant project documentation.

Build a concrete verification checklist before execution. Do not start by blindly running the entire test suite.

---

## 3. Discover Verification Infrastructure

Prefer the project's own commands and CI configuration. Inspect relevant files such as:

```text
AGENTS.md
README
package.json
pom.xml
build.gradle / build.gradle.kts
Makefile
Taskfile.yml
pyproject.toml
Cargo.toml
go.mod
CI workflow files
```

Identify available signals:

```text
unit tests
integration tests
end-to-end tests
type checking
lint
static analysis
architecture checks
build / packaging
```

For Java backend projects in this Skill Pack, load [Testing](../java-spring-backend/references/coding/testing.md) when project-specific testing rules are relevant.

Do not invent a parallel command when the repository already defines the canonical check.

---

## 4. Evaluate Existing Tests

Do not equate "tests exist" with "behavior is verified".

For relevant tests, ask:

- Does the test assert observable behavior or merely implementation details?
- Does it verify the actual requirement?
- Could an incorrect implementation still pass?
- Is only the happy path covered?
- Are important branches hidden behind mocks?
- Does the expected result encode the same mistake as the implementation?
- Is the fixture realistic enough to exercise the contract?

Treat agent-generated tests with the same skepticism as production code.

Watch for:

```text
weak assertions
tests that cannot fail meaningfully
excessive mocking
mock behavior that contradicts the real dependency
duplicated tests
unrealistic fixtures
missing negative cases
assertions copied directly from implementation logic
```

Do not modify production behavior merely to satisfy an incorrect test.

---

## 5. Identify Missing Verification

For every meaningful behavior change, consider the smallest useful test level.

### Happy path

Verify the expected normal result.

### Boundaries

Check important values around transitions:

```text
limit - 1
limit
limit + 1
```

Also consider empty, one-item, many-item, duplicate, minimum, maximum, and large-input behavior when relevant.

### Invalid input

Verify malformed, missing, unsupported, or internally inconsistent input.

### Error paths

Verify expected behavior for dependency errors, timeouts, parsing failures, permission failures, and partial failures when relevant.

### Regression tests

For a bug fix, prefer a test that fails without the fix and passes with it.

### Integration tests

Use when correctness depends on component interaction, for example:

```text
service + database
API + authentication
repository + transaction
message producer + consumer contract
```

### End-to-end tests

Use only when the behavior cannot be meaningfully validated at a lower level.

Do not add every test type mechanically. Choose the lowest level that gives meaningful confidence.

---

## 6. Run Targeted Checks First

Start close to the changed code:

```text
specific unit test
specific test class / package
affected module tests
targeted integration test
targeted type check / compile
targeted lint / static analysis
```

The objective is fast, high-signal feedback.

If targeted checks fail, investigate before spending time on broad verification.

---

## 7. Run Broader Verification

After targeted checks pass, run the repository's normal broader checks when practical and relevant.

Typical stack:

```text
tests
+
type checking / compilation
+
lint / static analysis / architecture checks
+
build / packaging
```

One signal does not replace the others. A successful compile does not prove behavior; passing tests do not prove security or all edge cases; lint does not prove correctness.

Do not rerun checks that already passed and are unaffected by subsequent changes.

---

## 8. Investigate Failures

For each failure:

1. determine whether it is caused by the current change;
2. identify the root cause rather than treating the first error line as the cause;
3. fix it only when current task permissions allow and it belongs to the task;
4. rerun the smallest relevant check;
5. rerun broader affected checks after the fix when necessary.

Do not make a failing pipeline green by:

```text
weakening assertions
disabling tests
skipping checks
disabling lint rules without justification
suppressing compiler errors
catching and hiding exceptions
changing expected output to match a bug
```

If a failure is pre-existing or unrelated, report evidence for that conclusion.

---

## 9. Explicit Edge-Case Reasoning

Automated checks may still miss important cases. Reason about relevant scenarios explicitly:

```text
null / absent values
empty values
zero / negative values
duplicates
minimum / maximum
boundary - 1 / boundary / boundary + 1
missing records
repeated operations / idempotency
retry / timeout
partial failure
concurrency / ordering
unexpected state
```

If an important case cannot be executed automatically, classify it as manual verification or residual risk rather than silently assuming it is safe.

---

## 10. Verification Speed

Slow feedback directly reduces the effectiveness of agent-assisted development.

When verification is materially slow, report the bottleneck separately rather than silently avoiding checks. Potential causes include:

```text
oversized test suites
serial tests that could safely run in parallel
redundant CI jobs
unnecessary dependency work
slow type checking
repeated uncached build work
```

Do not perform unrelated build-system optimization during a normal verification task unless requested.

---

## 11. Output Format

Start with:

```text
Verification Summary

Changed behavior:
Verification strategy:
```

Then list executed checks:

```text
PASS     <command or check>
FAIL     <command or check>
NOT RUN  <important check and reason>
```

Then provide only meaningful sections that apply:

```text
Testing Gaps
Residual Risks
Failure Analysis
```

Finish with:

```text
Verification Result:

READY
READY WITH RISKS
NOT READY
```

Explain the result briefly.

Never say "all checks passed" when important checks were not run. Never infer a successful check from unchanged-looking code.

Final principle:

> Verification should create independent evidence that the change works; it should not merely confirm the implementation's own assumptions.
