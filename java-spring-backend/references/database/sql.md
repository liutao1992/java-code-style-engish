# SQL and PostgreSQL Standard

This document defines SQL correctness, safe scope, readability, and PostgreSQL query standards.

This document answers:

> Is the SQL itself correct, safe, and clear, and—when evidence exists—does it need optimization?

For framework-specific MyBatis `#{}` / `${}`, Mapper XML, ResultMap, TypeHandler, and related usage, read:

- [MyBatis](../coding/mybatis.md)

For table, column, constraint, index, and Migration design, read:

- [Database Design](database-design.md)

Core principle:

> Make SQL correct first, clear second, and optimized last. Performance conclusions should be supported whenever possible by PostgreSQL execution plans and real data.

---

## 1. Parameterization and Injection Safety

Ordinary data values must enter SQL through the parameterization mechanism provided by the target framework. Do not concatenate raw user input into SQL.

When SQL structure cannot be parameterized—for example dynamic sort fields, column names, or table names—first map the input through a strict allow-list.

Principle:

```text
ordinary data → parameter binding
SQL identifiers / structure → allow-list mapping before use
```

MyBatis-specific binding syntax is maintained in `mybatis.md`; this document does not duplicate `#{}` / `${}` rules.

---

## 2. SELECT Columns

As a rule, avoid:

```sql
SELECT *
FROM ryxx;
```

Select only required columns:

```sql
SELECT
    id,
    xm,
    zjhm,
    rqsj,
    lqsj
FROM ryxx;
```

Benefits include:

* less unnecessary data transfer;
* lower impact from Schema changes;
* clearer mappings;
* reduced risk of accidentally exposing newly added columns.

---

## 3. COUNT and NULL

To count rows, normally use:

```sql
COUNT(*)
```

unless the NULL semantics of a specific column are intentionally required.

Use:

```sql
IS NULL
IS NOT NULL
```

for NULL checks.

Do not use:

```sql
column = NULL
column <> NULL
```

Conditions and aggregates involving NULL must account for SQL three-valued logic.

---

## 4. WHERE Conditions and Time Ranges

WHERE conditions should preferably operate directly on original columns.

For a time range, prefer:

```sql
WHERE rqsj >= #{startTime}
  AND rqsj <  #{endTime}
```

rather than mechanically using:

```sql
WHERE DATE(rqsj) = #{date}
```

for a daily query.

Prefer half-open time ranges:

```text
[start, end)
```

Do not manually represent the end of a day as `23:59:59.999999`.

Column and parameter types should match. Avoid accidental reliance on PostgreSQL implicit type conversion.

---

## 5. JOIN

A JOIN must make explicit:

* the join condition;
* one-to-one / one-to-many cardinality;
* whether the result set may be multiplied;
* whether a filter belongs in the JOIN condition or final WHERE clause.

Do not allow an accidental Cartesian product because a join condition is missing.

With `LEFT JOIN`, be aware that:

```sql
LEFT JOIN csxx c ON ...
WHERE c.zt = '1'
```

may make the result semantics effectively approach an INNER JOIN. Confirm that this is the intended business behavior.

---

## 6. DISTINCT and GROUP BY

Do not add:

```sql
DISTINCT
```

merely because duplicate rows appear.

First determine whether duplication comes from an incorrect JOIN, a one-to-many relationship, or a query-model problem.

Use DISTINCT only when the business genuinely requires deduplication.

Aggregate queries must have clear grouping dimensions and follow PostgreSQL GROUP BY semantics.

---

## 7. ORDER BY and Pagination

When stable order is required—especially for pagination—use an explicit `ORDER BY`.

Prefer an ordering that deterministically resolves ties, for example:

```sql
ORDER BY cjsj DESC, id DESC
```

For ordinary data volumes, this may be appropriate:

```sql
LIMIT #{pageSize}
OFFSET #{offset}
```

For deep pagination, evaluate Keyset Pagination based on the actual business sort fields; do not mechanically rewrite existing pagination.

Client-controlled sort fields must be mapped through an allow-list.

---

## 8. EXISTS and IN

When only existence matters, express existence rather than loading a full list and checking it in Java.

PostgreSQL example:

```sql
SELECT EXISTS (
    SELECT 1
    FROM ryxx
    WHERE zjhm = #{identityNumber}
);
```

`IN` is suitable for a bounded value set, but do not build unbounded huge lists.

For large volumes, evaluate based on the actual case:

* batching;
* JOIN;
* temporary tables;
* PostgreSQL arrays;
* other established project solutions.

---

## 9. INSERT

INSERT statements should list columns explicitly rather than relying on database column order.

For example:

```sql
INSERT INTO ryxx (
    id,
    xm,
    zjhm,
    rqsj
)
VALUES (
    #{id},
    #{name},
    #{identityNumber},
    #{entryTime}
);
```

Rely on database defaults only when their responsibility and business semantics are explicit.

---

## 10. UPDATE

UPDATE must have an explicit WHERE condition and the actual modification scope must be understood.

For state races, consider a conditional update when appropriate:

```sql
UPDATE csxx
SET zt = #{targetStatus}
WHERE id = #{id}
  AND zt = #{expectedStatus};
```

Whether conditional update, optimistic locking, pessimistic locking, or another mechanism is needed depends on real concurrency and transaction requirements.

For consistency rules, read:

- [Transactions](../architecture/transactions.md)

---

## 11. DELETE

DELETE must have an explicit scope.

Do not accidentally generate:

```sql
DELETE FROM ryxx;
```

Before deletion, confirm the target project's existing:

* physical / logical deletion semantics;
* related-data handling;
* data authorization;
* business impact.

Do not change deletion semantics merely because a different implementation is convenient.

---

## 12. UPSERT

PostgreSQL supports:

```sql
INSERT ... ON CONFLICT ...
```

but conflict semantics must be based on an explicit PRIMARY KEY or UNIQUE Constraint.

Do not use without justification:

```sql
ON CONFLICT DO NOTHING
```

merely to hide duplicate-data problems.

---

## 13. Batch Operations

For large writes, evaluate batch operations rather than executing one unbounded SQL statement per item.

Depending on the project and data volume, consider:

* JDBC / MyBatis Batch;
* PostgreSQL multi-VALUES INSERT;
* COPY;
* batched commits.

Batch size must account for data volume, SQL length, memory, transaction scope, and database capacity. Do not copy a fixed “best practice” threshold without evidence.

---

## 14. N+1

Inspect the call pattern, not just an individual SQL statement.

Typical problem:

```text
query 100 main records
       ↓
execute 100 related queries in a loop
```

When database access occurs inside a loop, evaluate:

* batch query;
* JOIN;
* IN;
* one query followed by in-memory grouping;
* changing the query model.

Do not eliminate N+1 by creating an unbounded giant JOIN or IN list.

---

## 15. PostgreSQL Functions, LIKE, and Indexes

When a WHERE condition applies a function to an indexed column, evaluate whether the index can still serve the query.

For example:

```sql
WHERE lower(xm) = lower(#{name})
```

High-frequency fuzzy search such as:

```sql
LIKE '%keyword%'
ILIKE '%keyword%'
```

must not be assumed acceptable on a large table. Based on real requirements, evaluate `pg_trgm`, GIN / GiST, or full-text search.

Do not create indexes solely from intuition; index design is governed by `database-design.md`.

---

## 16. OR, UNION, Subqueries, and CTEs

Large OR expressions should be evaluated for readability and execution plan. Do not merge semantically different queries merely to make SQL shorter.

When deduplication is not required, consider:

```sql
UNION ALL
```

but do not mechanically replace `UNION`.

A subquery is not inherently a performance problem; do not automatically rewrite every subquery as a JOIN.

CTEs:

```sql
WITH ...
```

can help decompose complex queries and staged calculations, but do not rewrite simple SQL into CTEs merely because they appear more sophisticated.

---

## 17. Indexes Must Match Real Queries

Indexes should serve real:

```text
WHERE
JOIN
ORDER BY
GROUP BY
```

patterns.

For example, if a query repeatedly uses:

```sql
WHERE fzxbh = #{centerCode}
  AND rqsj >= #{startTime}
  AND rqsj <  #{endTime}
```

then a composite index such as:

```text
(fzxbh, rqsj)
```

may be worth evaluating.

Whether it should be created—and in what column order—must be determined from actual data volume and execution plans.

For Schema-level index rules, read:

- [database-design.md](database-design.md)

---

## 18. EXPLAIN and Performance Conclusions

For complex or performance-sensitive SQL, use:

```sql
EXPLAIN
```

to inspect scan types, Join Strategy, Sort, estimated rows, and similar information.

When real execution information is required, consider:

```sql
EXPLAIN ANALYZE
```

but remember that it actually executes the SQL. Do not casually run it on dangerous production write statements.

Do not reduce performance analysis to:

```text
uses index = fast
Seq Scan = slow
```

Consider:

* data volume;
* rows returned;
* execution time;
* buffers;
* rows scanned;
* estimation error.

If no real execution plan is available, clearly state that a recommendation is structural analysis only. Do not claim that a change “is already faster” or “will definitely use the index.”

---

## 19. SQL Readability

Complex SQL should use consistent formatting and clear structure.

Recommended:

```sql
SELECT
    a.id,
    a.zjhm,
    a.rqsj
FROM ryxx a
WHERE a.fzxbh = #{centerCode}
  AND a.rqsj >= #{startTime}
  AND a.rqsj <  #{endTime}
ORDER BY
    a.rqsj DESC,
    a.id DESC;
```

Core business rules should normally be expressed by the business layer. SQL should focus on data retrieval, aggregation, and data operations that the database is well suited to perform.

---

## 20. Codex SQL Checklist

After writing or modifying SQL, check:

1. Ordinary data uses parameter binding; structural dynamic content is allow-listed.
2. Unnecessary `SELECT *` is avoided.
3. NULL and time-range semantics are correct.
4. No unintended implicit type conversion occurs.
5. JOINs do not lack conditions or accidentally multiply the result set.
6. DISTINCT is not merely hiding an incorrect JOIN.
7. Pagination has stable ordering.
8. UPDATE / DELETE scope is safe.
9. No accidental N+1 or unbounded batch / IN pattern exists.
10. PostgreSQL syntax and types are correct.
11. Performance conclusions are supported by data, indexes, and EXPLAIN evidence.
12. SQL correctness and readability are not sacrificed for speculative performance gains.

Final principle:

> Persistence frameworks define “how Java binds to and maps SQL”; the SQL reference defines “whether the database statement itself is correct, safe, clear, and efficient.”
