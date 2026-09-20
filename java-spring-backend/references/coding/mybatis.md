# MyBatis / MyBatis-Plus Coding Standard

This reference defines MyBatis-specific data-access, mapping, and XML conventions. It applies to ordinary MyBatis and, where explicitly stated, MyBatis-Plus. SQL correctness, performance, and PostgreSQL semantics belong to [SQL and PostgreSQL](../database/sql.md); business-layer responsibilities belong to [Application Layering](../architecture/layering.md).

Core rules:

- Mapper performs data access; Service evaluates business rules, makes business decisions, and orchestrates business flows.
- In MyBatis-Plus projects, reuse `BaseMapper` for applicable basic CRUD; keep custom SQL in explicit Mapper XML rather than Wrapper chains.
- Keep native SQL directly understandable and debuggable. Simple `<if>` conditions and reuse of stable fragments through `<sql>` / `<include>` are allowed when they improve clarity.
- Preserve a clear boundary between physical database naming and Java business properties.

---

## 1. Mapper Responsibility and Boundaries

Mapper defines data-access methods, binds parameters, maps results, and executes SQL. It must not handle HTTP / RPC, call third-party services, decide authorization, evaluate business state transitions, or orchestrate business transactions.

The Service layer determines business conditions and desired values before calling Mapper. SQL may apply those inputs to read, aggregate, insert, update, or delete data. A conditional write that checks an **expected value supplied by Service** for concurrency safety is still a data-access operation; deciding the next business state in SQL is not.

Follow [layering.md](../architecture/layering.md) for model and Service responsibilities, and [transactions.md](../architecture/transactions.md) for transaction boundaries. Do not establish cross-business transaction flow in Mapper.

---

## 2. Mapper Packages and MyBatis Infrastructure

Business Mapper interfaces belong in `<module>.mapper`, such as `place.mapper.PlaceMapper`. Shared MyBatis technical components belong in the project's existing technical package, for example:

```text
common.mybatis.handler       TypeHandler
common.mybatis.interceptor   Interceptor
common.mybatis.plugin        Plugin
common.mybatis.config        Configuration
```

Do not put a shared TypeHandler into a business Mapper package merely because that Mapper uses it. Follow an existing clear project structure instead of creating a competing hierarchy.

Before adding a Mapper, TypeHandler, Interceptor, Plugin, ResultHandler, or configuration, search for similar implementations, check whether they can be reused, establish the component's responsibility, and only then choose its package.

---

## 3. Mapper / DAO Method Naming

For **project-defined** methods, use verbs that describe data access:

| Purpose | Prefix | Examples |
| --- | --- | --- |
| One result | `get` | `getById`, `getByCode` |
| Collection | `list` | `listPlaces`, `listByQuery` |
| Count | `count` | `countByStatus` |
| Insert / delete / update | `insert` / `delete` / `update` | `insertBatch`, `deleteById`, `updateStatus` |

Prefer a meaningful condition in the name over mechanically enforcing plural nouns. Avoid vague or business-flow names such as `findData`, `handle`, and `executeBusiness`.

Keep the framework's existing `BaseMapper` names (for example, `selectById` and `updateById`); do not add forwarding methods solely to rename them. Service business actions may use `save` / `remove` according to [layering.md](../architecture/layering.md), while custom Mapper methods use persistence verbs.

---

## 4. Mapper Parameters and Return Contracts

Pass a small number of simple parameters directly and use `@Param` when several values must be named in XML:

```java
int updateStatus(@Param("id") String id, @Param("status") String status);
```

For numerous query conditions, prefer a Query model over an ever-growing parameter list or `Map<String, Object>`. Query model responsibilities belong to [layering.md](../architecture/layering.md#92-query); parameter-count and method-formatting guidance belongs to [java.md](java.md#51-control-parameter-count).

Standard MyBatis collection queries represent no matching rows as an **empty collection**, not `null`. Do not add mechanical `list == null ? emptyList : list` or `Optional.ofNullable(list)` defenses at each caller. Single-object query absence (`null`, `Optional`, or exception) follows the project contract.

If a custom Mapper implementation or wrapper explicitly changes the collection contract, honor the actual contract. For independently nullable SDK / Client collections, normalize once at the nearest source boundary rather than repeating fallbacks across Services.

---

## 5. MyBatis-Plus Usage

Only apply these requirements when the project actually uses MyBatis-Plus; do not introduce it solely to follow this reference.

An entity Mapper with a real persistence entity should extend `BaseMapper<DO>` and reuse applicable basic CRUD, including `insert`, `selectById`, `updateById`, and `deleteById`. Do not redeclare equivalent methods. A read-only projection or multi-table aggregate without a corresponding entity need not invent one solely to extend `BaseMapper`.

Custom queries, updates, and statistics must use explicit Mapper methods backed by native SQL in Mapper XML. Do not construct custom conditions in business code with `QueryWrapper`, `LambdaQueryWrapper`, `UpdateWrapper`, `LambdaUpdateWrapper`, or `Wrappers.*` factories.

The aim is to locate SQL from the Mapper method, inspect its main structure, compare it with logged SQL, and reproduce it in a database client or `EXPLAIN` without reconstructing Java Wrapper chains. This restriction does not prohibit direct framework CRUD that already fits the operation.

---

## 6. Native SQL and Business Logic Boundary

**SQL performs data operations; Service owns business rules.** Service decides authorization, workflow transitions, eligibility, and the business statuses or types to use, then passes explicit values through Mapper parameters. Do not encode those decisions in SQL `CASE` expressions, dynamic tags, or reusable fragments.

Mapper XML must not hard-code business statuses, source codes, or types such as `'PENDING'` or `'FORMAL'`. Bind them with `#{status}`, `#{type}`, etc. Fixed SQL literals with purely technical meaning (`SELECT 1`, `COUNT(*)`, `IS NULL`, technical `LIMIT`) are not business constants.

Ordinary SQL filtering, joins, aggregation, technical value conversion, and concurrency checks are allowed. For example, this is a database statistic, not a Service decision:

```sql
SELECT
    COUNT(*) AS total_count,
    COUNT(CASE WHEN zt = #{status} THEN 1 END) AS matching_count
FROM csxx
```

The status value must come from Service; the SQL only counts matching rows. A `CASE` that determines the next workflow state, however, belongs in Service. Do not move legitimate database statistics into Java merely because they use `CASE` or `COALESCE`.

For SQL semantics, write scope, and performance, use [sql.md](../database/sql.md), not a second set of SQL rules here.

---

## 7. Dynamic SQL and Shared Fragments

Use small, local `<if>` conditions for genuinely optional filters. Keep the primary `SELECT` / `FROM` / `WHERE` structure recognizable; do not implement business branching with dynamic tags. Avoid unnecessary nesting or layers of `<choose>`, `<when>`, `<otherwise>`, `<where>`, `<set>`, and `<trim>`. Use a structural tag only when a concrete SQL need or established project convention justifies it.

When multiple statements need **identical, stable SQL**, reuse a clearly named `<sql>` fragment with `<include>`, for example a common column list:

```xml
<sql id="PlaceColumns">
    p.id,
    p.csbh,
    p.zt
</sql>

<select id="getById" resultMap="PlaceResultMap">
    SELECT
        <include refid="PlaceColumns"/>
    FROM csxx p
    WHERE p.id = #{id}
</select>

<select id="listByQuery" resultMap="PlaceResultMap">
    SELECT
        <include refid="PlaceColumns"/>
    FROM csxx p
    WHERE 1 = 1
    <if test="status != null">
        AND p.zt = #{status}
    </if>
</select>
```

Do not extract one-use or trivial fragments, create deeply nested include chains, or hide unrelated large portions of a statement. A little duplication is preferable when abstraction makes executed SQL harder to reconstruct and debug. Shared fragments must not contain business decisions.

Mapper XML should also have a clear namespace, a direct correspondence between Mapper methods and statement IDs, and explicit result mappings or TypeHandlers where needed.

---

## 8. Parameter Binding and SQL Safety

Use `#{...}` for ordinary data values: `WHERE id = #{id}`. `${...}` performs textual substitution and must never receive raw user input.

Use `${...}` only if a structural SQL identifier (such as a table or column name or approved sort field) cannot be parameterized. First map the input to a fixed identifier via a strict allow-list / Enum; do not treat arbitrary strings as validated SQL fragments.

For complete SQL injection and dynamic-sort rules, read [sql.md](../database/sql.md).

---

## 9. Database-to-Java Mapping

Keep physical database names and Java business names separate. Select the **shortest clear mapping path** for every result field, including aggregate and calculated columns.

Prefer an explicit `resultMap` when it is needed for column/property differences, nested objects, constructors, special types, or TypeHandlers:

```xml
<select id="getPerson" resultMap="PersonResultMap">
    SELECT p.id, p.zjhm, p.rqsj
    FROM person p
    WHERE p.id = #{id}
</select>

<resultMap id="PersonResultMap" type="PersonDO">
    <id property="id" column="id"/>
    <result property="identityNumber" column="zjhm"/>
    <result property="entryTime" column="rqsj"/>
</resultMap>
```

Do not rename `zjhm` to `identity_number` in SQL and then map that intermediate name again merely to reach `identityNumber`. If aliases are the chosen Java-facing mapping, use `resultType` with aliases matching properties (or the project's established underscore-to-camel-case mapping), without adding a naming-only `resultMap`.

`AS` and `resultMap` may coexist when each has a distinct role: for example, aliases disambiguate same-named JOIN columns while `resultMap` builds a nested association:

```xml
<select id="getUserWithDepartment" resultMap="UserWithDepartmentMap">
    SELECT
        u.id AS user_id,
        d.id AS department_id
    FROM sys_user u
    JOIN department d ON d.id = u.department_id
    WHERE u.id = #{id}
</select>

<resultMap id="UserWithDepartmentMap" type="UserDO">
    <id property="id" column="user_id"/>
    <association property="department" javaType="DepartmentDO">
        <id property="id" column="department_id"/>
    </association>
</resultMap>
```

Never rename Java business properties backward solely to rely on auto-mapping. For schema naming and DO boundaries, see [database-design.md](../database/database-design.md) and [layering.md](../architecture/layering.md#95-do).

---

## 10. TypeHandler

TypeHandler converts database and Java types, such as JSON / JSONB to collections, database codes to Enum values, or database-specific types to Java types. Specify an explicit TypeHandler in a `resultMap` when necessary.

It must not query business data, call Service / Manager, decide permissions or state, or orchestrate business exceptions. Reuse an existing suitable technical component before creating another.

---

## 11. Codex MyBatis Change Checklist

When modifying MyBatis code:

1. Inspect relevant code, existing implementations, and tests; identify the component's responsibility and package before adding anything.
2. Verify that Mapper handles only data access and Service makes business decisions and owns business flow.
3. Check custom Mapper method names and parameter / Query contracts; preserve framework method names without forwarding wrappers.
4. In MyBatis-Plus projects, reuse applicable `BaseMapper` CRUD, and replace custom Wrapper-built conditions with explicit Mapper XML SQL.
5. Check that SQL contains no business-state decisions or hard-coded business codes; preserve legitimate filtering, aggregation, and concurrency checks.
6. Keep dynamic `<if>` conditions small and `<sql>` / `<include>` reuse genuinely shared and readable.
7. Bind data with `#{...}`; allow `${...}` only for strict allow-listed SQL structure.
8. Check collection return contracts and the shortest necessary database-to-Java mapping; use ResultMap and TypeHandler only for actual mapping / technical needs.
9. Read [sql.md](../database/sql.md) when changing SQL and [transactions.md](../architecture/transactions.md) when transaction behavior is involved.
10. Run the target project's relevant existing tests; report unavailable tests or unverified behavior without inventing results.
