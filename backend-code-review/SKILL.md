---
name: backend-code-review
description: Review Java, Spring Boot, MyBatis, MyBatis-Plus, Rabbit-SQL, and PostgreSQL backend changes for correctness, business rules, layering, SOLID, APIs, databases, transactions, concurrency, security, and maintainability. Load team standards on demand and report only evidence-backed findings. Pure review is read-only by default. Verification execution belongs to review-and-test.
---

# Backend Code Review

Evaluate whether the specified backend changes are correct, preserve the target project's contracts, and comply with applicable team standards.

This Skill owns:

```text
review scope
review workflow
standards routing
evidence requirements
severity
review output
```

It does **not** own the executable verification pipeline. Running and reporting tests, type checks, lint/static analysis, and builds belongs to the sibling `review-and-test` Skill.

Detailed Java, business-rule, layering, API, Spring, MyBatis, Rabbit-SQL, SQL, database, transaction, concurrency, and testing knowledge remains centralized under `java-spring-backend/references`. Load only the references needed for the current change; do not duplicate those standards here.

---

## 1. Boundaries and Permissions

- Read the applicable `AGENTS.md` in the target project first.
- Pure review is read-only by default. Do not automatically fix, format, rename, install dependencies, update snapshots, publish comments, or rewrite Git history.
- When invoked as self-review inside an already-authorized development task, permission to fix comes from that original task and remains limited to its scope.
- Keep this Skill as a sibling of `java-spring-backend`; read its references directly rather than executing the development Skill's implementation workflow.
- Use existing tests and verification results as evidence when available, but do not own or duplicate the full verification run. Use `review-and-test` for that stage.
- If required references are missing, state that standards verification is limited.

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

Inspect workspace status first, then read staged and unstaged diffs separately. Do not run only one `git diff` and assume the scope is complete.

When no scope is specified:

- do not review the entire repository by default;
- do not invent a remote baseline;
- do not mix unrelated historical issues into current findings.

If a comparison baseline cannot be determined, state exactly what was reviewed.

---

## 3. Review Workflow

1. **Confirm scope.** Record the actual files / commits / workspace state being reviewed.
2. **Read the complete change.** Inspect affected methods, callers, models, SQL, configuration, and nearby contracts rather than isolated patch lines.
3. **Build project context.** Search for similar implementations, stable project conventions, central abstractions, and relevant recent history.
4. **Select standards.** Load only the references required by the routing table. Identify the actual persistence framework before applying persistence rules.
5. **Trace data and control flow.** Follow input through layers to persistence / external calls / output and identify changed invariants or contracts.
6. **Separate introduced problems from existing debt.** Prioritize defects introduced or materially worsened by the current change.
7. **Use evidence, not speculation.** Existing tests and prior validation may support a finding, but executable verification itself belongs to `review-and-test`.
8. **Form the result.** Merge duplicate root causes, order by impact, and state any evidence gaps.

---

## 4. Standards Routing

| Area involved | Load | Primary checks |
| --- | --- | --- |
| Java implementation | [Java](../java-spring-backend/references/coding/java.md) | naming, class design, constants, Enum, magic values, POJO defaults, parameters, Null, collections, exceptions, logging |
| Physical project directory / module | [Project structure](../java-spring-backend/references/architecture/project-structure.md) | business-module location, common directories, physical organization |
| Layering / models / responsibility / SOLID | [Layering](../java-spring-backend/references/architecture/layering.md) | responsibilities, dependencies, model boundaries, cross-module calls, overdesign |
| Business rules / Use Case / Entity concepts | [Business rules](../java-spring-backend/references/architecture/business-rules.md) | invariants, application flows, behavioral objects, persistence boundaries |
| Spring Framework | [Spring](../java-spring-backend/references/coding/spring.md) | MVC, Validation, DI, Bean, Proxy, Advice |
| HTTP API | [API](../java-spring-backend/references/api/api-design.md) | URL, Method, Request/VO, response, errors, compatibility, pagination, idempotency |
| Exceptions across layers | [Error handling](../java-spring-backend/references/architecture/error-handling.md) | translation, cause, logging ownership, external leakage |
| MyBatis / MyBatis-Plus | [MyBatis / MyBatis-Plus](../java-spring-backend/references/coding/mybatis.md) | Mapper/DAO, BaseMapper, Wrapper, XML, binding, ResultMap, TypeHandler, collection contracts |
| Rabbit-SQL | [Rabbit-SQL](../java-spring-backend/references/coding/rabbit-sql.md) | `@XQLMapper`, Baki, XQL mapping, binding, `${}`, pagination, Stream, Batch, Spring transactions |
| SQL | [SQL](../java-spring-backend/references/database/sql.md) | correctness, scope, injection, safety, PostgreSQL, performance evidence |
| Database schema | [Database design](../java-spring-backend/references/database/database-design.md) | types, Null, constraints, indexes, Migration, compatibility |
| Transactions / locks / consistency | [Transactions](../java-spring-backend/references/architecture/transactions.md) | necessity, scope, rollback, propagation, isolation, races |
| Concurrency / async | [Concurrency](../java-spring-backend/references/architecture/concurrency.md) | thread pools, context, exceptions, shared state, resource capacity |
| Test code encountered during review | [Testing](../java-spring-backend/references/coding/testing.md) | whether a test itself is misleading or encodes incorrect behavior; full coverage/verification assessment belongs to `review-and-test` |
| Permissions / tenant / data scope / security | target project's existing security standards and implementation | authorization bypass, isolation, data leakage, credential safety |

Load only the necessary union for multi-domain changes.

Examples:

```text
Controller URL change
→ API + necessary Spring

New VO / Request / Query package
→ Layering + Java

MyBatis-Plus Wrapper / Mapper XML
→ MyBatis + SQL when SQL semantics change

@XQLMapper / Baki / .xql
→ Rabbit-SQL + SQL when actual SQL changes

TransactionTemplate / @Transactional semantics
→ Transactions + necessary Spring

CompletableFuture with database work
→ Concurrency + Transactions
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

Before creating a standards-based finding, inspect the target project's established contracts for:

```text
API / serialization
model and package conventions
module structure
persistence framework
Spring MVC / validation
exceptions and logging
collection Null behavior
transaction / rollback
pagination
database naming / migration
```

Do not demand migration of stable historical code merely because this Skill Pack's default differs.

---

## 7. False-Positive Guardrails

### Duplicate validation

Report duplicate structural validation only after confirming that the same semantic constraint is already guaranteed by a trusted inbound boundary and that no other unvalidated entry path requires the lower-layer check.

### Collection Null defenses

Report meaningless collection-null fallbacks only when the source contract is explicitly non-null and no proxy, plugin, legacy implementation, or external compatibility requirement changes that contract.

### SOLID / architecture

Identify a concrete responsibility, dependency, substitutability, extension, or contract problem. Do not report generic advice such as `Follow SOLID`, `Extract an interface`, or `Add Strategy / Factory` without a triggering condition.

### Performance

Do not present "might be faster" as a confirmed defect without data, plan evidence, capacity information, or a structurally obvious problem such as N+1 or an unbounded query.

### Transactions

Do not report a transaction issue merely because a write exists, two Mappers appear, or `@Transactional` is absent. Identify the operations that must succeed or roll back together and how current behavior violates that requirement.

### Persistence framework

`Mapper` / `DAO` naming does not prove MyBatis. Confirm dependencies, imports, annotations, XML/XQL resources, `BaseMapper`, `@XQLMapper`, Baki, or framework APIs before applying rules.

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
Evidence: project contract, code path, existing test/result, or applicable reference
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

Do not claim tests, lint, builds, or other verification passed unless those results were already available and explicitly inspected. For executable verification, use `review-and-test`.

Final principle:

> The review Skill must prove why something is a problem; domain references define the detailed rule; `review-and-test` proves the completed change with executable checks.
