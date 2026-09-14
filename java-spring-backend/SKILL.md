---
name: java-spring-backend
description: Develop, fix, and refactor Java, Spring Boot, MyBatis, MyBatis-Plus, Rabbit-SQL, and PostgreSQL code according to team backend standards. Use it to implement backend features, determine responsibilities, business rules and models, design APIs, database access, transactions, concurrency, and testing boundaries. Use backend-code-review for pure code review.
---

# Java Spring Backend

Turn the current task into the smallest code change that fits the target project's existing design.

Core approach:

> The Skill owns workflow and standards routing; references are the single source of detailed domain rules. Load only the standards actually needed by the current task and do not duplicate a second set of coding rules inside the Skill.

## Before You Start

- Read the applicable `AGENTS.md` in the target project and follow the user's current requirements and execution permissions.
- Route pure code-review work to the companion [backend-code-review](../backend-code-review/SKILL.md).
- Keep the two Skills as sibling directories; resolve reference links relative to the Skill Pack.
- When the target project already has stable contracts, framework mechanisms, and directory structures, continue using them. Do not bulk-migrate historical code merely because this Skill recommends a different default.

---

## Workflow

1. **Clarify the task.** Confirm the goal, affected modules, behavior changes, and acceptance criteria. Ask the user only about critical business intent that truly cannot be determined from code, contracts, and tests.
2. **Build context.** Read relevant code, similar implementations, build configuration, tests, and available recent Git history. If they cannot be found, state that honestly.
3. **Select standards.** Load only the references actually involved according to the routing below. If the task expands into a new area, load the additional reference then; do not recursively load all documents.
4. **Determine responsibilities.** Before adding or moving files, determine the business module, logical responsibility, model type, and dependency boundaries, then choose the physical location and Package.
5. **Implement the minimum change.** Prefer existing capabilities. Do not invent business states, codes, defaults, compatibility behavior, or extra abstractions.
6. **Self-review.** Inspect the complete diff and new files, then self-review the current task scope using the method defined by [backend-code-review](../backend-code-review/SKILL.md).
7. **Validate.** Run the project's existing relevant tests, static checks, and architecture checks. Do not assume a specific Wrapper, Formatter, or ArchUnit configuration exists.
8. **Report.** Explain actual changes, key behavior, validation results, failures, and items not run. Never describe an unexecuted check as passing.

---

## Standards Routing

### Java Implementation

Load [Java](references/coding/java.md):

```text
naming
class design
Lombok
class / record
constants / Enum / magic values
POJO field defaults
method design
method parameters / parameter objects
Null / Optional
collections / generics
collection return contracts
BigDecimal / time
Java catch / throw
logging
formatting / comments
```

Ordinary Java implementation does not automatically require Spring, API, or layering rules.

### Project Directories and Business Modules

Load [Project and business-module structure](references/architecture/project-structure.md):

```text
new business module
changes to physical project directories
business-first / technology-first organization
responsibility directories inside a module
ownership of common / util / constant / third
cross-module physical organization
```

Load `layering.md` only when logical class responsibilities must also be determined.

### Layering, Models, and Responsibility Packages

Load [Application layering](references/architecture/layering.md):

```text
Controller / Service / Manager / Mapper / Client / Adapter responsibilities
inbound / outbound adapters
dependency direction
Request / Query / DTO / BO / DO / VO classification
responsibility Packages
cross-module calls
caller-context boundaries
Service decomposition
SOLID
over-abstraction
```

When physical module / directory organization is involved, also load `project-structure.md`; when model implementation details are involved, also load Java.

### Business Rules and Use Cases

Load [Business rules and use-case boundaries](references/architecture/business-rules.md):

```text
core business rules / stable invariants
application-specific business rules / use-case flows
behavioral business objects
difference between Clean Architecture Entity and persistence DO
Service as a Use Case responsibility
whether an anemic model is actually a problem
whether business rules belong in objects or Service / Manager
boundary between Request / VO and application input/output models
dependency direction of core business rules
when not to add Entity / UseCase / Repository / Command / Result
```

This document borrows responsibility concepts from Clean Architecture but does not require the target project to be converted into full Clean Architecture. When concrete cross-layer dependencies are involved, also load `layering.md`; for model implementation, also load Java.

### Spring Framework

Load [Spring](references/coding/spring.md):

```text
@RestController / @Controller
@RequestMapping / @GetMapping / @PostMapping
Spring MVC annotation mechanisms
Bean Validation / @Valid / @Validated
dependency injection
Spring Bean lifecycle
@ConfigurationProperties
@Transactional proxy behavior
TransactionTemplate Spring API
@Async / @Cacheable proxy behavior
@RestControllerAdvice / @ExceptionHandler
```

Business responsibility itself is determined by `layering.md`; whether a transaction is needed is determined by `transactions.md`.

### HTTP API

Load [API](references/api/api-design.md):

```text
URL
HTTP Method
Path / Query / Body
external semantics of Request / VO
ApiResponse<T> or the project's unified response type
pagination / sorting
error codes / HTTP Status
idempotency
compatibility
externally exposed sensitive fields
```

If only Spring annotations change and the HTTP contract does not, Spring alone may be sufficient.

### Exceptions and Error Boundaries

Load [Error handling](references/architecture/error-handling.md):

```text
Mapper / Client exception propagation
whether Manager / Service should translate exceptions
exception cause
duplicate log + throw
which boundary owns error handling
whether internal exceptions leak to clients
```

If Java `catch` / `throw` / logging APIs are also involved, load Java; for Web Advice load Spring; for HTTP error contracts load API.

### MyBatis / MyBatis-Plus

Load [MyBatis / MyBatis-Plus](references/coding/mybatis.md):

```text
MyBatis Mapper / DAO interfaces
Mapper XML
MyBatis-Plus
BaseMapper<T>
QueryWrapper / LambdaQueryWrapper
UpdateWrapper / LambdaUpdateWrapper
business constants in XML
@Param
#{}/ ${}
ResultMap
TypeHandler
Interceptor / Plugin
MyBatis dynamic SQL
Mapper List<T> return contract
MyBatis technical Packages
```

When an ordinary `Mapper` / `DAO` name appears but the persistence framework is unclear, first inspect dependencies, annotations, and SQL resources. Do not apply MyBatis-Plus `BaseMapper` / Wrapper rules based on the name alone.

Load SQL only when actual SQL is modified.

### Rabbit-SQL

Load [Rabbit-SQL](references/coding/rabbit-sql.md):

```text
rabbit-sql / rabbit-sql-spring-boot-starter
@XQLMapper / @XQLMapperScan
@XQL / @Arg
Baki / BakiDao
XQLFileManager
xql-file-manager.yml
*.xql
:name named parameters
${} XQL string templates
#if / #for / #choose dynamic SQL
@CountQuery / @PageableConfig
PagedResource / IPageable
Stream queries
Batch
QueryCacheManager / executionWatcher
Rabbit-SQL Spring transactions
```

Rabbit-SQL Mapper and MyBatis Mapper are both outbound database adapters, but their framework rules differ. `@XQLMapper` does not need to extend MyBatis-Plus `BaseMapper`, and `.xql` is not MyBatis Mapper XML.

When SQL inside XQL actually changes, also load SQL. When Spring transactions are involved, also load Transactions and the necessary Spring rules.

### SQL / PostgreSQL

Load [SQL](references/database/sql.md):

```text
SELECT / JOIN
WHERE / NULL / time ranges
INSERT / UPDATE / DELETE
pagination / sorting
N+1 / Batch
PostgreSQL
index usage
EXPLAIN
```

Without a real execution plan, performance judgments can only be structural recommendations.

### Database Design

Load [Database design](references/database/database-design.md):

```text
tables / columns
types
Null / default values
primary keys / unique constraints
indexes
Migration
physical database naming
Schema compatibility
```

When SQL or Java mappings are also modified, load the corresponding SQL / MyBatis / Rabbit-SQL standards.

### Transactions

Load [Transactions](references/architecture/transactions.md):

```text
whether a transaction is needed
consistency scope
Service / Manager transaction boundaries
@Transactional
TransactionTemplate / programmatic transactions
propagation / isolation
read-then-write
conditional updates
optimistic / pessimistic locking
consistent snapshots
rollbackFor / rollback semantics
long transactions
transactions and thread switches
```

If Spring Proxy behavior or `TransactionTemplate` API details are involved, also load Spring.

### Concurrency

Load [Concurrency](references/architecture/concurrency.md):

```text
CompletableFuture
@Async
Executor / ThreadPoolTaskExecutor
thread pools
ThreadLocal / MDC / SecurityContext
Lock / synchronized / volatile
async exceptions
exceptionally / fallback
retries
concurrent resource capacity
```

When transactions are involved, also load Transactions.

### Testing

Load [Testing](references/coding/testing.md):

```text
bug fixes
new / changed business behavior
new tests
test failures
API / SQL / transaction / concurrency validation
Mock / integration tests / Testcontainers
```

Testing standards decide how to validate; they do not replace the relevant domain reference that decides how to implement.

### Security

For authentication, authorization, data scope, tenant isolation, and sensitive information, prioritize the target project's existing security standards, contracts, and implementation.

This Skill Pack currently has no standalone `security.md`. Never bypass authentication / authorization / tenant isolation, and never hard-code or log credentials such as passwords, Tokens, Secrets, or private keys.

---

## Common Combinations

```text
Too many parameters in an ordinary business method
→ Java

Magic values / constants class / fixed value domain / POJO defaults
→ Java

New business module
→ Project structure + Layering

New VO / Query / DO
→ Layering + Java

The same business-state rule is repeated as if + set in multiple Services / entry points
→ Business rules + Layering + Java

Decide whether a rule belongs in a behavioral business object or should remain in Service / Manager
→ Business rules + Layering

Determine whether a Controller Request must be converted to Command / DTO before calling Service
→ Business rules + Layering; add API if the HTTP contract changes

Convert traditional CRUD into an Entity / UseCase / Repository structure
→ Business rules + Layering; first prove stable invariants, responsibility benefits, and compatibility with the target project; do not migrate mechanically

Controller URL or response-contract change
→ API + necessary Spring

MVC Mapping annotation only
→ Spring

Duplicate structural validation between Bean Validation and Service
→ Spring

Null defense after standard MyBatis List<T> query
→ MyBatis + Java

Project uses MyBatis-Plus and adds a Mapper / DAO
→ MyBatis

QueryWrapper / LambdaQueryWrapper / UpdateWrapper appears
→ MyBatis

Business status or type is hard-coded in Mapper XML
→ MyBatis; add SQL when SQL correctness is also evaluated

New @XQLMapper / .xql / xql-file-manager.yml
→ Rabbit-SQL; add SQL when actual SQL changes

Baki leaks directly into Service / Controller, `${}` receives external input, or a Stream lifecycle issue appears in Rabbit-SQL
→ Rabbit-SQL + necessary Layering / SQL / Java

Rabbit-SQL and Spring transactions change together
→ Rabbit-SQL + Transactions + necessary Spring

Normalize a nullable collection from a third-party SDK
→ Java; add Layering if Client / Adapter responsibility must also be evaluated

New ResultMap / TypeHandler
→ MyBatis

Modify actual SQL in a Mapper
→ First identify MyBatis / Rabbit-SQL, then load the matching framework rules + SQL

Modify table columns and mappings
→ Database design + corresponding persistence-framework rules

Determine whether a transaction is required / where its boundary belongs
→ Transactions + necessary Layering

Local TransactionTemplate transaction block
→ Transactions + Spring

Determine whether @Transactional self-invocation works
→ Transactions + Spring

CompletableFuture database operation
→ Concurrency + necessary Transactions

Bug fix plus regression test
→ Relevant domain + Testing
```

Principle:

> For multi-domain tasks, load the union that is genuinely necessary. Do not load all references "just to be safe."

---

## Responsibility Check Before Creating a File

Before adding a file, answer in order:

```text
Which business module does it belong to?
        ↓
What is its logical responsibility?
        ↓
Can an existing implementation be reused?
        ↓
If it is a model, what model responsibility does it have?
        ↓
Which responsibility Package should it live in?
        ↓
Where is that Package physically located in the target project?
        ↓
Is a new file actually necessary?
```

`project-structure.md` owns the business-module location. `layering.md` owns the responsibilities of Controller / Service / Manager / Mapper / Client / models. `business-rules.md` owns how core business rules and application use-case flows are layered.

---

## Delivery Requirements

After completing the task, report:

* the purpose of the change;
* key behavior;
* necessary file locations;
* tests, static checks, and architecture checks actually executed;
* failures or omitted checks and the reasons;
* remaining real risks or business questions that still need confirmation.

Checks that already passed and are unaffected by later changes do not need to be rerun mechanically.

Final principle:

> The Skill owns workflow and routing; references are the single source of domain knowledge; the target project's real contracts take priority over the Skill's default examples.
