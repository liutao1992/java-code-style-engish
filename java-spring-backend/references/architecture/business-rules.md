# Business Rules and Use-Case Boundaries

This document defines how to distinguish **stable core business rules** from **application-specific use-case rules**, and when business behavior should be encapsulated in a behavioral business object instead of remaining in Service / Manager orchestration.

It answers:

> What kind of business rule is this, where should the rule live conceptually, and does introducing a behavioral domain object or extra application model create real value?

It does **not** redefine Controller / Service / Manager / Mapper / Client responsibilities or model Package placement; use [layering.md](layering.md) for those. It does not define physical project directories; use [project-structure.md](project-structure.md).

Related references:

- [Application layering and model boundaries](layering.md)
- [Project and business-module structure](project-structure.md)
- [Java coding](../coding/java.md)
- [Transactions](transactions.md)

Core principles:

> The less a rule depends on HTTP, databases, frameworks, and one specific entry point—and the more stable it remains across use cases—the closer it is to a core business rule.

> Borrow responsibility concepts from Clean Architecture, not a mandatory directory template. Do not mechanically create `Entity`, `UseCase`, `Repository`, `Command`, `Result`, or conversion layers.

---

## 1. Distinguish Core Business Rules from Application Use-Case Rules

### 1.1 Core business rules

If the implementation changes from:

```text
HTTP → RPC
Web → automated device
MyBatis → Rabbit-SQL
PostgreSQL → another persistence mechanism
```

and the rule still holds, it is usually close to the business itself.

Examples:

```text
Only cases pending storage may be stored.
An archived case cannot be stored again.
A cabinet slot with no remaining capacity cannot accept another item.
Loan interest follows a defined business formula.
```

These rules are usually state invariants, calculations, transitions, or behavior constraints of a business concept.

### 1.2 Application use-case rules

A rule is closer to an application use case when it describes how the current system coordinates work to achieve a user goal, for example:

```text
load case
→ load cabinet slot
→ check current state
→ apply the storage rule
→ occupy the slot
→ create the storage record
→ persist changes
→ return the result
```

Principle:

> A core business rule answers “what must this business concept always obey?” A use-case rule answers “what must this application coordinate to achieve this goal?”

---

## 2. Service Is the Default Use-Case Boundary

This Skill Pack does not require separate `*UseCase` classes. By default, Service is the application use-case boundary.

Detailed Service responsibilities, dependencies, naming, and Package placement are defined in [layering.md](layering.md). This document only decides whether a rule belongs to application orchestration or to a more stable business abstraction.

Consider dedicated types such as:

```text
StoreCaseUseCase
ApproveCaseUseCase
```

only when the target project already uses that style or when a use case has become an independently nameable and independently changing responsibility.

Do not merely rename:

```text
PlaceService.store(...)
→ StorePlaceUseCase.execute(...)
```

when no responsibility boundary actually changes.

Principle:

> `Use Case` is first a responsibility, not a required class name.

---

## 3. Stable Invariants May Belong in Behavioral Business Objects

When the same stable business rule is strongly tied to an object's state and must hold across multiple use cases, consider encapsulating it as behavior instead of repeating caller-side field manipulation.

Repeated caller-side logic such as:

```java
if (caseInfo.getStatus() != CaseStatus.PENDING_STORAGE) {
    throw new BusinessException("The current case cannot be stored");
}
caseInfo.setCabinetId(cabinetId);
caseInfo.setStatus(CaseStatus.STORED);
```

may justify a behavioral expression such as:

```java
caseInfo.store(cabinetId);
```

when the target project has a meaningful independent business model and the invariant is real and stable.

Typical candidates include:

```text
state transitions
object invariants
stable business calculations
behavior constraints that apply across entry points
```

Do not introduce a new behavioral model merely because “rich domain models are better” when:

- the feature is simple CRUD;
- the rule occurs only in one small use case;
- the new object creates valueless DO ↔ Domain conversion;
- the target project has no separate domain model and existing responsibilities are clear;
- the proposed behavior only wraps getters/setters;
- the invariant cannot be confirmed from requirements, tests, or stable code.

Principle:

> Encapsulate stable meaning and anti-bypass invariants, not architecture fashion.

---

## 4. Shared Domain Semantics May Be Extracted into a Base Class

When multiple **behavioral domain objects** repeatedly carry the same properties, a base class may be appropriate only when those properties express one stable shared domain abstraction.

Before extracting inheritance, confirm:

```text
Do the subclasses represent specializations of the same business concept?
Do the shared properties have the same business meaning and lifecycle?
Would invariants on the base type be valid for every subtype?
Can callers safely reason about each subtype through the base contract?
```

If the main justification is only “these fields are duplicated,” inheritance is usually too strong. Keep the models separate or prefer composition/value objects when that better represents the domain.

Do not create a universal domain superclass merely to centralize technical or persistence metadata such as:

```text
id
createTime
updateTime
deleted
version
```

unless those fields genuinely participate in the shared domain abstraction.

Likewise, Request / Query / DTO / BO / DO / VO models should not inherit from a domain base class merely to reuse fields across different contracts and responsibilities.

Principle:

> Domain inheritance expresses substitutable shared business meaning, not repeated storage fields.

### 4.1 Similar Structure Does Not Imply a Shared Abstraction

Before extracting a base class, shared DTO-like model, converter, helper, or common utility from similar code, determine whether the duplicated parts have the same semantics and the same reason to change.

For example, these models may currently contain identical fields:

```text
PlaceResponse
PlaceExportRow
PlaceDTO
```

but their contracts may change independently because one follows an external API, one follows an export format, and one follows an internal application transfer need.

Keep structurally similar code separate when the similarity is accidental and the responsibilities or change drivers differ. Extract shared code only when the shared meaning is stable enough that changing the abstraction should correctly affect all consumers.

Do not use DRY as a reason to couple independent contracts.

Principle:

> Similar code is not automatically the same responsibility; reuse stable semantics, not appearance alone.

---

## 5. A Clean Architecture Entity Is Not a Persistence DO

A Clean Architecture Entity is conceptually close to:

```text
critical business data
+
critical business rules
```

In this Skill Pack, `*DO` under the default `<module>.domain` responsibility Package is still a **database persistence model**. Its classification and Package semantics are owned by [layering.md](layering.md).

Therefore, Clean Architecture terminology does not justify:

```text
adding business methods to every DO
interpreting <module>.domain as an Entity package
wrapping every table in a rich domain object
adding Repository wrappers around existing Mappers
```

If an independent behavioral model exists, persistence DO ↔ behavioral-object conversion must represent a real responsibility difference.

Principle:

> A persistence DO and a business-rule Entity are different concepts even when naming or directories look similar.

---

## 6. Core Business Objects Do Not Own I/O or Application Flow

A core business object may own behavior such as:

```text
store
archive
borrow
approveTransition
calculateAmount
ensureAvailable
```

It should not itself:

```text
query or persist through Mapper / DAO
open application transactions
send HTTP / RPC requests
invoke vendor SDKs
write messages
send notifications
read the current Web user
construct HTTP responses
```

External collaboration remains application-layer orchestration. Exact Service / Manager / Mapper / Client dependency rules are defined by [layering.md](layering.md).

Principle:

> Business objects maintain their own stable rules; the application coordinates I/O and collaboration around them.

---

## 7. Rules That Depend on External State Need Application Coordination

Not every business rule belongs inside one object.

Examples include:

```text
whether a code is unique
whether the caller has permission
whether a cabinet slot currently has capacity
whether an unfinished record already exists
whether multiple writes must succeed atomically
```

These depend on current database state, caller context, other objects, external systems, or transaction consistency. Coordinate them in the application layer using the responsibilities defined by `layering.md`.

Do not make a business object access Mapper / Client merely to force all business logic into the object.

Use `transactions.md` to determine consistency and transaction scope.

---

## 8. Service Code Should Expose Business Meaning

For a complex use case, high-level application code should make the business sequence understandable:

```text
load required state
→ check use-case conditions
→ invoke stable business behavior
→ coordinate other capabilities
→ persist results
→ build business output
```

The goal is not to require Entity classes. The goal is to avoid hiding business meaning inside a long database-field script such as:

```text
query table
→ compare magic status
→ set fields
→ update
→ query another table
→ assemble protocol object
```

Whether the rule is expressed by a behavioral object or remains in Service / Manager depends on stability, reuse, anti-bypass value, and project conventions.

---

## 9. Add Input/Output Isolation Only for Real Semantic Differences

This section answers only **whether an extra application input/output model is justified**. The meanings and default Packages of Request / Query / DTO / BO / DO / VO remain defined by [layering.md](layering.md).

Do not require a fixed chain such as:

```text
Request
→ Command
→ DTO
→ UseCase
→ Result
→ VO
```

Separate application models are useful when:

- multiple protocols reuse the same use case;
- an external Request contains protocol fields the application layer should not know;
- trusted server-side context must be separated from client input;
- the use case needs a stable internal contract that materially differs from the external one;
- public output differs materially from the internal business object.

Extra conversion is usually unnecessary when:

- Request / Query already expresses the needed business input without protocol leakage;
- Command would have exactly the same fields and meaning;
- Result and VO are field-for-field copies;
- the conversion layer isolates no real source of change.

Principle:

> Add a boundary model because the semantics differ, not because every layer transition needs another class.

---

## 10. Decision Flow for Rule Placement

When a business decision appears, ask:

```text
Would the rule still hold if HTTP / UI / persistence technology changed?
        ↓
No → likely protocol, application-flow, or technical rule
        ↓
Yes
        ↓
Is it a state transition, invariant, or stable calculation of one business concept?
        ↓
Yes → if reuse / anti-bypass value is real, consider a behavioral business object
        ↓
No
        ↓
Does it depend on database state, permissions, multiple objects, external systems, or a transaction?
        ↓
Yes → coordinate it in the application layer
        ↓
No → follow the target project's existing responsibility model; do not add a layer merely for classification
```

Then check:

```text
Is the rule duplicated across entry points or use cases?
Does duplicated code share the same semantics and reason to change, or only a similar shape?
Can callers bypass it and mutate state directly?
Would extraction make business intent clearer?
Would extraction couple contracts that should evolve independently?
Would extraction introduce valueless conversion or another ceremonial layer?
```

---

## 11. An “Anemic Model” Is Not a Defect by Itself

None of these alone proves an architectural problem:

```text
DO contains only fields
Service contains business checks
there is no Entity class
there is no UseCase class
there is no Repository interface
```

Adjustment is warranted only when there is a concrete risk, for example:

- the same invariant is duplicated and drifting across Services;
- callers can bypass a critical state check;
- protocol, SQL, vendor integration, and business rules are mixed into one use-case class;
- a supposed core business object depends on framework or persistence details;
- external APIs expose persistence or internal core objects directly and thereby pollute internal contracts.

Principle:

> Evaluate responsibilities and change risk, not anemic-model versus rich-domain-model ideology.

---

## 12. Business-Rule Checklist

When business rules, state transitions, or behavioral domain objects are involved, check:

1. The rule comes from explicit requirements, contracts, tests, or stable implementation rather than invention.
2. Core business rules and application use-case rules were distinguished first.
3. Stable invariants are not duplicated across multiple entry points without reason.
4. A behavioral object, when introduced, encapsulates real invariant/transition/calculation meaning rather than setters.
5. A domain base class represents a true shared business abstraction and `is-a` relationship; field duplication alone is insufficient.
6. Similar fields or implementation shape were not treated as proof of one shared abstraction; extracted code has shared semantics and a common reason to change.
7. Persistence DO is not confused with a Clean Architecture Entity.
8. Core business objects remain unaware of HTTP, Spring, persistence frameworks, and vendor SDKs.
9. Database state, permissions, multi-object coordination, and transaction rules remain application-layer responsibilities.
10. Extra `Entity / UseCase / Repository / Command / Result` types each have a real semantic responsibility.
11. Input/output conversion layers isolate an actual semantic or change boundary.
12. Relevant tests are added or adjusted according to `testing.md` when behavior changes.
13. The target project's stable architecture is preserved instead of being migrated wholesale.

Final principle:

> Identify stable business meaning first, then application coordination. Encapsulate rules when that prevents drift or bypass; otherwise prefer the simplest structure that preserves clear responsibilities.
