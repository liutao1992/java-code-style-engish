# Database Design Standard

This document defines standards for database schemas, tables, columns, constraints, indexes, migrations, and physical database naming.

This document answers:

> How should the database structure itself be designed and constrained?

For Java model responsibilities, read:

- [Application Layering and Model Boundaries](../architecture/layering.md)

For mapping between database columns and Java properties, read:

- [MyBatis](../coding/mybatis.md)

For SQL and PostgreSQL query rules, read:

- [SQL and PostgreSQL](sql.md)

Core principle:

> The database expresses data structure and the minimum guarantees of data integrity; Java expresses business semantics. The two are isolated through explicit mapping.

---

## 1. Database Naming

Database objects use the target project's established convention of:

```text
lowercase Chinese pinyin + underscores
```

This applies to:

* tables;
* columns;
* indexes;
* constraints;
* sequences;
* views.

Examples from an existing schema might include:

```text
ryxx
ajxx
csxx
zjhm
rqsj
lqsj
cjsj
gxsj
```

Do not mix English, pinyin, and Chinese naming without a project basis inside the same Schema.

---

## 2. Use a Unified Business Vocabulary

Before adding a table, column, or other database object, search:

* Schema;
* migrations;
* SQL;
* Mapper XML;
* data dictionaries;

and confirm whether the project already has database terminology for the same business concept.

For example, if the project already uses:

```text
zjhm  → identity document number
rqsj  → entry time
lqsj  → exit time
csbh  → place code
fzxbh → sub-center code
```

continue to use the same terms rather than inventing another pinyin abbreviation for the same concept.

Principle:

> One business concept should have one stable database representation.

---

## 3. Database and Java Are Different Naming Boundaries

Physical database names do not have to become Java property names directly.

As a rule:

```text
Database
→ project-standard database naming

Java
→ English business semantics
```

Map between them explicitly through Mapper / ResultMap or the actual persistence mechanism.

This document does not duplicate ResultMap and TypeHandler rules. Read:

- [mybatis.md](../coding/mybatis.md)

Do not weaken Java business naming merely to reduce mapping code.

---

## 4. Schema Changes Must Be Versioned

Database structure must be managed through the project's formal SQL / Migration mechanism.

Do not:

* make changes only by hand in a database client;
* change the database without preserving a Migration;
* rely on ORM auto-creation for production Schema;
* use `ddl-auto=create` to maintain production tables.

If the project already uses:

```text
Flyway
Liquibase
custom Migration tooling
```

continue to use that mechanism.

---

## 5. Table Creation Must Consider Complete Constraints

When adding a business table, explicitly consider:

* primary key;
* data types;
* Null semantics;
* defaults;
* unique constraints;
* indexes;
* table comments;
* comments for important columns.

Database constraints should not be delegated entirely to compensating Java code.

---

## 6. Primary Keys

Business tables should normally have an explicit primary key.

A primary key should be:

* stable;
* unique;
* independent of easily changing display values.

Do not normally use a name, phone number, identity-document number, place name, or another mutable or sensitive business field directly as the system primary key.

The actual key-generation strategy must follow the target project's existing convention; the Skill Pack must not invent one.

---

## 7. Column Types

Column types should express real data semantics rather than using `varchar` for everything.

For example:

```text
time          → timestamp / date
integer       → integer / bigint
exact numeric → numeric
boolean       → boolean
dynamic JSON  → jsonb (when genuinely needed)
```

Do not serialize data that should be structured into arbitrary strings without a real reason.

---

## 8. String Length

`varchar` lengths should come from real business or protocol constraints.

Do not use without evidence:

```sql
varchar(255)
```

for everything.

Codes, names, identity values, URLs, and remarks have different length characteristics and should be designed accordingly.

Use an appropriate type such as `text` for genuinely long text.

---

## 9. NULL

Whether a column allows NULL must have clear meaning.

Do not make everything unconsciously:

```text
NULL
```

and do not avoid NULL by making everything:

```text
NOT NULL DEFAULT ''
```

Distinguish the business meaning of:

```text
NULL
empty string
0
special date
```

For example, “has not exited yet” may be represented clearly as:

```text
lqsj = NULL
```

Do not invent `1970-01-01`, empty strings, or other sentinels to replace an unknown state.

---

## 10. Default Values

A database default must have real business or technical meaning.

Do not casually add values such as:

```text
0
''
1970-01-01
```

merely to avoid NULL.

If a status or time is generated by the database, confirm that this agrees with Java, API, and historical-data semantics.

Do not let Java defaults and database defaults express different business meanings.

---

## 11. Status Columns

Status columns should use stable codes or the project's existing value system.

Do not use UI display text directly as an immutable database protocol unless the project is already intentionally designed that way.

How Java represents status belongs to Java / business-model rules. This document only requires database values to be stable and semantically consistent.

Do not invent new status codes or compatibility aliases.

---

## 12. Time Columns

Time columns should express clear business meaning and reuse existing project terminology.

For example, a project might already use:

```text
lssj → entry/recording time
gxsj → update time
rqsj → entry time
lqsj → exit time
```

Do not introduce long-lived ambiguous names such as:

```text
sj
time
date
```

Cross-time-zone systems must have an explicit database time-zone strategy and application convention.

---

## 13. Money and Exact Numeric Values

Money, ratios, and other values requiring exact calculations should use an exact numeric type such as PostgreSQL `numeric`.

Do not use `real` / `double precision` for data with exact financial semantics.

For corresponding Java precision rules, read the Java coding reference.

---

## 14. Unique Constraints

Data that must be unique by business definition should preferably use a database UNIQUE Constraint as the final consistency safeguard.

Do not rely only on:

```text
SELECT confirms absence
→ INSERT
```

because concurrent requests may race.

For concurrency and transaction handling, read:

- [transactions.md](../architecture/transactions.md)

---

## 15. Foreign-Key Strategy

Whether to use database Foreign Keys follows the target project's established architecture.

If the project consistently avoids physical foreign keys, relationships and integrity-maintenance responsibilities must still be explicit.

Do not change the project's global foreign-key strategy without authorization in a single task.

---

## 16. Indexes

Indexes serve real query and uniqueness needs.

Before adding an index, analyze:

* WHERE;
* JOIN;
* ORDER BY;
* uniqueness;
* query frequency;
* data volume;
* write cost.

Do not add an index merely because a column exists.

---

## 17. Composite Indexes

Composite-index columns and ordering should come from actual query patterns and PostgreSQL execution plans.

For example, if a long-running query pattern is:

```sql
WHERE fzxbh = ?
  AND rqsj >= ?
  AND rqsj < ?
```

then this may be worth evaluating:

```text
(fzxbh, rqsj)
```

but do not mechanically create it based on the example alone.

Execution-plan analysis is governed by `sql.md`.

---

## 18. Do Not Create Duplicate Indexes

Before adding an index, check:

* existing ordinary indexes;
* PRIMARY KEY;
* indexes created by UNIQUE Constraints;
* whether an existing composite index already covers the need.

Do not create indexes with substantially overlapping effects.

---

## 19. Table and Column Comments

New business tables and important columns should have informative comments explaining:

* business meaning;
* status meaning;
* special constraints.

Do not put an entire complex business process into database comments.

---

## 20. Deletion Semantics

Before designing deletion, confirm whether the project uses:

```text
physical deletion
or
logical deletion
```

If logical deletion is used, reuse the established field and semantics.

Codex must not convert DELETE to logical deletion—or the reverse—on its own.

---

## 21. Redundant Columns

Do not add redundant columns casually merely to avoid JOINs.

If redundancy is truly needed, make explicit:

* source of truth;
* update responsibility;
* consistency strategy;
* why the benefit outweighs maintenance cost.

If the maintenance strategy cannot be explained, the redundant column should not be added.

---

## 22. Large Columns

Before placing large text, JSON, binary, or similar data in a frequently accessed primary table, evaluate:

* whether ordinary queries need the data;
* row width and I/O;
* whether a separate table is more appropriate;
* whether object storage is more appropriate.

Do not store file contents directly in an ordinary business table without a real basis.

---

## 23. Schema Compatibility

When modifying an existing Schema, consider:

* historical data;
* online compatibility;
* older application versions;
* Null / default values;
* index-creation cost;
* rollback or recovery strategy.

Without an explicit requirement, do not:

* delete existing columns;
* change the business meaning of existing columns;
* change primary-key semantics;
* casually loosen or tighten critical constraints.

---

## 24. Codex Database-Design Checklist

Before adding or modifying database structure, check:

1. Whether the same business concept and database terminology already exist.
2. Whether naming follows project vocabulary.
3. Whether data types express real semantics.
4. Whether Null and default values have explicit meaning.
5. Whether a primary key, unique constraint, or other integrity constraint is required.
6. Whether indexes come from real queries and do not duplicate existing indexes.
7. Whether necessary table / column comments are present.
8. Whether changes are managed through the formal Migration mechanism.
9. Whether historical data and compatibility are considered.
10. Whether Java mappings must change as well; when needed, load `mybatis.md` rather than duplicating mapping rules here.

Final principle:

> The database-design reference maintains database-side facts only. Java models and persistence mappings are maintained by their dedicated references.
