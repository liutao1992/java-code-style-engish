---
name: backend-code-review
description: Review Java, Spring Boot, MyBatis, MyBatis-Plus, Rabbit-SQL, and PostgreSQL backend code or changes. Load team standards on demand; check correctness, business rules, layering, APIs, databases, transactions, concurrency, security, and tests; and report findings with precise locations and evidence. Pure review is read-only by default.
---

# Backend Code Review

Evaluate whether the specified backend changes are correct, whether they break the target project's contracts, and whether they violate applicable team standards.

This Skill is responsible only for:

```text
review scope
review workflow
standards routing
evidence requirements
severity
output format
```

Detailed Java, business-rule, layering, API, Spring, MyBatis, Rabbit-SQL, SQL, database, transaction, concurrency, and testing rules are read directly from `java-spring-backend/references`. Do not maintain a second copy of those standards in this file.

---

## 1. Boundaries and Permissions

- Read the applicable `AGENTS.md` in the target project first.
- Pure review is read-only by default. Do not automatically fix, format, rename, install dependencies, update snapshots, publish comments, or create a separate agent.
- When this Skill is invoked for self-review from a development task, permission to fix comes from the original development task and is limited to that task's scope.
- Keep this Skill as a sibling of `java-spring-backend`; read its references directly and do not execute the development Skill's implementation workflow.
- If references are missing, explicitly state that standards verification is limited. Do not claim that full team-standard validation was completed.

---

## 2. Determine the Review Scope

Prefer the scope explicitly specified by the user:

```text
file
directory
commit
commit range
PR
branch diff
```

If the user only says "review the current changes" without specifying a comparison target, cover all identifiable changes in the current workspace:

```text
unstaged changes
+
staged changes
+
untracked files related to the current task
```

When using Git in practice, first inspect the workspace status, then read both workspace and staged diffs separately. Do not run only one `git diff` and assume the scope is complete.

When no scope is specified:

* do not review the entire repository by default;
* do not guess a remote baseline or comparison branch;
* do not mix unrelated historical issues into the current findings.

If the comparison baseline cannot be determined, state what was actually reviewed.

---

## 3. Review Workflow

1. **Confirm scope.** Make clear what is being reviewed and whether staged / unstaged / untracked changes are included.
2. **Read the complete change.** Do not inspect only isolated patch lines; read affected methods, callers, models, SQL, configuration, and related tests.
3. **Build project context.** Search for similar implementations, existing contracts, build configuration, and available recent Git history. If they do not exist, state that honestly.
4. **Select standards.** Load only the references actually required by the routing table. Do not recursively load every standard. When encountering a Mapper / DAO, first identify the actual persistence framework rather than assuming MyBatis from the name.
5. **Validate along the data flow.** Confirm where input originates, which layers it passes through, and what state or contract it ultimately affects.
6. **Separate new and existing problems.** Prioritize problems introduced or worsened by the current change. Do not mix unrelated pre-existing issues into the findings.
7. **Run necessary validation.** Run existing tests, static checks, and architecture checks that are relevant to the scope and do not modify code or shared environments without authorization.
8. **Form the result.** Merge findings with the same root cause, sort by severity, and describe the actual validation scope.

---

## 4. Standards Routing

| Area involved | Load | Primary checks |
| --- | --- | --- |
| Java implementation | [Java](../java-spring-backend/references/coding/java.md) | naming, class design, constants, Enum, magic values, POJO defaults, parameters, Null, collections, exception implementation, logging, formatting |
| Physical project directory / module | [Project structure](../java-spring-backend/references/architecture/project-structure.md) | business-module location, common directories, physical organization |
| Layering / models / responsibility Packages / SOLID | [Layering](../java-spring-backend/references/architecture/layering.md) | responsibilities, dependencies, model boundaries, cross-module calls, overdesign |
| Business-rule layers / Use Case / Entity concepts | [Business rules](../java-spring-backend/references/architecture/business-rules.md) | core invariants, application flows, behavioral business objects, persistence DO boundaries, whether input/output models need isolation |
| Spring Framework | [Spring](../java-spring-backend/references/coding/spring.md) | MVC, Validation, DI, Bean, Proxy, Advice |
| HTTP API | [API](../java-spring-backend/references/api/api-design.md) | URL, Method, Request/VO, response, errors, compatibility, pagination, idempotency |
| Exceptions across layers | [Error handling](../java-spring-backend/references/architecture/error-handling.md) | translation, cause, logging ownership, external leakage |
| MyBatis / MyBatis-Plus | [MyBatis / MyBatis-Plus](../java-spring-backend/references/coding/mybatis.md) | MyBatis Mapper/DAO, BaseMapper, Wrapper, Mapper XML, binding, ResultMap, TypeHandler, collection contracts |
| Rabbit-SQL | [Rabbit-SQL](../java-spring-backend/references/coding/rabbit-sql.md) | `@XQLMapper`, Baki, XQL registration/mapping, parameter binding, `${}`, dynamic XQL, pagination, Stream, Batch, Spring transaction integration |
| SQL | [SQL](../java-spring-backend/references/database/sql.md) | correctness, scope, injection, safety, PostgreSQL, performance evidence |
| Database schema | [Database design](../java-spring-backend/references/database/database-design.md) | types, Null, constraints, indexes, Migration, compatibility |
| Transactions / locks / consistency | [Transactions](../java-spring-backend/references/architecture/transactions.md) | necessity, scope, rollback, propagation, isolation, races |
| Concurrency / async | [Concurrency](../java-spring-backend/references/architecture/concurrency.md) | benefits, thread pools, context, exceptions, shared state, resource capacity |
| Bugs / behavior changes / tests | [Testing](../java-spring-backend/references/coding/testing.md) | regression protection, boundaries, assertions, integration validation |
| Permissions / tenant / data scope / security | Existing security standards, contracts, and implementation in the target project | authorization bypass, isolation, data leakage, credential security |

For multi-domain changes, load only the necessary union. For example:

```text
Controller URL change
→ API + necessary Spring

Mapping annotation only
→ Spring

New VO Package
→ Layering + Java

The same stable state rule is repeated as if + set in multiple Services / entry points
→ Business rules + Layering + Java

New Entity / UseCase / Repository / Command / Result structure
→ Business rules + Layering; confirm real responsibility value rather than reporting based only on an architecture school

Magic values / giant catch-all constants class / fixed value domain / POJO defaults
→ Java

Null fallback after MyBatis List<T>
→ MyBatis + Java

MyBatis-Plus Mapper does not extend BaseMapper, or Wrapper-based condition construction appears
→ MyBatis

Business status / type code hard-coded in Mapper XML
→ MyBatis; add SQL if SQL correctness is also being evaluated

@XQLMapper / Baki / .xql / xql-file-manager.yml change
→ Rabbit-SQL; add SQL when the actual SQL changes

Rabbit-SQL `${}` receives external input, dynamic XQL, pagination count, Stream resource lifecycle
→ Rabbit-SQL + necessary SQL / Java

Rabbit-SQL Spring transaction change
→ Rabbit-SQL + Transactions + necessary Spring

Local TransactionTemplate transaction
→ Transactions + Spring

CompletableFuture database operation
→ Concurrency + necessary Transactions
```

---

## 5. Conditions for a Valid Finding

Report only issues supported by clear evidence. At least one of the following must apply:

* reproducible incorrect behavior;
* a clear call-chain or data-flow risk;
* an existing API, database, permission, transaction, or business contract is broken;
* the current change bypasses a stable target-project implementation without justification;
* an explicit rule in an applicable reference is violated and there is no project-specific exception;
* a security, data-integrity, or concurrency risk has a concrete triggering condition.

Do not create a finding based only on:

```text
personal preference
"more elegant"
possible future expansion
an architecture school
code that simply feels uncomfortable
```

If a key contract is missing and the issue cannot be confirmed, mark it as "needs confirmation" or state that evidence is insufficient. Do not present speculation as a confirmed defect.

---

## 6. Project Contracts Take Priority

Default recommendations in references must not automatically override stable conventions already established in the target project.

Before creating a standards-based finding, check whether the current project already has explicit conventions for:

```text
API / serialization contracts
model and Package conventions
module structure
persistence framework and SQL resource organization
Spring MVC style
Validation boundaries
exception and logging systems
collection Null contracts
transaction / rollback conventions
pagination structures
database naming and migration mechanisms
```

If the current change simply continues a stable historical contract, do not create a finding merely because the Skill's default style differs.

Likewise, do not require opportunistic migration of unrelated historical code merely for "consistency."

---

## 7. Common Sources of False Positives

The following cases require confirmation of key premises before reporting them.

### 7.1 Duplicate Structural Validation

Read the detailed rules in `spring.md`.

Report "duplicate structural validation" only when all of the following are confirmed:

1. the current call path has already passed through a trusted structural-validation boundary;
2. the business-layer check has the same semantics as that structural constraint rather than being an independent business rule;
3. there is no other unvalidated entry point that requires the current method to enforce a shared input contract.

Do not mechanically report an issue merely because the same field has both Bean Validation and a handwritten check.

### 7.2 Collection Null Defenses

Read the general rules in `java.md`; for MyBatis sources also read `mybatis.md`.

Report meaningless Null defenses only when all of the following are confirmed:

1. the return value is a collection;
2. the current source explicitly guarantees non-null;
3. project-specific proxies / plugins / legacy implementations have not changed the contract;
4. the current check is not compatibility handling for a known nullable third-party source.

Do not generalize this into "collections must never be Null."

### 7.3 SOLID / Architecture Issues

Read the detailed rules in `layering.md`.

You must identify a specific responsibility, dependency, or contract risk. Do not report generic comments such as:

```text
Follow SOLID
Extract an interface
Add Strategy / Factory
```

without a triggering condition.

### 7.4 Performance Issues

Without real data, an execution plan, capacity information, or call-frequency evidence, do not present "might be faster" as a confirmed performance defect.

When the SQL structure clearly contains risks such as N+1, unbounded queries, or incorrect index assumptions, explain the trigger and the scope of evidence.

### 7.5 Transaction Issues

Read the detailed rules in `transactions.md`.

Do not report a transaction issue automatically just because there are two Mappers, there is a write, or `@Transactional` is absent. Identify which operations must succeed/rollback together and how the current implementation violates that consistency requirement.

### 7.6 Misidentifying the Persistence Framework

An interface named `Mapper` / `DAO` does not prove that it is MyBatis.

Before review, inspect:

```text
dependencies
imports / annotations
Mapper XML / .xql
BaseMapper / @XQLMapper
Baki / MyBatis APIs
```

Confirm the actual framework before loading the corresponding reference.

Do not report that a Rabbit-SQL `@XQLMapper` "does not extend MyBatis-Plus BaseMapper," and do not review MyBatis Mapper XML using XQL rules.

### 7.7 False Positives Around "Anemic Models" and Clean Architecture

Read the detailed rules in `business-rules.md`.

Do not report an architecture issue merely because:

```text
DO contains only fields
business checks live in Service
the project has no Entity / UseCase / Repository
Controller calls Service directly
```

Create a finding only when there is concrete evidence, for example:

* the same stable business invariant is implemented repeatedly across multiple use cases and semantic drift has already appeared;
* callers can bypass critical state constraints and directly modify state, so the business contract can be broken;
* a supposed core business object depends back on Spring, Mapper, Client, or external protocol types;
* multiple same-field models and pure forwarding classes were introduced merely to fit an architecture pattern and have created real maintenance cost or error risk.

Likewise, do not mechanically suggest:

```text
split every Service into UseCase classes
turn every DO into a rich Entity
wrap Mapper in another Repository layer
convert every Request into Command
add Result before every VO
```

Explain why the current rule is a stable core invariant or an application flow, and what real risk the proposed adjustment would eliminate.

---

## 8. Severity

### P0 — Blocking / Catastrophic

Examples:

* clearly causes broad data destruction;
* severe authentication bypass or sensitive-data exposure;
* a production core capability is guaranteed to become unavailable.

### P1 — High Priority

Examples:

* major functional error;
* data-consistency or authorization boundary is broken;
* a released critical API is clearly incompatible;
* high-probability concurrency error or incorrect transaction commit.

### P2 — Medium Priority

Examples:

* functional error under specific conditions;
* a maintainability issue has already created a real misuse risk;
* a stable project boundary is clearly violated, making future changes error-prone.

### P3 — Low Priority

Examples:

* local clarity or consistency problem;
* real maintenance cost exists, but primary behavior is not currently affected.

Severity is determined by actual impact and trigger probability, not by the rule name or code style.

---

## 9. How to Write a Finding

Each finding should contain:

```text
[P1/P2/P3] Concise title
Location: file:line or the smallest relevant range
Problem: what is specifically wrong
Trigger: under what input / call path / state it occurs
Impact: what result it causes
Evidence: project contract, code path, test, or applicable reference
Recommendation: minimal direction; a full refactoring plan is unnecessary
```

Prefer a causal chain that can be verified:

```text
change
→ triggering condition
→ incorrect behavior
→ user / data / contract impact
```

Do not put long tutorials inside findings.

---

## 10. Review Output

When findings exist:

1. list findings first, ordered by severity;
2. then list any necessary items that still need confirmation;
3. finally provide a brief note about the validation scope and checks that were not run.

When there are no findings, state clearly:

> No defects with sufficient supporting evidence were found within the actual review scope.

Still explain:

* what was reviewed;
* what validation was actually run;
* which checks were not run and why.

Final principle:

> The Review Skill is responsible for proving why something is a problem; the domain references define what the correct rule is.
