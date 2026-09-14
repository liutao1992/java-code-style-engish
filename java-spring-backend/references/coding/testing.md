# Testing Standard

This document defines project testing standards.

It draws on unit-testing guidance from the *Alibaba Java Development Manual* and adapts it to modern Java, Spring Boot, MyBatis, and PostgreSQL project practices.

Core principles:

> Tests verify business behavior, boundaries, and failure semantics. Do not test implementation details merely to increase coverage, Mock count, or test count.

> Good tests should be automated, independent, and repeatable, and should fail reliably when code behavior changes incorrectly.

---

## 1. Behavior Changes Need Tests

Prioritize adding or updating tests for:

* new business functionality;
* changes to business rules;
* changes to state transitions;
* Bug fixes;
* critical SQL changes;
* authorization, tenant, or data-scope changes;
* transaction, consistency, or concurrency changes;
* important API-contract changes;
* exception mapping or critical failure-semantic changes.

Do not mechanically add tests for every trivial line of code. Testing effort should prioritize business risk and regression risk.

---

## 2. AIR Principle

Unit tests should follow AIR:

```text
A — Automatic
I — Independent
R — Repeatable
```

### 2.1 Automatic

Tests must be executable automatically by the build system and CI.

Do not judge results manually through:

```java
System.out.println(...)
```

or:

```text
run test
→ inspect console manually
→ decide manually whether it is correct
```

Use explicit assertions.

Tests should not require a person to:

* enter data;
* click a page;
* modify a database;
* start an ad-hoc local service;
* inspect logs and decide whether the test passed.

If a test genuinely depends on an external environment, classify it explicitly as an integration or end-to-end test and manage it through the project's existing test infrastructure.

---

### 2.2 Independent

Tests must not depend on execution order.

Do not design:

```text
testCreate()
   ↓ creates data

testUpdate()
   ↓ depends on testCreate

testDelete()
```

Do not call one `@Test` method directly from another merely to reuse setup logic.

Each test should prepare its own preconditions and ensure any necessary isolation and cleanup.

Avoid shared mutable static test state such as:

```java
static List<String> sharedData = new ArrayList<>();
```

that allows tests to contaminate one another.

---

### 2.3 Repeatable

The same test, run repeatedly against the same code in a controlled environment, should produce the same result.

Do not depend on:

* the accidental current system time;
* accidental random values;
* uncontrolled external networks;
* machine-specific absolute paths;
* test execution order;
* data left behind by a previous test;
* hand-maintained data in a developer's local database;
* whether a third-party service happens to be online.

Control unstable factors such as time, randomness, and external systems through dependency injection, Fixture, Mock, Stub, Testcontainers, or the project's existing test infrastructure.

---

## 3. BCDE Test-Design Principle

When designing test scenarios, consider at least BCDE:

```text
B — Border
C — Correct
D — Design
E — Error
```

### 3.1 Border

Check boundaries that actually exist in the current business contract, for example:

* minimum / maximum values;
* empty collections;
* single-element collections;
* first / last pagination page;
* time start / end boundaries;
* crossing day / month / year;
* NULL;
* duplicate values;
* data ordering;
* batch-size boundaries;
* boundaries before / after state transitions.

Do not invent a business threshold merely to claim boundary coverage.

---

### 3.2 Correct

At minimum, verify that core valid input produces the expected business result.

Do not prove only:

```text
no exception was thrown
```

When relevant, assert the real result, such as:

* return values;
* state changes;
* data writes;
* call conditions;
* important side effects;
* external responses.

---

### 3.3 Design — Business Design and Contracts

Tests should come from real design and business rules, not be reverse-engineered from the current implementation.

For example, if a requirement states:

```text
only PENDING status may be audited
```

tests should validate that contract rather than merely checking how many times a private method was invoked.

Existing:

* API contracts;
* state machines;
* database constraints;
* authorization rules;
* transaction constraints;

are all valid sources for test scenarios.

---

### 3.4 Error

Consider real failure scenarios, for example:

* invalid input;
* resource not found;
* disallowed state;
* insufficient permission;
* unique-constraint conflicts;
* external-dependency failure;
* unexpected affected-row count;
* concurrency conflicts;
* required exception propagation and translation.

Error tests should validate the expected failure semantics. Do not settle for:

```java
assertThrows(Exception.class, ...);
```

when the project's exception contract is clear. Prefer the precise exception type, error code, or key business result when possible.

---

## 4. Bug Fixes

For a Bug fix, when practical use this sequence:

```text
reproduce the Bug
    ↓
add a failing test
    ↓
confirm the test fails reliably
    ↓
fix the Bug
    ↓
test passes
```

The test should lock in the behavioral root cause rather than merely cover one implementation branch.

Avoid:

```text
fix implementation
→ change the old test until it passes
```

without proving the original problem cannot recur.

---

## 5. Test Directories and Organization

For an ordinary Maven / Gradle Java project, the defaults are:

```text
src/main/java
→ production code

src/test/java
→ test code

src/test/resources
→ test resources
```

Test-class Packages normally mirror the production code to simplify navigation.

Do not put unit tests under the production source directory.

If the target project uses different source sets, modular test directories, or a dedicated integration-test structure, follow the existing build configuration rather than migrating it to these defaults.

---

## 6. Test Granularity

Keep unit tests small enough that failures are easy to locate.

A unit test normally focuses on:

```text
one class
one business capability
one explicit behavior
```

Do not make a unit test validate an entire cross-system path.

If a test needs to validate all of:

```text
Spring container
+
database
+
multiple Beans
+
real SQL
```

it is closer to an integration test. Design it accordingly instead of forcing it into the label “unit test” through excessive mocking.

Principle:

> Use unit tests for fast local business behavior and integration tests for framework, database, and component collaboration.

---

## 7. Test Naming

Test names should describe the business behavior and its condition.

Recommended:

```java
shouldRejectAuditWhenPlaceAlreadyRejected()
```

or:

```java
audit_shouldRejectWhenPlaceAlreadyRejected()
```

Keep the convention consistent within a module.

Avoid:

```java
test1()
testAudit()
testMethod()
```

A failing test name should reveal as much as possible about:

```text
what behavior failed
+
under what condition
```

---

## 8. AAA Structure

For complex tests, prefer:

```text
Arrange
Act
Assert
```

For example:

```java
@Test
void shouldRejectAuditWhenPlaceAlreadyRejected() {
    // Arrange
    Place place = rejectedPlace();

    // Act + Assert
    assertThrows(
            PlaceStatusException.class,
            () -> place.audit(APPROVED));
}
```

AAA exists to make test structure clear. Do not mechanically add three comments to every trivial test.

If Arrange is much larger than Act / Assert, inspect whether the Fixture is too heavy, the production code has too many responsibilities, or the test granularity is wrong.

---

## 9. What Unit Tests Should Focus On

Unit tests primarily verify:

* business rules;
* state changes;
* parameter boundaries;
* exception branches;
* important algorithms;
* branch selection;
* dependency invocation conditions;
* return values and important side effects.

Usually there is little value in testing:

* Java getters / setters;
* ordinary Lombok-generated accessors;
* behavior already guaranteed by the JDK;
* basic framework behavior already well-covered by Spring / MyBatis;
* simple field copying with no business value;
* unconditional one-line forwarding methods.

Test value matters more than test count.

---

## 10. Mock and Test Doubles

Mock only dependencies outside the current test scope.

For example, when testing a Service, appropriate mocks may include:

```text
Mapper
external HTTP Client
message sender
object-storage Client
```

Do not Mock the object under test merely to make the test pass.

Avoid over-mocking until a test proves only:

```text
called A
then called B
then called C
```

without asserting the business result.

Interaction assertions are useful when the collaboration itself has business meaning, for example:

```text
when the state is invalid, UPDATE must not run
when validation fails, the external system must not be called
after success, a required message must be sent exactly once
```

Do not mechanically `verify()` every internal call order.

---

## 11. Design for Testability

When code is difficult to test, inspect the design before resorting to unusual testing techniques.

Testability can be improved with:

* constructor dependency injection;
* `Clock`;
* explicit external Client boundaries;
* small, clear business methods;
* replaceable external dependencies.

Do not change a method that should be `private` to `public` solely for testing.

Do not default to reflection-based direct testing of private methods.

If a private method contains a large amount of independently testable business logic, reconsider whether that reveals too much responsibility or a missing abstraction.

Principle:

> Prefer testing observable business behavior. If testing requires breaking encapsulation, inspect the design first.

---

## 12. Service Tests

Service tests focus on:

* business flow;
* business validation;
* invocation conditions for Manager / Mapper / Client;
* exception branches;
* business results;
* side effects that must not occur.

For example:

```text
state not allowed
→ throw business exception
→ do not execute Mapper update
```

Mock external dependencies for a pure unit test where appropriate.

Use integration tests for complex database behavior.

---

## 13. Controller / API Tests

Controller tests focus on the interface boundary:

* HTTP routing;
* parameter binding;
* parameter validation;
* HTTP status;
* project-standard response format;
* exception mapping;
* authorization boundary;
* serialization contract.

Do not repeat the full Service business test suite in Controller tests.

The target project may use `ApiResponse<T>` as a unified response by default, but the actual project contract takes precedence.

---

## 14. Mapper / SQL Tests

For complex SQL, consider database integration tests.

Especially for:

* PostgreSQL-specific SQL;
* dynamic SQL;
* multi-table JOINs;
* time ranges;
* NULL;
* aggregation;
* pagination;
* unique constraints;
* conditional UPDATE;
* lock-related queries;
* ResultMap / TypeHandler mapping.

Do not prove SQL correctness by mocking the Mapper.

A Mock Mapper can only prove that an upper layer invoked a Mapper under a condition. It cannot validate:

```text
SQL syntax
SQL results
database constraints
actual mapping
```

---

## 15. Database Test Data

Database tests must not depend on data manually inserted by a developer in advance.

Do not use:

```text
open database manually before the test
→ insert a row
→ run the test
```

Prepare test data repeatably through methods such as:

* inserts from test code;
* SQL Fixture;
* test-data Builder;
* Flyway / Liquibase test migrations;
* Testcontainers initialization scripts;
* the project's existing test-data tools.

Test data must respect real database constraints and business preconditions. Do not bypass normal rules to create states that cannot exist in production unless the test specifically verifies behavior with corrupted / abnormal data.

---

## 16. Database Cleanup and Isolation

A test should not leave uncontrolled dirty data in a shared test environment.

Depending on the project's test architecture, use:

* transaction rollback;
* cleanup after the test;
* per-test schema / database;
* Testcontainers environment recreation;
* unique test-data prefixes / IDs.

Do not mechanically use transaction rollback for every database test.

If the test is specifically validating:

* commit;
* rollback;
* cross-transaction visibility;
* locking;
* isolation level;

a framework-level automatic rollback can hide the behavior under test. Choose cleanup according to the test goal.

---

## 17. PostgreSQL Tests

Database tests should use the same database type as production whenever practical.

If the project uses PostgreSQL, do not default to H2 for PostgreSQL-specific behavior.

When SQL depends on PostgreSQL-specific:

```text
syntax
types
JSON / JSONB
arrays
window functions
locks
isolation
index behavior
RETURNING
```

prefer a real PostgreSQL test environment or Testcontainers.

---

## 18. Time, Randomness, and Environment Variables

For time-dependent logic, avoid uncontrolled direct dependence on:

```java
LocalDateTime.now()
Instant.now()
```

when it makes tests vary with execution time.

For complex time logic, prefer injecting:

```java
Clock
```

or using the project's existing time abstraction.

When random values are involved, ensure the test can control or seed the random source.

When behavior depends on configuration, environment variables, time zone, or Locale, set them explicitly in the test or through project test configuration rather than relying on a developer machine's defaults.

---

## 19. Concurrency Tests

Do not attempt to prove concurrency correctness solely through:

```java
Thread.sleep(...)
```

Concurrency tests should verify as appropriate:

* final data results;
* concurrency conflicts;
* affected-row counts;
* optimistic locking;
* unique constraints;
* duplicate execution;
* necessary thread coordination;
* timeouts and failure paths.

Prefer explicit synchronization points with:

* `CountDownLatch`;
* Barrier;
* Future;
* the project's existing concurrency-test tools;

instead of relying on “sleep long enough and it should finish.”

For concurrency rules, read:

- [concurrency.md](../architecture/concurrency.md)

---

## 20. Transaction Tests

Transaction tests should verify final data state and consistency, not merely assert the presence of:

```java
@Transactional
```

Depending on requirements, test:

* successful commit;
* rollback on failure;
* multi-table consistency;
* conditional updates;
* required isolation and lock behavior;
* cross-transaction visibility.

Do not prove transaction behavior with Mock Mappers.

For transaction rules, read:

- [transactions.md](../architecture/transactions.md)

---

## 21. Test Data and Fixtures

Test data should be:

* minimal;
* semantically clear;
* directly relevant to the current scenario;
* explicit about defaults;
* structured so that when a field is changed, the reason that field matters is visible.

Avoid constructing dozens of irrelevant fields in every test.

Use techniques such as:

```text
Fixture
Builder
Object Mother
Factory Method
```

when they improve readability.

Do not turn test Fixtures into another complicated business framework.

---

## 22. Assertions

Use explicit assertions.

Do not rely on:

```java
System.out.println(...)
```

and manual inspection.

Assertions should focus on important business results, such as:

```text
return value
critical fields
exception type
error code
final database state
important dependency called / not called
```

Do not increase assertion count by checking many fields unrelated to the current behavior.

Failure messages should help developers understand the business deviation quickly.

---

## 23. Coverage

Coverage is an aid for discovering test blind spots, not a measure of test quality by itself.

Do not increase coverage merely by:

* testing getters / setters;
* testing branches with no business value;
* invoking private methods through reflection;
* writing tests with no business assertions;
* excluding important but difficult code from coverage.

The *Alibaba Java Development Manual* includes general coverage guidance, but this Skill Pack does not impose a universal hard threshold such as:

```text
70%
100%
```

If the target project already enforces coverage through:

```text
JaCoCo
SonarQube
CI Quality Gate
```

follow the existing gate and do not lower it without authorization.

Even without a coverage gate, prioritize:

* core business logic;
* high-risk branches;
* Bug regressions;
* authorization and security boundaries;
* transaction consistency;
* database constraints;
* concurrency conflicts.

Principle:

> Aim for reliable verification of critical behavior, not an attractive percentage.

---

## 24. Do Not Change Tests to Accommodate an Incorrect Implementation

When an existing test fails, first determine:

```text
Is the implementation wrong?
or
Did the requirement genuinely change?
```

Only change an existing assertion when the requirement, contract, or business rule clearly changed.

Do not make an incorrect implementation “green” by:

* deleting the failing test;
* commenting out the failing test;
* weakening assertions;
* replacing a precise exception with `Exception.class`;
* adding meaningless waits;
* introducing execution-order dependencies;
* broadening Mock behavior merely to make the test pass.

---

## 25. Test Code Also Needs Quality

Test code is maintained code. Being under `src/test/java` does not remove basic quality requirements.

Avoid:

* excessive copy/paste;
* giant test methods;
* unclear test data;
* magic numbers whose business meaning cannot be explained;
* multiple behaviors packed into one test;
* failures that provide no clue about the reason.

At the same time, do not over-abstract tests into an unreadable generic testing framework merely to satisfy DRY.

Moderate repetition in tests is acceptable when it improves scenario independence and readability.

Principle:

> Production code prioritizes removing business duplication; test code prioritizes scenario clarity. Choose repetition vs. abstraction according to readability.

---

## 26. Codex Testing Checklist

After modifying code, determine:

1. whether existing behavior changed;
2. whether the task adds a business capability or fixes a Bug;
3. whether important BCDE boundary, correct, design, and error scenarios exist;
4. whether SQL, transactions, authorization, exceptions, or concurrency are involved;
5. whether existing tests already cover the change;
6. whether new tests satisfy AIR: automatic, independent, repeatable;
7. whether tests incorrectly depend on manual database data, external networks, or execution order;
8. whether the scenario needs a unit test or actually requires a database / Spring integration test;
9. whether Mock isolates only dependencies outside the current test scope;
10. whether tests verify real business results rather than only invocation counts;
11. whether testing has broken encapsulation or production design;
12. whether Null, boundaries, exceptions, rollback, or concurrency conflicts need coverage;
13. whether database tests run against an environment that can validate real PostgreSQL behavior;
14. whether low-value tests are being added only for coverage, or an existing quality gate is being weakened.

After running tests, report truthfully:

```text
which tests were run
which passed
which failed
which were not run
why they were not run
```

Never claim a test passed if it was not actually executed.

Final principle:

> AIR keeps tests reliably executable; BCDE helps design complete scenarios. Tests should protect real business contracts and critical boundaries, not serve a coverage percentage or current implementation details.
