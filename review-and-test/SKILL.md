---
name: review-and-test
description: Validate completed changes with repository-defined tests, compilation/type checking, lint/static analysis, architecture checks, builds, and explicit edge-case reasoning. Use the centralized testing standard for test quality and scenario design. Report PASS/FAIL/NOT RUN honestly.
---

# Review and Test

Validate completed changes with executable evidence before commit, PR, or merge.

This Skill owns:

```text
changed-behavior understanding
verification-plan construction
repository-check discovery
targeted execution
broader execution when appropriate
failure investigation
PASS / FAIL / NOT RUN reporting
```

It does **not** own architecture or implementation review; that belongs to `backend-code-review`. It also does not maintain its own test-design standard. Test quality, scenario design, mocks, assertions, fixtures, integration-test guidance, and related testing knowledge live in [testing.md](../java-spring-backend/references/coding/testing.md).

---

## 1. Boundaries and Permissions

- Read the applicable `AGENTS.md` first.
- Prefer repository-defined commands, CI configuration, and existing test infrastructure.
- Load [testing.md](../java-spring-backend/references/coding/testing.md) when test quality, missing coverage, or new/changed tests must be evaluated.
- A request only to review and test does not authorize unrelated refactoring, dependency upgrades, snapshot rewrites, or weakening checks.
- If invoked inside an already-authorized implementation/fix task, fixes may be made only within that task's scope.
- Do not rewrite Git history; use `rework-commits` only when explicitly requested.
- Never report a command as passed unless it was actually executed successfully against the current relevant state.

---

## 2. Verification Workflow

1. **Understand changed behavior.** Use the requirement, diff, nearby implementation, contracts, and relevant tests to identify what changed and what could regress.
2. **Discover verification infrastructure.** Inspect repository-defined build files, scripts, CI workflows, and documentation to identify canonical tests, compilation/type checks, lint/static analysis, architecture checks, and build commands.
3. **Load testing standards when needed.** Use [testing.md](../java-spring-backend/references/coding/testing.md) for test design, test quality, regression coverage, mocks, assertions, fixtures, integration/database tests, concurrency tests, and transaction tests. Do not restate those rules here.
4. **Build a verification plan.** Select the smallest checks that directly exercise the changed behavior, then identify broader checks required for integration confidence.
5. **Run targeted checks first.** Execute the closest relevant unit, integration, compile, or static check. Investigate failures before expanding the scope.
6. **Run broader checks when appropriate.** After targeted checks pass, execute the repository's normal affected-module or project-wide checks when practical and relevant.
7. **Investigate failures.** Determine whether each failure is caused by the current change, pre-existing, environmental, or unrelated. Fix only when authorized and within scope, then rerun the smallest affected checks and any broader checks invalidated by the fix.
8. **Reason about unexecuted risks.** For important cases that cannot be verified automatically, record the missing evidence explicitly instead of assuming correctness.
9. **Report evidence.** Distinguish PASS, FAIL, and NOT RUN and give the final readiness result.

---

## 3. Discover Repository Checks

Prefer the project's own commands and inspect relevant files such as:

```text
AGENTS.md
README
pom.xml
build.gradle / build.gradle.kts
package.json
Makefile
Taskfile.yml
pyproject.toml
Cargo.toml
go.mod
CI workflow files
```

Typical independent verification signals include:

```text
unit tests
integration tests
end-to-end tests
compilation / type checking
lint
static analysis
architecture checks
build / packaging
```

Do not invent a parallel command when the repository already defines the canonical check.

---

## 4. Verification Scope

Start with the narrowest high-signal checks related to the change, for example:

```text
specific test method / class
specific test package
affected module tests
targeted integration test
targeted compilation
targeted lint / static analysis
```

Then expand according to dependency and integration risk.

Examples:

```text
business-rule change
→ focused behavioral/regression tests
→ affected module tests

Mapper / SQL change
→ relevant database integration test when available
→ affected module tests / build

transaction or concurrency change
→ targeted transaction/concurrency verification
→ affected integration suite

public API contract change
→ relevant API test
→ affected module checks
```

The detailed criteria for designing these tests belong to [testing.md](../java-spring-backend/references/coding/testing.md).

---

## 5. Failure Investigation

For each failure:

1. identify the failing command and smallest relevant error;
2. trace the root cause rather than treating the first visible stack frame as sufficient evidence;
3. determine whether the current change caused or exposed it;
4. fix only when the task authorizes the fix and it belongs to scope;
5. rerun the smallest relevant check;
6. rerun broader checks whose previous result is no longer valid after the fix.

Do not make a pipeline green by:

```text
weakening assertions
disabling tests
skipping required checks
suppressing compiler errors
silencing lint without justification
catching and hiding exceptions
changing expected output to match a bug
```

If a failure is pre-existing or environmental, report the evidence for that conclusion.

---

## 6. Reuse Existing Evidence Carefully

A prior check result may be reused only when:

- it was actually executed against the relevant code state;
- subsequent changes could not affect the checked behavior;
- the command and result are known rather than inferred.

Do not rerun checks mechanically when their evidence remains valid, but do not claim an old result after affected code changed.

---

## 7. Output Format

Start with:

```text
Verification Summary

Changed behavior:
Verification strategy:
```

Then list executed or important omitted checks:

```text
PASS     <command or check>
FAIL     <command or check>
NOT RUN  <important check and reason>
```

Add only applicable sections:

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

Use:

- `READY` when the necessary verification passed and no material unverified risk remains;
- `READY WITH RISKS` when executed checks pass but meaningful verification remains unavailable or intentionally omitted;
- `NOT READY` when a relevant check fails or a material correctness issue remains unresolved.

Never say "all checks passed" when important checks were not run.

Final principle:

> This Skill executes and reports verification; `testing.md` defines how tests should be designed and judged; `backend-code-review` evaluates implementation quality and architecture.
