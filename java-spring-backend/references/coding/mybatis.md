# MyBatis / MyBatis-Plus Coding Standard

This document defines usage standards for MyBatis and MyBatis-Plus.

This document answers:

> How should Mapper, Mapper XML, MyBatis-Plus BaseMapper, parameter binding, ResultMap, TypeHandler, dynamic SQL, and MyBatis technical components be organized and implemented?

For SQL correctness, safe scope, PostgreSQL semantics, and performance, read:

- [SQL and PostgreSQL](../database/sql.md)

Related references:

- [Application Layering and Model Boundaries](../architecture/layering.md)
- [Database Design](../database/database-design.md)
- [Transactions](../architecture/transactions.md)
- [Java Coding](java.md)

Core principles:

> Mapper defines data-access contracts and MyBatis mappings; it does not own business flow.

> Physical database naming and Java business naming are isolated through explicit MyBatis mapping.

> In MyBatis-Plus projects, reuse the basic CRUD supplied by `BaseMapper`. Keep custom conditional queries and updates as explicit Mapper methods with locatable SQL instead of hiding query semantics in Wrapper chains.

> MyBatis technical rules belong here. SQL rules are not duplicated here.

---

## 1. Mapper Responsibility

Mapper is the outbound database-access boundary.

Primary responsibilities:

* define data-access methods;
* map parameters;
* map results;
* associate Mapper XML;
* execute corresponding SQL.

Ordinary MyBatis example:

```java
public interface PlaceMapper {

    PlaceDO getById(String id);

    List<PlaceDO> listByQuery(PlaceQuery query);

    int insert(PlaceDO place);

    int update(PlaceDO place);
}
```

For the basic form in a MyBatis-Plus project, see section 7.

Mapper does not own:

* HTTP / RPC;
* business authorization decisions;
* state transitions;
* complete business flows;
* business transaction orchestration;
* third-party service calls.

For layering responsibilities, read:

- [layering.md](../architecture/layering.md)

---

## 2. Mapper Package

Within a business module:

```text
<module>.mapper
```

is reserved for that module's database-access Mappers.

For example:

```text
place.mapper.PlaceMapper
case.mapper.CaseMapper
```

Do not place a technical component in a business `mapper` Package merely because a Mapper uses it.

For example:

```text
place.mapper.JsonStringListTypeHandler
```

is normally the wrong responsibility because TypeHandler is MyBatis infrastructure rather than a Place Mapper.

Principle:

> Package placement follows the component's own responsibility, not its current consumer.

---

## 3. MyBatis Technical Infrastructure

Common MyBatis technical components may follow the target project's existing structure under locations such as:

```text
common.mybatis.handler
common.mybatis.interceptor
common.mybatis.plugin
common.mybatis.config
```

Typical mapping:

```text
TypeHandler   → common.mybatis.handler
Interceptor   → common.mybatis.interceptor
Plugin        → common.mybatis.plugin
Configuration → common.mybatis.config
```

If the project already has another clear shared technical Package, follow it rather than creating a second hierarchy just for this reference.

---

## 4. Search Before Adding a MyBatis Component

Before adding any of the following:

```text
Mapper
TypeHandler
Interceptor
Plugin
ResultHandler
MyBatis Configuration
```

confirm:

1. whether the same or similar capability already exists;
2. where components of the same kind currently live;
3. whether the existing capability can be reused;
4. whether the component is a business Mapper or shared MyBatis infrastructure;
5. whether adding a new component is actually necessary.

Workflow:

```text
determine responsibility
→ search similar implementations
→ determine Package
→ create or modify
```

---

## 5. Mapper / DAO Method Naming

Mapper / DAO method names should directly express data-access intent. When the target project has no stronger stable convention, use these prefixes for custom methods:

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
listPlaces
listByQuery
listByStatus
countByQuery
countByStatus
insert
insertBatch
update
updateStatus
deleteById
deleteByStatus
```

Use `get` for a single object. Avoid vague names such as:

```text
queryOne
findData
loadInfo
```

Use `list` for collection queries. When directly naming a resource collection, plural nouns are fine:

```text
listPlaces
listCases
```

When the important semantic is the condition, `listByStatus` and `listByQuery` are equally valid. Do not sacrifice condition meaning merely to force plural naming.

Use `count` for counts, for example:

```text
countByQuery
countByStatus
```

Use persistence verbs for writes:

```text
insert...
delete...
update...
```

Service-layer create / delete business actions use `save` / `remove` by default, while Mapper / DAO uses `insert` / `delete`. This keeps business actions distinct from persistence terminology. For Service naming, read `layering.md`.

Keep the original names of framework methods already supplied by MyBatis-Plus `BaseMapper`, for example:

```text
selectById
selectList
selectCount
insert
updateById
deleteById
```

Do not add a pass-through wrapper merely to rename a framework method. These naming rules primarily govern project-defined Mapper / DAO methods.

Avoid:

```text
handle
process
doQuery
executeBusiness
```

Principle:

> Custom Mapper / DAO methods use stable data-access verbs. Names express what data is accessed and under what conditions; they do not express a complete business flow and do not wrap framework methods merely for naming style.

---

## 6. Mapper Parameters

A single or small number of parameters can be passed directly.

When several simple parameters must be referenced explicitly in XML, use:

```java
@Param
```

For example:

```java
int updateStatus(@Param("id") String id, @Param("status") String status);
```

When query conditions become numerous, prefer a Query model instead of continually extending the parameter list or using:

```java
Map<String, Object>
```

For Query responsibilities and Package placement, read:

- [layering.md](../architecture/layering.md#92-query)

For ordinary Java parameter-count guidance and Query thresholds, read:

- [java.md](java.md#51-control-parameter-count)

Method-declaration blank lines and wrapping follow the Java formatting reference. Keep a signature on one line when it remains clear within the project's line width; do not mechanically wrap merely because `@Param` is present.

This document does not maintain a second parameter-count or formatting standard.

### 6.1 Null Contract for Collection Queries

An ordinary MyBatis collection query such as:

```java
List<PlaceDO> listByQuery(PlaceQuery query);
```

represents “zero to many rows.” Standard MyBatis collection queries should represent no matches with an empty collection, not `null`.

Therefore callers should not mechanically add:

```java
List<PlaceDO> places = placeMapper.listByQuery(query);
List<PlaceDO> safePlaces = places == null
        ? new ArrayList<>()
        : places;
```

or:

```java
Optional.ofNullable(places)
        .orElseGet(Collections::emptyList);
```

When the non-Null collection contract already exists, use it directly:

```java
List<PlaceDO> places = placeMapper.listByQuery(query);
return places.stream()
        .map(...)
        .toList();
```

If the target project has a custom Mapper implementation, plugin, proxy, or another data-access wrapper that explicitly changes the standard collection-return contract, follow the actual project contract. Do not infer only from a method name.

This rule does not apply to a single-object query such as:

```java
PlaceDO getById(String id);
```

Whether absence returns `null`, `Optional`, or throws follows the project contract.

If a Null collection comes from a third-party SDK, external Client, or another source that genuinely permits Null, normalize it once at the nearest Client / Adapter boundary and expose a stable collection contract upward. Do not repeat fallback handling at every Service / Manager layer.

Principle:

> Normalize source-specific Null behavior once at the boundary, then let upper layers depend on a stable contract. Standard collection queries use an empty collection for no rows, not Null.

---

## 7. MyBatis-Plus Usage

This section applies only when the target project actually uses MyBatis-Plus. Do not introduce MyBatis-Plus into an ordinary MyBatis project merely to apply these rules.

### 7.1 Entity Mapper / DAO Extends `BaseMapper`

In a MyBatis-Plus project, a business Mapper / DAO corresponding to a persistence entity should extend:

```java
BaseMapper<T>
```

For example:

```java
public interface PlaceMapper extends BaseMapper<PlaceDO> {

    List<PlaceDO> listByQuery(PlaceQuery query);

    long countByQuery(PlaceQuery query);
}
```

Prefer `BaseMapper` methods for basic CRUD where they already clearly provide the needed operation:

```text
insert
selectById
updateById
deleteById
```

Do not redeclare identical basic CRUD in every Mapper.

If a data-access interface is not a single-table persistence-entity Mapper—for example a pure aggregate query, a cross-table read-only projection, or a project-specific DAO abstraction—do not fabricate a meaningless `BaseMapper<...>` entity merely to satisfy this form. Respect the target project's real data-access boundary.

Principle:

> MyBatis-Plus Mappers with a real persistence entity extend `BaseMapper` to reuse basic CRUD. Do not duplicate framework capabilities and do not invent an entity merely to inherit the interface.

### 7.2 Do Not Use MyBatis-Plus Wrapper Condition Builders in Business Code

Business code must not use MyBatis-Plus Wrapper condition builders for query or update conditions, including:

```text
QueryWrapper
LambdaQueryWrapper
UpdateWrapper
LambdaUpdateWrapper
Wrappers.query(...)
Wrappers.lambdaQuery(...)
Wrappers.update(...)
Wrappers.lambdaUpdate(...)
```

Do not hide complex conditions in Service / Manager through chained Wrapper construction.

Reasons include:

1. SQL logic becomes scattered in Java condition-building code instead of remaining reusable and centrally maintainable;
2. when investigating slow SQL, production SQL, or database logs, developers cannot easily locate the corresponding XML / Mapper implementation from recognizable SQL fragments;
3. complex Wrapper usage spreads data-access details into Service / Manager and weakens the Mapper / SQL boundary;
4. progressively chained conditions often make final SQL semantics less readable and reviewable than explicit XML.

For custom conditional queries, updates, and statistics, prefer explicit Mapper methods with SQL maintained in XML:

```java
public interface PlaceMapper extends BaseMapper<PlaceDO> {

    List<PlaceDO> listByQuery(PlaceQuery query);

    int updateStatus(
            @Param("id") String id,
            @Param("expectedStatus") String expectedStatus,
            @Param("targetStatus") String targetStatus);
}
```

```xml
<select id="listByQuery" resultMap="PlaceResultMap">
    SELECT
        id,
        csbh,
        zt
    FROM csxx
    <where>
        <if test="placeCode != null and placeCode != ''">
            AND csbh = #{placeCode}
        </if>
        <if test="status != null">
            AND zt = #{status}
        </if>
    </where>
</select>
```

Direct primary-key CRUD from `BaseMapper` is not Wrapper condition building and may be used normally.

Principle:

> Use MyBatis-Plus to reuse stable basic CRUD. Do not use Wrapper to move business queries back into Java; keep conditional SQL explicit, searchable, and reusable through Mapper methods and SQL resources.

### 7.3 Do Not Hard-Code Business Constants in Mapper XML

Mapper XML must not directly hard-code business statuses, types, source codes, or other fixed business identifiers.

Avoid:

```xml
SELECT
    id,
    csbh,
    zt
FROM csxx
WHERE zt = '1'
  AND lx = 'FORMAL'
```

Pass the values explicitly from the Java boundary through Mapper / DAO parameters:

```java
List<PlaceDO> listByStatusAndType(@Param("status") String status, @Param("type") String type);
```

```xml
SELECT
    id,
    csbh,
    zt
FROM csxx
WHERE zt = #{status}
  AND lx = #{type}
```

This keeps business constants under Java business semantics and prevents the same code from being scattered through multiple SQL files.

This rule applies to **business constants**, not every SQL literal. Normal SQL structure such as the following may remain:

```text
SELECT 1
COUNT(*)
IS NULL / IS NOT NULL
fixed LIMIT with explicit technical meaning
required literals inside CASE / COALESCE and similar SQL structures
```

But if `'1'`, `'0'`, `'PENDING'`, or `'FORMAL'` actually means a business status or type, it must not remain hard-coded merely because it is convenient.

Principle:

> SQL structure may own SQL literals; business constants belong to the Java business contract and enter XML through Mapper parameters rather than being duplicated as business codes in SQL.

---

## 8. `#{}` and `${}`

Use:

```xml
#{placeId}
```

for ordinary data parameters, for example:

```xml
WHERE id = #{placeId}
```

`${}` is text substitution, not ordinary parameter binding.

Evaluate `${}` only when SQL structure cannot be represented with PreparedStatement parameters, for example:

* table name;
* column name;
* sort field;
* a fixed SQL structural fragment.

Such values must first go through strict allow-list mapping:

```text
user input
   ↓
allow-list / Enum mapping
   ↓
fixed SQL identifier
   ↓
${}
```

Forbidden:

```text
user input → ${} → SQL
```

For SQL injection and dynamic-sort safety, read `sql.md`.

---

## 9. Database-to-Java Mapping Boundary

Physical database columns and Java properties may follow different naming systems.

Under this Skill Pack:

```text
Database
→ project-standard database naming

Java
→ English business semantics
```

For example, an existing database may use:

```text
zjhm
rqsj
lqsj
csbh
```

while Java uses:

```text
identityNumber
entryTime
exitTime
placeCode
```

Use ResultMap, column aliases, TypeHandler, or equivalent MyBatis mechanisms to form the mapping boundary.

Do not let physical database naming spread into Java business models merely to save mapping code.

### 9.1 Do Not Map the Same Field Twice

Every selected field must have one clear owner for converting its SQL result name into a Java property name.

This rule applies to **all result fields**, without regard to how the field is produced, including:

```text
ordinary table columns
JOIN columns
aggregate columns
statistics columns
calculated columns
CASE expressions
function results
subquery projections
other SQL expressions
```

Do not use SQL `AS` to perform Java-facing naming for a field and then map that alias again through an explicit `resultMap`. Once a field's SQL-to-Java naming conversion has been expressed by one mapping mechanism, do not express the same conversion a second time through another mechanism.

Avoid even for ordinary table fields:

```xml
<select id="getPerson" resultMap="PersonResultMap">
    SELECT
        p.zjhm AS identity_number,
        p.rqsj AS entry_time,
        p.lqsj AS exit_time
    FROM person p
    WHERE p.id = #{id}
</select>

<resultMap id="PersonResultMap" type="PersonDO">
    <result property="identityNumber" column="identity_number"/>
    <result property="entryTime" column="entry_time"/>
    <result property="exitTime" column="exit_time"/>
</resultMap>
```

Here the same fields are mapped twice:

```text
zjhm → identity_number → identityNumber
rqsj → entry_time      → entryTime
lqsj → exit_time       → exitTime
```

The problem is the duplicated mapping boundary, not whether the field is a normal column, aggregate, statistic, or calculated expression.

Choose one mapping strategy for each query.

If `resultMap` owns the mapping boundary, keep the SQL result names at their database / SQL names and perform the Java-property mapping only in `resultMap`:

```xml
<select id="getPerson" resultMap="PersonResultMap">
    SELECT
        p.zjhm,
        p.rqsj,
        p.lqsj
    FROM person p
    WHERE p.id = #{id}
</select>

<resultMap id="PersonResultMap" type="PersonDO">
    <result property="identityNumber" column="zjhm"/>
    <result property="entryTime" column="rqsj"/>
    <result property="exitTime" column="lqsj"/>
</resultMap>
```

If SQL aliases own the mapping boundary, use `resultType` and let the aliases map directly to Java properties according to the project's MyBatis auto-mapping convention:

```xml
<select id="getPerson" resultType="PersonDO">
    SELECT
        p.zjhm AS identity_number,
        p.rqsj AS entry_time,
        p.lqsj AS exit_time
    FROM person p
    WHERE p.id = #{id}
</select>
```

The same rule applies to statistics, aggregates, and calculated fields. For example, do not combine:

```xml
<select id="selectFormalStats" resultMap="PlaceFormalStatsMap">
    SELECT
        COUNT(*) AS total_count,
        COUNT(CASE WHEN f.yxx = #{enabledStatus} THEN 1 END) AS enabled_count
    FROM zfba_cs_001 f
</select>

<resultMap id="PlaceFormalStatsMap" type="PlaceFormalStatsDO">
    <result property="totalCount" column="total_count"/>
    <result property="enabledCount" column="enabled_count"/>
</resultMap>
```

Prefer either alias + `resultType`:

```xml
<select id="selectFormalStats" resultType="PlaceFormalStatsDO">
    SELECT
        COUNT(*) AS total_count,
        COUNT(CASE WHEN f.yxx = #{enabledStatus} THEN 1 END) AS enabled_count
    FROM zfba_cs_001 f
</select>
```

or a single explicit `resultMap` mapping boundary when the SQL result names can remain stable.

This rule does not forbid `AS` itself. `AS` may still be required for SQL semantics, duplicate-column disambiguation, derived tables, expressions, or stable SQL result labels. The prohibition is specifically against using `AS` as one Java-facing naming conversion and then adding another mapping of the same field through `resultMap`.

Principle:

> Every result field is mapped from SQL naming to Java naming once. The rule applies equally to ordinary columns, joined columns, aggregates, statistics, calculations, and expressions.

For database naming, read:

- [database-design.md](../database/database-design.md)

For DO responsibility, read:

- [layering.md](../architecture/layering.md#95-do)

---

## 10. ResultMap

When database column names or types differ from Java properties, prefer clear explicit mappings.

For example:

```xml
<resultMap id="PersonResultMap" type="PersonDO">
    <id property="id" column="id"/>
    <result property="identityNumber" column="zjhm"/>
    <result property="name" column="xm"/>
    <result property="entryTime" column="rqsj"/>
    <result property="exitTime" column="lqsj"/>
</resultMap>
```

For special types, specify a TypeHandler explicitly when needed:

```xml
<result
    property="tags"
    column="bq"
    typeHandler="com.example.common.mybatis.handler.JsonStringListTypeHandler"/>
```

A mapping should let the reader see:

```text
database column
→ Java property
→ special type conversion
```

Do not rename Java business properties backward merely to rely on auto-mapping.

---

## 11. TypeHandler

TypeHandler performs **technical conversion** between database types and Java types.

Common examples:

```text
JSON / JSONB ↔ Java Collection / Object
database code ↔ Java Enum
database-specific type ↔ Java type
```

TypeHandler does not:

* query business data;
* call Service / Manager;
* decide permissions;
* decide business state;
* implement full business exception handling.

Principle:

> TypeHandler converts types; it does not implement business flow.

Search the project for an equivalent implementation before adding a shared TypeHandler.

---

## 12. Dynamic SQL

MyBatis dynamic SQL may appropriately use:

```text
<if>
<choose>
<when>
<otherwise>
<foreach>
<where>
<set>
<trim>
```

For example:

```xml
<where>
    <if test="placeName != null and placeName != ''">
        AND csmc LIKE CONCAT('%', #{placeName}, '%')
    </if>
    <if test="status != null">
        AND zt = #{status}
    </if>
</where>
```

Dynamic SQL selects SQL structure. It should not implement complete business flows or complex business state machines.

Correctness, safety, and efficiency of the SQL conditions themselves are determined by `sql.md`.

---

## 13. Reusing SQL Fragments

Use:

```xml
<sql>
<include>
```

to reuse stable and clearly scoped SQL fragments where appropriate.

Do not create layers of nested `<sql>` fragments merely to eliminate a few repeated lines.

> SQL readability takes precedence over formal DRY.

---

## 14. Mapper XML

Mapper XML should keep:

* a clear namespace;
* easy correspondence between SQL and Mapper methods;
* clear parameter names;
* explicit ResultMap where needed;
* readable dynamic SQL;
* explicit TypeHandler usage;
* business constants passed as Mapper parameters instead of scattered literals.

Mapper XML should not contain:

* complete business flows;
* business authorization orchestration;
* extensive decisions unrelated to database access;
* unjustified hard-coded business statuses, types, or codes.

For SQL formatting, SELECT, JOIN, write scope, pagination, and performance, read:

- [sql.md](../database/sql.md)

---

## 15. Mapper and Transactions

Mapper executes database operations, but it does not define the complete business transaction boundary.

Do not add business transactions at the Mapper layer merely because INSERT / UPDATE / DELETE exists there.

For transaction rules, read:

- [transactions.md](../architecture/transactions.md)

This reference requires only that Mapper itself not orchestrate cross-business transaction semantics.

---

## 16. SQL Rules Are Not Duplicated in the MyBatis Reference

The following are maintained by `sql.md`:

```text
SELECT *
COUNT / NULL
JOIN / LEFT JOIN
WHERE / time ranges
INSERT / UPDATE / DELETE
pagination / sorting
EXISTS / IN
N+1
Batch
PostgreSQL syntax
indexes and EXPLAIN
SQL performance
```

If a MyBatis task changes actual SQL, also load `sql.md`. If it changes only `BaseMapper`, ResultMap, TypeHandler, or parameter mapping, do not load all SQL rules merely for form.

---

## 17. Codex MyBatis Change Workflow

When modifying MyBatis / MyBatis-Plus code:

1. Determine whether the change concerns a Mapper interface, Mapper XML, BaseMapper, ResultMap, TypeHandler, or other MyBatis infrastructure.
2. Search the current project for similar implementations.
3. Before adding a component, determine its Package from its responsibility.
4. Check custom Mapper / DAO names for clear `get / list / count / insert / delete / update` data-access semantics; do not wrap existing MyBatis-Plus methods merely to rename them.
5. In MyBatis-Plus projects, check whether entity Mapper / DAO extends `BaseMapper<DO>` and whether basic CRUD has been redeclared unnecessarily.
6. Check for `QueryWrapper`, `LambdaQueryWrapper`, `UpdateWrapper`, and related Wrapper usage; business conditional SQL should be an explicit Mapper method + XML.
7. Check whether XML hard-codes business status, type, source, or other business constants; pass them through Mapper parameters.
8. Check whether parameters should use `#{}` and whether any `${}` is truly structural and allow-listed.
9. Check collection Mapper Null contracts; standard collection queries should not trigger mechanical Null fallback in upper layers.
10. Check every selected field for a single SQL-to-Java naming-mapping boundary, regardless of whether it is an ordinary column, JOIN column, aggregate, statistic, calculation, or expression; do not use a Java-facing SQL alias and then map the same field again through `resultMap`.
11. Ensure TypeHandler performs technical conversion only.
12. Ensure dynamic SQL remains readable and does not hide business flow.
13. When SQL changes, also read `sql.md`.
14. When transactions are involved, read `transactions.md`.
15. Run the target project's relevant existing tests.

Key review points:

* Mapper owns only data access;
* custom Mapper / DAO names accurately express single-object, collection, count, insert, delete, and update semantics;
* MyBatis-Plus entity Mappers correctly reuse `BaseMapper`;
* no meaningless forwarding wrappers are added around existing `BaseMapper` methods solely for naming;
* Wrapper does not hide conditional SQL inside Java business code;
* XML does not hard-code values that belong to the Java business contract;
* MyBatis technical components are not placed inside a business Mapper Package;
* existing TypeHandler / Interceptor / Plugin implementations are not duplicated;
* Query / DO responsibility and Package follow `layering.md`;
* standard `List<T>` query callers do not add unjustified `list == null ? emptyList : list` defenses;
* when a source really is nullable, normalization occurs once at the nearest source boundary rather than at every Service / Manager layer;
* physical database naming does not leak into Java unnecessarily;
* every selected field—ordinary, joined, aggregate, statistical, calculated, or expression-based—uses one SQL-to-Java naming-mapping boundary rather than being mapped twice through `AS` and `resultMap`;
* ResultMap clearly expresses column/property mapping;
* `${}` does not receive raw user input;
* TypeHandler does not contain business logic;
* Mapper XML does not hide complex business flows;
* SQL and transaction rules are not re-invented in this reference.

Final principle:

> The MyBatis / MyBatis-Plus reference defines how Java connects to and maps SQL. MyBatis-Plus is used to reuse stable basic CRUD; custom conditions remain explicit SQL; standard collection queries return empty collections for no rows. The SQL reference determines whether SQL itself is correct, safe, clear, and efficient.
