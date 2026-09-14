# AGENTS.md

This file retains hard constraints that apply continuously. Detailed coding knowledge is loaded by Skills on demand and is not read in full by default.

## Core Rules

1. Before making changes, read the relevant code, similar implementations, and tests, and inspect available recent Git history. If they cannot be found, state that honestly.
2. Make only the minimum changes necessary for the current task. Do not opportunistically refactor, rename, reformat, or upgrade unrelated dependencies.
3. Before adding a class or component, search for existing implementations. First determine its responsibility and model type, then determine the Package, and only then create the file. Reuse existing capabilities whenever possible.
4. Do not invent business states, business codes, default values, or compatibility rules. Do not scatter magic values with business or technical semantics through the code. Prefer Enum for fixed, finite value domains. Maintain cross-class constants by responsibility rather than creating a large catch-all constant repository. Request / Query (both belong to the `<module>.request` Package by default), DTO / BO / DO / VO, and other POJOs must not define field default values. Unless explicitly required, do not change APIs, fields, state transitions, permissions, data scopes, deletion semantics, transaction semantics, or database constraints.
5. Controllers must not access Mapper / DAO directly. HTTP semantics must not leak into Service / Manager. Lower layers must not depend back on upper layers. For cross-module calls, prefer the other module's Service / Facade. Introduce Manager only when needed. A Service whose primary responsibility is ordinary CRUD, query, and resource-management operations must use a `*ManageService` name, for example `PlaceManageService`; do not name such a class only `PlaceService`. Services centered on a specific business use case or domain action may use a more specific responsibility-oriented name instead.
6. Database objects use lowercase Chinese Pinyin with underscores and should reuse existing terminology where possible. Java uses English business semantics. DO properties should also prefer English names; isolate physical database naming through explicit persistence mappings such as SQL column aliases or MyBatis ResultMap. Database Pinyin must not leak into business models or APIs.
7. Concrete business API outputs should prefer VO. The default unified HTTP response wrapper is `ApiResponse<T>`. If the target project already has another unified response type, historical API, or serialization contract, follow the project's existing convention and do not force migration merely to match this standard. Do not classify every data model as DTO.
8. Structural constraints already guaranteed by a trusted inbound boundary, such as Bean Validation, must not be mechanically repeated in Service / Manager with equivalent null, blank, size, and similar checks. In multi-entry scenarios, complete the truly missing entry validation or shared contract. Do not hide input that should be rejected by using unsupported fallbacks such as empty strings, `0`, default codes, or default states.
9. When a clear non-null collection contract already exists, upper layers must not mechanically add defenses such as `list == null ? emptyList : list` or `Optional.ofNullable(list)`. Prefer empty collections to represent no results. If an external or legacy source genuinely permits null, normalize it once at the boundary closest to the source rather than adding fallback logic at every Service / Manager layer.
10. Transactions are determined by data-consistency requirements. Ordinary snapshot reads do not explicitly start transactions by default and must not receive transactions mechanically based on the number of Mapper calls. Atomic operations may live in Manager; writes spanning multiple Managers may be coordinated in Service. Keep the transaction scope limited to the necessary operations.
11. Do not introduce concurrency, asynchronous execution, caching, or additional abstractions for speculative performance gains. Business code must not create threads directly or arbitrarily create thread pools. When asynchronous execution is truly needed, prefer the project's existing Executor and evaluate connection pools, context propagation, exceptions, and transaction boundaries. Do not assume transactions propagate across threads.
12. Use SOLID to identify real responsibility, extension, contract, interface, and dependency problems. Do not use SOLID as a reason to mechanically create Interface + Impl, Strategy, Factory, Repository wrappers, or additional layers.
13. Do not bypass authentication, authorization, data-permission controls, or tenant isolation. Do not hard-code or log credentials such as passwords, Tokens, or private keys. Do not weaken existing security mechanisms.
14. After changes, inspect the complete diff and run the project's existing relevant tests, static checks, and architecture checks. Do not disable checks, delete failing tests, weaken assertions, or hide exceptions merely to make validation pass.
15. Report actual validation results and unverified items honestly. A review request is read-only by default and must not automatically become a fix task.
16. Do not rewrite Git history merely to make a branch look cleaner. Commit-history restructuring is a separate operation and requires explicit user intent.

## Skill Entry Points

- For Java / Spring Boot / MyBatis / MyBatis-Plus / Rabbit-SQL / PostgreSQL development, fixes, and refactoring, use [java-spring-backend](java-spring-backend/SKILL.md).
- For backend implementation review, use [backend-code-review](backend-code-review/SKILL.md). It finds evidence-backed correctness, architecture, security, transaction, concurrency, and maintainability problems; it does not own the verification pipeline.
- For test quality and executable verification after implementation or review, use [review-and-test](review-and-test/SKILL.md). It owns tests, type checks, lint/static analysis, builds, and verification reporting.
- For cleaning up or splitting a completed branch into semantic commits, use [rework-commits](rework-commits/SKILL.md). Invoke it explicitly only; it rewrites local Git history and must preserve the final repository tree exactly.

Preferred completion flow for non-trivial backend changes:

```text
implementation
→ backend-code-review
→ fix confirmed issues
→ review-and-test
→ rework-commits only when explicitly requested
→ PR / merge
```

Do not automatically create a separate agent, and do not rerun checks that have already passed and are unaffected by later changes.

## Source and Priority of Rules

Apply rules in this order: explicit requirements of the current task → correctness, security, and data integrity → project architecture rules → applicable domain standards → reasonable consistency with the current module → general language and framework conventions. This ordering must never be used to override higher-level instructions or bypass security boundaries.

Domain standards are maintained centrally under [java-spring-backend/references](java-spring-backend/references/). This directory is the only detailed backend standards source for this Skill Pack; do not maintain duplicate copies.

Keep `java-spring-backend`, `backend-code-review`, and `review-and-test` as sibling Skills so they can share the central references without duplication. `rework-commits` is intentionally independent of backend language/framework rules, but should remain a sibling Skill when this pack is installed as a whole. Keeping only this file does not replace the corresponding Skill files. For concrete security implementation, also inspect the target project's existing security standards; this pack does not contain a standalone `security.md`.
