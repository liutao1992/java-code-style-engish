# Rabbit-SQL Coding Standard

This document defines project usage standards for Rabbit-SQL and its Spring Boot Starter.

This document answers:

> How should `@XQLMapper`, `.xql`, `XQLFileManager`, Baki, parameter binding, dynamic SQL, pagination, Stream, Batch, caching, and Spring transactions be organized and used?

For SQL correctness, safe scope, PostgreSQL semantics, and performance, read:

- [SQL and PostgreSQL](../database/sql.md)

Related references:

- [Application Layering and Model Boundaries](../architecture/layering.md)
- [Project and Business Module Structure](../architecture/project-structure.md)
- [Java Coding](java.md)
- [Spring](spring.md)
- [Transactions](../architecture/transactions.md)
- [Testing](testing.md)

MyBatis / MyBatis-Plus is a different persistence framework. For its rules, read:

- [MyBatis / MyBatis-Plus](mybatis.md)

Core principles:

> A Rabbit-SQL Mapper is still an outbound database-access boundary. Business code should normally access the database through responsibility-specific `@XQLMapper` interfaces instead of allowing Baki, SQL names, or XQL technical details to spread into Service / Controller.

> Bind data values with `:name` named parameters through prepared statements. `${}` is only for trusted SQL templates or allow-listed SQL structure and must not carry untrusted external input.

> Dynamic XQL expresses SQL structure and query conditions. It does not replace structural validation in Controller, business rules in Service / Manager, authorization decisions, or transaction design.

> When the target project already has an established Rabbit-SQL version, configuration style, and stable conventions, follow them. Do not upgrade dependencies, switch persistence frameworks, or bulk-migrate existing code merely to apply this reference.

---

## 1. First Confirm the Project Actually Uses Rabbit-SQL

Load this document when one or more of these features are present:

```text
rabbit-sql
rabbit-sql-spring-boot-starter
@XQLMapper
@XQLMapperScan
@XQL
@Arg
Baki / BakiDao
XQLFileManager
xql-file-manager.yml
*.xql
PagedResource
@CountQuery
@PageableConfig
```

If the project uses MyBatis / MyBatis-Plus, do not apply this reference merely because an interface is also called Mapper.

Likewise, do not apply MyBatis-Plus-specific requirements such as:

```text
BaseMapper<T>
QueryWrapper
LambdaQueryWrapper
Mapper XML
```

to Rabbit-SQL `@XQLMapper` interfaces.

When a project uses Rabbit-SQL, MyBatis, JPA, or other frameworks together, identify the actual data-access implementation from dependencies, annotations, SQL resources, and the call chain. Never infer the framework solely from a `Mapper` / `DAO` name.

Principle:

> Identify the persistence technology first, then load its framework rules. The logical Mapper / DAO responsibility may be similar, but technical implementation rules must not be mixed.

---

## 2. Prefer `@XQLMapper` Interfaces in Business Code

Rabbit-SQL provides both the Baki API and XQL interface mapping.

In an ordinary business module, prefer:

```text
Service / Manager
      ↓
@XQLMapper Mapper
      ↓
*.xql
      ↓
Database
```

For example:

```java
@XQLMapper("place")
public interface PlaceMapper {

    PlaceDO getById(@Arg("id") String id);

    List<PlaceDO> listByQuery(PlaceQuery query);
}
```

Avoid direct calls in ordinary Service / Controller code such as:

```java
baki.query("&place.listByQuery")
baki.execute("&place.updateStatus", args)
```

because this easily creates a dependency chain like:

```text
business layer
→ Rabbit-SQL API
→ XQL alias / SQL name
→ parameter Map
```

which spreads database-access protocol and SQL-location details into the business layer.

Baki is more appropriate for:

* data-access infrastructure;
* low-level generic capabilities that genuinely do not fit interface mapping;
* development diagnostics, tooling, or controlled technical scenarios;
* an existing stable Baki data-access abstraction in the target project.

If a business scenario truly needs Baki, keep it contained inside a responsibility-specific data-access boundary rather than letting upper layers assemble SQL or SQL names everywhere.

Principle:

> The business layer depends on a data-access contract, not the Rabbit-SQL execution API. `@XQLMapper` is the default boundary for ordinary business data access; Baki is a lower-level capability, not a Service shortcut for SQL execution.

---

## 3. Mapper and XQL Resource Organization

Rabbit-SQL Mapper continues to follow the general layering standard and normally lives under:

```text
<module>.mapper
```

XQL resources can be organized centrally under resources, for example:

```text
src/main/resources/
├── xql-file-manager.yml
└── xqls/
    ├── place/
    │   └── place.xql
    ├── casecenter/
    │   └── case.xql
    └── equipment/
        └── equipment.xql
```

If the target project already has a stable location such as:

```text
sql/
xql/
rabbit-sql/
```

follow it rather than migrating resources for this reference.

Use stable, clear business semantics for aliases in `xql-file-manager.yml`:

```yaml
files:
  place: xqls/place/place.xql
  case: xqls/casecenter/case.xql
```

Avoid meaningless aliases such as:

```text
a
sql1
common
misc
all
```

Do not put every SQL statement in the entire system into one giant XQL file. Prefer splitting by business module, aggregate, or stable data-access responsibility so the relationship is easy to trace:

```text
Mapper
↔ XQL alias
↔ XQL file
↔ SQL object
```

Prefer XQL files on the classpath and versioned with the application by default. Rabbit-SQL supports `file://`, FTP, HTTP(S), and other remote resources, but ordinary business systems should not introduce runtime remote SQL without a clear requirement because it adds configuration drift, availability, authorization, and deployment-consistency risks.

If remote XQL is truly used, at minimum confirm:

* the source is trusted and access-controlled;
* Token / Secret is not hard-coded in the repository;
* configuration and release processes can trace versions;
* startup / runtime behavior for remote unavailability is explicitly designed;
* caller input cannot select an arbitrary remote SQL URL.

---

## 4. Keep SQL Object Names Clearly Mapped to Mapper Methods

Name XQL SQL objects with syntax such as:

```sql
/*[listByQuery]*/
select ...;
```

By default, prefer the same SQL object name as the Mapper method:

```text
PlaceMapper.listByQuery
↕
/*[listByQuery]*/
```

This improves code search, IDE navigation, slow-SQL diagnosis, and manual reading.

Use `@XQL` to specify SQL name or behavior only when there is a real need, for example:

* the method name genuinely differs from the SQL object name;
* the same SQL must map to methods with different return types;
* the method prefix does not reliably imply the SQL type;
* the default execution type must be overridden explicitly.

Do not mechanically add redundant `@XQL` to every method.

### 4.1 Mapper Method Naming Still Follows Team Conventions

Custom Rabbit-SQL Mapper methods use the same default team verbs:

```text
get one object      → get
get multiple objects→ list
get a count         → count
insert              → insert
delete              → delete
modify              → update
```

Examples:

```text
getById
getByCode
listByQuery
listByStatus
countByQuery
insert
updateStatus
deleteById
```

Rabbit-SQL can infer SQL type from common method-name prefixes. Query prefixes include `select / query / find / get / fetch / search / list`; writes include `insert / save / add / append / create`, `update / modify / change`, and `delete / remove`.

The team's `get`, `list`, `insert`, `update`, and `delete` conventions align naturally with those inference rules.

However, `count` is not a default Rabbit-SQL query prefix, so a `count...` method should not rely on implicit inference. Declare query type explicitly:

```java
@XQL(type = SqlStatementType.query)
long countByQuery(PlaceQuery query);
```

If the SQL object name also differs, specify `value` as well:

```java
@XQL(value = "countEnabled", type = SqlStatementType.query)
long countByStatus(@Arg("status") String status);
```

Principle:

> Keep stable team naming conventions. When the framework cannot reliably infer behavior from the name, supplement it with annotations rather than weakening clear team naming to satisfy inference.

---

## 5. Mapper Parameters: Prefer Typed Models or `@Arg`

Rabbit-SQL mapped interfaces support:

```text
single Map / DataRow / JavaBean
multiple @Arg parameters
batch Iterable
```

When ordinary business query conditions are numerous, prefer an existing Query model:

```java
List<PlaceDO> listByQuery(PlaceQuery query);
```

The XQL can use JavaBean properties as named parameters:

```sql
/*[listByQuery]*/
select id, csbh, zt
from csxx
where 1 = 1
-- #if :placeCode != blank
  and csbh = :placeCode
-- #fi
-- #if :status != null
  and zt = :status
-- #fi
;
```

For a few independent parameters, use:

```java
int updateStatus(@Arg("id") String id, @Arg("status") String status);
```

Parameter names must match XQL named parameters.

Do not default to:

```java
Map<String, Object>
DataRow
Object[]
```

as parameter bags for stable business interfaces that have clear semantics.

These dynamic structures are appropriate only when the data shape is genuinely dynamic, the code is infrastructure, or the target project already has an explicit contract for them.

For ordinary Java parameter count, Query usage, and parameter-object design, read `java.md`. Query lives under `<module>.request` by default.

---

## 6. Bind Data Values with `:name`

Rabbit-SQL named parameters such as:

```sql
:id
:name
:status
```

are bound through PreparedStatement behavior.

Values coming from:

```text
HTTP Request
RPC / Message
Service parameters
database query results
external-system responses
user input
```

should use `:name` by default.

For example:

```sql
where id = :id
  and zt = :status
```

Do not switch to text templating merely because string concatenation appears more convenient.

Principle:

> Anything that can be bound as a data value uses `:name`. PreparedStatement parameters are the default path; do not turn data into SQL text.

---

## 7. Strictly Restrict `${}` Text Templates

Rabbit-SQL:

```text
${name}
${!name}
```

performs text substitution and is not PreparedStatement data binding.

Even if `${!name}` applies safety handling to some string collections, do not treat it as equivalent to `:name` or pass arbitrary external input to it.

`${}` is permitted only for two kinds of cases.

### 7.1 Trusted Internal XQL Template Reuse

For example:

```sql
/*{whereCondition}*/
where id = :id;

/*[getById]*/
select id, csbh
from csxx
${whereCondition};
```

The template content is defined in a version-controlled XQL file and does not come from an external request.

### 7.2 SQL Structure That Cannot Be Parameterized

For truly dynamic structure such as:

```text
column name
table name
ORDER BY field
sort direction
controlled SQL fragment
```

first perform Enum or allow-list mapping in Java / a stable configuration boundary:

```text
external input
→ Enum / allow-list
→ fixed SQL identifier
→ ${}
```

Forbidden:

```text
user input
→ ${}
→ SQL
```

For `IN` value collections, prefer `#for` + `:item` so each value remains prepared-statement-bound instead of converting the user collection into text templating.

Also read `sql.md` for SQL injection and dynamic-sort safety.

---

## 8. Do Not Hard-Code Business Constants in XQL

Do not directly embed business statuses, business types, source codes, or fixed business identifiers in `.xql`.

Avoid:

```sql
where zt = '1'
  and lx = 'FORMAL'
```

Pass them explicitly through Mapper parameters:

```java
List<PlaceDO> listByStatusAndType(@Arg("status") String status, @Arg("type") String type);
```

```sql
where zt = :status
  and lx = :type
```

When the value domain is fixed, prefer a responsibility-specific Enum on the Java side according to `java.md`, then pass the established stable code at the data-access boundary.

This rule targets business constants. It does not forbid every SQL literal: `COUNT(*)`, `IS NULL`, function arguments, and values genuinely belonging to SQL structure are judged according to SQL semantics.

Principle:

> XQL expresses SQL; Java business contracts own business codes. Do not scatter the same business state across Java and multiple XQL files.

---

## 9. Dynamic SQL Expresses SQL Structure Only

Rabbit-SQL provides dynamic capabilities through SQL comments such as:

```text
#check
#var
#if / #else / #fi
#guard / #throw
#switch / #case / #end
#choose / #when / #end
#for / #done
```

Use these for:

* optional query conditions;
* SQL branches;
* `IN` parameter expansion;
* database-dialect differences;
* SQL-local variables and structural assembly.

Do not implement in XQL:

```text
business authorization decisions
audit state machines
complete business flows
cross-data-source business validation
structural validation already owned by Controller
business rules owned by Service / Manager
```

### 9.1 Use `#if` for Optional SQL Conditions

For example:

```sql
-- #if :status != null
  and zt = :status
-- #fi
```

Do not nest so many `#if` branches that XQL becomes an unreadable SQL program. If branches represent different data-access semantics, prefer separate SQL objects / Mapper methods.

### 9.2 Prefer `#for` for Parameterized Collections

When constructing `IN`, preserve value parameterization:

```sql
and id in (
-- #for item of :ids; last as isLast
  :item
  -- #if !:isLast
  ,
  -- #fi
-- #done
)
```

The goal is not merely to “produce valid SQL”; collection elements must still enter through named parameters and the prepared-statement path.

### 9.3 `#check` Does Not Replace Business Validation

`#check` may enforce a necessary precondition directly related to the current SQL execution, but it should not repeat the same structural constraint already guaranteed by trusted inbound Bean Validation, nor should it move business status, authorization, or cross-table rules into XQL.

The responsibility order remains:

```text
structural constraint
→ inbound boundary

business rule
→ Service / Manager

SQL execution condition
→ XQL / Database
```

### 9.4 `#var` Does Not Own Business Decisions

`#var` is appropriate for lightweight SQL-local calculation and dynamic-script helper variables. If a variable represents a business state transition, permission result, or complex business algorithm, compute it in the Java business layer and pass it as an explicit parameter.

---

## 10. Reuse SQL Fragments Only When Readability Improves

Rabbit-SQL supports independent templates and inline templates.

Extract a template only when multiple SQL statements genuinely need to share the same logic. Do not create recursively nested templates merely to remove a few repeated SQL lines.

A particularly useful case is shared list/page and count filtering:

```text
list/page SQL condition
          ↕
shared inline condition
          ↕
count SQL condition
```

This can prevent bugs where:

```text
a filter is added to the list query
but the count query is not updated
```

Prefer a narrowly scoped inline template when the fragment serves only one SQL group; do not pollute global templates with local conditions.

Principle:

> Reuse stable, same-responsibility SQL fragments only when it prevents semantic drift. Do not sacrifice locatability merely to satisfy DRY.

---

## 11. Do Not Leak Unnecessary Dynamic Framework Types

Rabbit-SQL supports return types such as:

```text
List / Set / Stream
Optional
Map
DataRow
JavaBean
PagedResource / IPageable
scalar values
BatchResult
```

Ordinary business Mappers should prefer responsibility-specific Java types, for example:

```text
PlaceDO
List<PlaceDO>
PlaceStatsDO
```

Do not let:

```text
DataRow
Map<String, Object>
```

leak through Service, Controller, and public API without a real reason.

Dynamic-column reports or infrastructure capabilities may legitimately use DataRow / Map when a stable model cannot be defined, but the boundary must be explicit.

Database DOs are not exposed directly through external APIs; read `layering.md` for model boundaries.

`PagedResource<T>` / `IPageable` are Rabbit-SQL technical pagination types. When the target project has a unified public pagination contract, convert at the appropriate boundary instead of making framework pagination types part of the public HTTP contract.

For a missing single object and Optional usage, follow the target project's existing contract. Do not bulk-change historical signatures merely because Rabbit-SQL supports Optional.

---

## 12. Pagination Queries Must Have Explicit Count Semantics

Rabbit-SQL can build pagination and count automatically and also supports:

```text
@CountQuery
@PageableConfig
```

Simple queries may reuse the framework's default pagination mechanism.

Pay particular attention to count consistency when the query contains:

* `GROUP BY`;
* `DISTINCT`;
* multi-level CTEs;
* complex JOINs;
* custom pagination SQL;
* different dynamic conditions between list and count;
* a structure where the default count rewrite cannot reliably represent the real total.

When an independent count SQL is needed, associate it explicitly with `@CountQuery` and preferably reuse the stable common conditions so list data and total do not drift.

Do not assume a complex SQL total is correct merely because the framework can auto-count it. Actual SQL semantics remain governed by `sql.md`.

---

## 13. Stream Queries Must Have an Explicit Connection Lifecycle

A Rabbit-SQL Stream query is lazy: it executes at a terminal operation and holds underlying database resources.

Close it at an explicit boundary:

```java
try (Stream<PlaceDO> stream = placeQuery.stream()) {
    return stream.map(...).toList();
}
```

Do not:

* forget to close it;
* store an unclosed Stream in a member field;
* return a database Stream directly from a Controller as an HTTP response;
* consume it across threads when its lifecycle is unclear;
* convert every List query to Stream merely to “avoid one loop.”

Choose Stream only when data volume, processing pattern, and resource lifetime genuinely benefit from lazy consumption.

Principle:

> A database Stream is a resource lifecycle, not just a Java collection API. Whoever creates the lazy query must make close responsibility explicit.

---

## 14. Use Framework Batch Capabilities Instead of Per-Item SQL Loops

For batch insert / update, prefer Rabbit-SQL batch capabilities or the target project's existing batch abstraction instead of:

```text
for each item
→ one SQL statement
→ one database round trip
```

Interface mapping can use batch types, and Baki also provides collection-oriented batch execution.

Batch size follows project and database capacity; do not invent a fixed number. Consider Rabbit-SQL batchSize together with connection-pool capacity, per-row data size, transaction scope, and PostgreSQL parameter limits.

Whether an entire batch belongs in one transaction is determined by the consistency requirements in `transactions.md`; “batch” alone does not justify a larger transaction.

---

## 15. Spring Boot Projects Use the Spring Transaction System

With `rabbit-sql-spring-boot-starter`, Rabbit-SQL can participate in Spring-managed database transactions.

Determine transaction boundaries from `transactions.md` first:

```text
first determine which database operations must commit / roll back together
→ determine whether Service / Manager owns the boundary
→ choose the Spring transaction implementation
```

If the project already uses `@Transactional` / `TransactionTemplate`, keep a unified Spring transaction approach instead of introducing a second Rabbit-SQL-specific habit.

The Spring Boot Starter provides a simple Spring `Tx` wrapper, but do not introduce it merely because the framework supports it when the target project does not already use it.

In a Spring Boot Starter project, do not mix in the old Rabbit-SQL Core transaction implementation:

```text
com.github.chengyuxing.sql.transaction.Tx
```

because the Starter is already integrated with Spring global transactions.

If Rabbit-SQL and MyBatis / JPA access the database within the same business transaction, confirm:

* a compatible Spring TransactionManager is used;
* the data source belongs to the same transactional resource when that is required;
* multi-data-source code does not accidentally use the wrong default transaction manager;
* transaction propagation across threads is not assumed.

For Spring Proxy and transaction API details, read `spring.md`.

---

## 16. Cache Is Not a Default Optimization

Rabbit-SQL provides a `QueryCacheManager` extension point, but do not enable query caching merely because it exists.

Before introducing caching, make explicit:

```text
whether repeated queries actually exist
whether the cache Key is complete
the business basis for TTL
how writes invalidate cached values
how transaction commit timing is handled
how multiple instances stay consistent
how much staleness is acceptable
```

Without monitoring, benchmarks, or a clear business benefit, do not add caching for speculative performance.

Caching must not hide an obviously slow SQL statement, incorrect index design, or N+1 behavior.

---

## 17. SQL Observability and Logging

Rabbit-SQL offers SQL interceptors, execution watchers, and related extension points for SQL observation, timing, and diagnostics.

If the project already has unified SQL monitoring / tracing / metrics, integrate with it rather than creating a parallel system.

Logs and monitoring should help locate:

```text
XQL alias / SQL object
execution time
failure type
necessary caller context
```

but must not log for convenience:

```text
password
Token / Secret
full identity information
sensitive business content
large amounts of unmasked SQL parameters
```

Slow-SQL conclusions require real execution plans, data volume, and database metrics. Do not infer “slow” merely from SQL length.

---

## 18. The IDEA Rabbit SQL Plugin Is a Development Aid, Not a Substitute for Standards or Tests

The official plugin can provide:

* XQL file creation and registration;
* SQL-name completion;
* Java ↔ XQL navigation;
* dynamic SQL parsing / execution tests;
* Mapper interface generation;
* SQL description and metadata viewing.

Using it can reduce mistakes in aliases, SQL names, and dynamic scripts.

Generated code must still follow the target project's:

```text
Package
naming
method wrapping
model responsibilities
exceptions
transactions
```

Do not keep generated formatting that conflicts with project style merely because a plugin produced it.

Manual dynamic-SQL tests in the plugin do not replace automated tests.

---

## 19. Rabbit-SQL Test Focus

When adding or modifying XQL, at minimum verify:

1. the XQL alias is registered correctly;
2. `@XQLMapper` value matches the alias;
3. Mapper method and SQL object names match, or `@XQL` is explicit when they do not;
4. execution type can be inferred correctly, and non-default prefixes such as `count...` declare query type explicitly;
5. Java parameter names / `@Arg` match `:name`;
6. all external data values remain prepared-statement-bound;
7. `${}` comes only from trusted templates or allow-listed SQL structure;
8. major dynamic-SQL branches all generate valid SQL;
9. list and count filters remain semantically consistent;
10. Stream is closed on every exit path;
11. Batch behavior, failure semantics, and transaction scope are correct;
12. PostgreSQL-specific SQL is validated in a genuinely compatible environment;
13. the change does not break existing Mapper return models or upper-layer contracts.

For real SQL behavior, prefer the project's existing integration tests. When database semantics matter, use `testing.md` to evaluate Testcontainers or a test database.

---

## 20. Codex Rabbit-SQL Change Workflow

When modifying Rabbit-SQL code:

1. **Identify the framework.** Confirm Rabbit-SQL from dependencies, `@XQLMapper`, Baki, `.xql`, and configuration; do not apply MyBatis-Plus rules.
2. **Locate the boundary.** Find the complete mapping among business Mapper, XQL alias, XQL file, and SQL object.
3. **Prefer interface mapping.** Ordinary business queries use `@XQLMapper`; Baki does not spread into Service / Controller without a real reason.
4. **Keep naming aligned.** SQL object names normally match Mapper methods; methods follow `get / list / count / insert / update / delete` conventions.
5. **Confirm execution type.** `count...` and other methods whose type cannot be reliably inferred explicitly use `@XQL(type = ...)`.
6. **Design parameters.** Use Query / a clear JavaBean for multiple conditions and `@Arg` for a few parameters; do not default to Map / DataRow parameter bags.
7. **Check injection boundaries.** Data values use `:name`; `${}` is limited to trusted templates or allow-listed SQL structure.
8. **Check business constants.** Status, type, source codes, and similar values are passed from Java rather than hard-coded in XQL.
9. **Control dynamic SQL.** `#if / #for / #choose` expresses SQL structure, not complete business rules.
10. **Check return models.** Framework types such as DataRow / Map / PagedResource do not leak unnecessarily into business or HTTP boundaries.
11. **Check resources.** Stream closes explicitly; Batch avoids per-row database round trips.
12. **Check transactions.** Spring Boot projects use `transactions.md` and `spring.md` and do not mix in Core Tx.
13. **Check performance.** Cache, parallelism, and batch size require real evidence; slow SQL is analyzed with `sql.md` and execution plans.
14. **Validate.** Test XQL registration, mappings, parameters, dynamic branches, pagination count, transactions, and database compatibility.

Final principle:

> Rabbit-SQL is valuable because SQL remains native, explicit, and locatable while interface mapping and dynamic capabilities are available. Good use does not mean moving more logic into XQL; it means clear Mapper contracts, searchable SQL, safe parameters, and explicit resource and transaction boundaries.
