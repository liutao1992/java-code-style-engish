---
name: backend-code-review
description: Review Java, Spring Boot, MyBatis, MyBatis-Plus, Rabbit-SQL, and PostgreSQL backend changes for correctness, business rules, layering, SOLID, APIs, databases, transactions, concurrency, security, and maintainability. Report only evidence-backed findings. Pure review is read-only by default. Executable verification belongs to review-and-test.
---

# Backend Code Review

Evaluate whether the specified backend changes are correct, preserve the target project's contracts, and comply with applicable team standards.

This Skill owns:

```text
review scope
review workflow
evidence requirements
finding severity
review output
```

It does **not** own the executable verification pipeline. Running tests, compilation, lint/static analysis, architecture checks, and builds belongs to the sibling `review-and-test` Skill.

Detailed backend standards remain centralized under `../java-spring-backend/references`. Use [routing.md](../java-spring-backend/references/routing.md) as the single routing index rather than maintaining another domain routing table here.

---

## 1. Boundaries and Permissions

- Read the applicable `AGENTS.md` in the target project first.
- Pure review is read-only by default. Do not automatically fix, format, rename, install dependencies, update snapshots, publish comments, or rewrite Git history.
- When invoked as self-review inside an already-authorized development task, permission to fix comes from that original task and remains limited to its scope.
- Read detailed standards directly from `../java-spring-backend/references`; do not execute the development Skill's implementation workflow.
- Existing tests and prior verification results may be used as evidence when available, but executable verification itself belongs to `review-and-test`.
- If required project context or standards are unavailable, state the evidence gap instead of guessing.

---

## 2. Determine Review Scope

Prefer the scope explicitly supplied by the user:

```text
file
directory
commit
commit range
PR
branch diff
```

If the user only says "review the current changes", cover all identifiable changes in the current workspace:

```text
unstaged
+
staged
+
relevant untracked files
```

Inspect workspace status first, then staged and unstaged diffs separately. Do not run only one `git diff` and assume the scope is complete.

When no scope is specified:

- do not review the entire repository by default;
- do not invent a remote baseline;
- do not mix unrelated historical debt into current findings.

If a comparison baseline cannot be determined, state exactly what was reviewed.

---

## 3. Review Workflow

1. **Confirm scope.** Record the actual files, commits, or workspace state being reviewed.
2. **Read the complete change.** Inspect affected methods, callers, models, SQL, configuration, and nearby contracts rather than isolated patch lines.
3. **Build project context.** Search similar implementations, stable project conventions, central abstractions, relevant tests, and recent history when available.
4. **Select standards.** Use [routing.md](../java-spring-backend/references/routing.md) to load only the references required by the current change. Confirm the actual framework before applying framework-specific rules.
5. **Trace data and control flow.** Follow input through logical boundaries to persistence or external calls and back to output. Identify changed invariants and contracts.
6. **Separate introduced problems from existing debt.** Prioritize defects introduced or materially worsened by the reviewed change.
7. **Use evidence, not speculation.** A finding must have a concrete trigger, broken contract, incorrect dependency, or applicable rule with supporting project context.
8. **Form the result.** Merge duplicate root causes, order findings by impact, and state necessary evidence gaps.

---

## 4. Standards Selection

Do not maintain a second routing matrix in this Skill. Use:

- [Backend Standards Routing Index](../java-spring-backend/references/routing.md)

Examples:

```text
Controller HTTP-contract change
→ API + necessary Spring

New VO / Request / Query placement
→ Layering + Java

Repeated invariant or questionable domain-object extraction
→ Business rules + Layering + Java

MyBatis-Plus Wrapper / Mapper XML
→ MyBatis + SQL when SQL semantics change

@XQLMapper / Baki / .xql
→ Rabbit-SQL + SQL when SQL semantics change

TransactionTemplate / @Transactional behavior
→ Transactions + necessary Spring

CompletableFuture with database work
→ Concurrency + Transactions

Test code itself appears incorrect or misleading
→ Testing; executable coverage and command execution still belong to review-and-test
```

---

## 5. Conditions for a Valid Finding

Report an issue only when supported by concrete evidence. At least one of the following should apply:

- reproducible or logically unavoidable incorrect behavior;
- a clear call-chain or data-flow failure condition;
- an API, database, permission, transaction, or business contract is broken;
- the change bypasses an established target-project mechanism without justification;
- an applicable reference rule is violated and there is no project-specific exception;
- a security, data-integrity, or concurrency risk has a concrete trigger.

Do not create findings based only on:

```text
personal preference
"more elegant"
possible future expansion
an architecture school
code that merely feels uncomfortable
```

If a key premise cannot be established, mark it as `needs confirmation` rather than presenting speculation as a defect.

---

## 6. Project Contracts Take Priority

Before creating a standards-based finding, inspect the target project's established contracts for the affected area, such as:

```text
API / serialization
model and Package conventions
module structure
persistence framework
Spring MVC / validation
exceptions and logging
collection Null behavior
transaction / rollback
pagination
database naming / migration
security / tenant / data scope
```

Do not demand migration of stable historical code merely because this Skill Pack's default differs.

---

## 7. False-Positive Guardrails

### Duplicate validation

Report duplicate structural validation only after confirming that the same semantic constraint is already guaranteed by a trusted inbound boundary and no other unvalidated entry path requires the lower-layer check.

### Collection Null defenses

Report meaningless collection-null fallbacks only when the source contract is explicitly non-null and no proxy, plugin, legacy implementation, or external compatibility requirement changes that contract.

### SOLID / architecture

Identify a concrete responsibility, dependency, substitutability, extension, or contract problem. Do not report generic advice such as `Follow SOLID`, `Extract an interface`, or `Add Strategy / Factory` without a triggering condition.

### Performance

Do not present "might be faster" as a confirmed defect without data, plan evidence, capacity information, or a structurally obvious problem such as N+1 or an unbounded query.

### Transactions

Do not report a transaction issue merely because a write exists, two Mappers appear, or `@Transactional` is absent. Identify the operations that must succeed or roll back together and how current behavior violates that requirement.

### Persistence framework

`Mapper` / `DAO` naming does not prove MyBatis. Confirm dependencies, imports, annotations, XML/XQL resources, `BaseMapper`, `@XQLMapper`, Baki, or framework APIs before applying framework-specific rules.

### Clean Architecture / anemic models

Do not require UseCase / Entity / Repository / Command / Result layers mechanically. Report only concrete invariant-bypass, dependency-direction, semantic-drift, or maintenance risks.

---

## 8. Severity

### P0 — Blocking / Catastrophic

Broad data destruction, severe authentication bypass, major sensitive-data exposure, or guaranteed outage of a core capability.

### P1 — High Priority

Major functional error, broken authorization/data-consistency boundary, released critical API incompatibility, or high-probability concurrency / transaction failure.

### P2 — Medium Priority

Functional error under specific conditions, important maintainability misuse risk, or clear violation of a stable project boundary.

### P3 — Low Priority

Local clarity or consistency issue with real but limited maintenance cost.

Severity follows actual impact and trigger probability, not the name of the violated rule.

---

## 9. Finding Format

Each finding should contain:

```text
[P1/P2/P3] Concise title
Location: file:line or smallest useful range
Problem: what is specifically wrong
Trigger: input / call path / state that exposes it
Impact: user / data / contract consequence
Evidence: project contract, code path, existing result, or applicable reference
Recommendation: minimal corrective direction
```

Prefer a verifiable causal chain:

```text
change
→ trigger
→ incorrect behavior
→ impact
```

---

## 10. Review Output

When findings exist:

1. list findings first by severity;
2. list only necessary `needs confirmation` items;
3. state the actual review scope and evidence limitations.

When there are no findings, state:

> No defects with sufficient supporting evidence were found within the actual review scope.

Do not claim tests, lint, builds, or other executable verification passed unless those results already exist and were explicitly inspected. For a current verification run, use `review-and-test`.

Final principle:

> This Skill proves why a completed implementation is or is not correct; references define the detailed standards; `review-and-test` independently verifies behavior with executable checks.
