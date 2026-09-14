# Backend Standards Routing Index

This file is the single routing index for detailed backend standards in this Skill Pack.

It answers:

> Which reference should be loaded for the current implementation, review, or verification task?

The references themselves remain the source of detailed rules. This file only maps task areas to those references and must not duplicate their domain guidance.

Load only the smallest necessary union. If the task expands into another area, load the additional reference then; do not recursively load every document.

---

## Routing Table

| Area involved | Load | Use it for |
| --- | --- | --- |
| Java implementation | [Java](coding/java.md) | naming, class design, constants, Enum, magic values, defaults, parameters, Null, collections, exceptions, logging, formatting |
| Physical project directory / business module | [Project structure](architecture/project-structure.md) | business-module location, business-first organization, physical directories, `common`, shared technical locations |
| Logical layering / models / responsibility Packages / SOLID | [Layering](architecture/layering.md) | Controller, Service, Manager, Mapper, Client, dependency direction, model classification, responsibility Packages, cross-module logical boundaries |
| Core business rules / application use cases / behavioral business objects | [Business rules](architecture/business-rules.md) | invariants, use-case boundaries, rule placement, behavioral objects, Clean Architecture concepts, domain-base-class decisions |
| Spring Framework | [Spring](coding/spring.md) | MVC annotations, Bean Validation, DI, Bean lifecycle, configuration properties, proxy behavior, Advice |
| HTTP API | [API](api/api-design.md) | URL, HTTP method, request/response contract, pagination, errors, idempotency, compatibility, externally exposed fields |
| Exception boundaries | [Error handling](architecture/error-handling.md) | propagation, translation, causes, logging ownership, external leakage |
| MyBatis / MyBatis-Plus | [MyBatis](coding/mybatis.md) | Mapper/DAO, XML, BaseMapper, Wrapper, binding, ResultMap, TypeHandler, interceptor/plugin |
| Rabbit-SQL | [Rabbit-SQL](coding/rabbit-sql.md) | `@XQLMapper`, Baki, XQL files, named parameters, dynamic SQL, pagination, Stream, Batch, Spring transaction integration |
| SQL / PostgreSQL query behavior | [SQL](database/sql.md) | SELECT/JOIN, predicates, writes, pagination, N+1, PostgreSQL, indexes, EXPLAIN evidence |
| Database schema | [Database design](database/database-design.md) | tables, columns, types, Null/defaults, keys, constraints, indexes, migrations, schema compatibility |
| Transactions / locking / consistency | [Transactions](architecture/transactions.md) | transaction necessity and scope, propagation, isolation, rollback, read-then-write, conditional update, optimistic/pessimistic locking |
| Concurrency / async | [Concurrency](architecture/concurrency.md) | executors, CompletableFuture, `@Async`, ThreadLocal/context propagation, synchronization, retries, async failure handling |
| Test design and testing standards | [Testing](coding/testing.md) | test scenarios, regression tests, test granularity, mocks, assertions, fixtures, integration/database/concurrency/transaction testing |
| Authentication / authorization / tenant / data scope / credentials | target project's existing security standards and implementation | security contracts, isolation, permissions, sensitive information |

---

## Common Combinations

```text
New business module
→ Project structure + Layering

New Request / Query / DTO / BO / DO / VO
→ Layering + Java

Repeated business-state rule across Services / entry points
→ Business rules + Layering + Java

Controller HTTP-contract change
→ API + necessary Spring

Spring MVC annotation-only change
→ Spring

Mapper XML with changed SQL semantics
→ MyBatis + SQL

@XQLMapper / .xql with changed SQL semantics
→ Rabbit-SQL + SQL

Table-column change plus Java persistence mapping
→ Database design + matching persistence reference

Determine transaction boundary
→ Transactions + necessary Layering

TransactionTemplate / @Transactional proxy behavior
→ Transactions + Spring

CompletableFuture database work
→ Concurrency + Transactions

Bug fix plus regression test
→ relevant domain reference + Testing
```

---

## Boundary Rules

Use the documents for different questions rather than loading overlapping guidance mechanically:

```text
Where does the business module / physical directory live?
→ project-structure.md

What logical responsibility does this class have, what may it depend on, and what responsibility Package does it use?
→ layering.md

Is this a stable core business rule or an application use-case rule, and should it move into a behavioral business object?
→ business-rules.md

How should the implementation be written in Java?
→ java.md

How should changed behavior be tested?
→ testing.md
```

Framework names do not prove framework usage. Before applying MyBatis, Rabbit-SQL, Spring transaction, or similar rules, inspect dependencies, imports, annotations, configuration, and resources.

The target project's stable contracts take priority over this Skill Pack's defaults. Do not use routing as a reason to migrate historical code unrelated to the current task.

Final principle:

> Route once, load narrowly, and keep detailed knowledge in its owning reference.
