# Business Rules and Use-Case Boundaries

This document borrows the distinction of Business Rules from *Clean Architecture* and applies it to the Spring Boot layering model used by this Skill Pack.

This document answers:

> Should a business rule belong to stable core business rules or to a specific application use-case flow? When is it worth encapsulating a rule in a behavioral business object, and when should Service / Manager continue to orchestrate it? Do input and output models really need another isolation layer?

Related references:

- [Application Layering and Model Boundaries](layering.md)
- [Project and Business Module Structure](project-structure.md)
- [Java Coding](../coding/java.md)
- [Transactions](transactions.md)

Core principles:

> Business logic itself has layers. The less a rule depends on HTTP, databases, frameworks, and specific entry points—and the more stable it remains across use cases—the closer it is to a core business rule. Application-specific flows organize these rules together with external capabilities.

> Borrow the responsibility analysis, not a directory template. Do not mechanically create `Entity`, `UseCase`, `Repository`, `Command`, `Result`, or additional conversion layers merely to imitate Clean Architecture.

---

## 1. First Distinguish Two Kinds of Business Rules

### 1.1 Core business rules

If the implementation changes from:

```text
HTTP → RPC
Web → automated device
MyBatis → Rabbit-SQL
PostgreSQL → another persistence mechanism
```

and the rule still holds, it is usually closer to the business itself.

Examples:

```text
Only cases pending storage may be stored.
An archived case cannot be stored again.
A cabinet slot with no remaining capacity cannot accept another item.
Loan interest is calculated using a defined business formula.
```

These rules are usually tightly related to the state, invariants, calculations, or allowed behavior of a business concept.

### 1.2 Application use-case rules

If a rule describes:

```text
what the current system must do to complete a user goal,
in what order data is read,
which business capabilities are invoked,
which records are written,
which notifications are sent,
and what result is returned,
```

it is closer to an application use-case flow.

For example, “store a case” may require:

```text
load case
→ load cabinet slot
→ check current slot
→ apply the case-storage rule
→ occupy the slot
→ create a storage record
→ persist changes
→ return the result
```

Principle:

> A core business rule answers “what must this business concept always obey?” An application use case answers “what steps must this application coordinate to achieve this business goal?”

---

## 2. In This Skill Pack, Service Is the Default Use-Case Boundary

This Skill Pack does not require separate `*UseCase` classes.

The default structure may remain:

```text
Controller / other inbound adapter
        ↓
      Service
        ↓
Manager / Mapper / Client
```

The Service itself acts as the application use-case boundary:

```text
Service
→ expresses the current business goal
→ performs business validation
→ coordinates core business rules
→ coordinates Mapper / Client / Manager
→ determines the consistency scope for the current use case
```

Only consider dedicated types such as:

```text
StoreCaseUseCase
ApproveCaseUseCase
```

when the target project already uses a Use Case / Application Service style, or when a single use case has become a stable responsibility that can be independently named, changed, and tested.

Do not merely transform:

```text
PlaceService.store(...)
↓ rename
StorePlaceUseCase.execute(...)
```

when the responsibility and dependencies are otherwise unchanged.

Principle:

> In this Skill Pack, `Use Case` is first a responsibility, not a required class name.

---

## 3. Stable Invariants May Be Encapsulated in Behavioral Business Objects

If the same stable business rule is strongly tied to an object's state and must hold across multiple use cases, consider encapsulating it as behavior instead of repeating caller-side logic such as:

```java
if (caseInfo.getStatus() != CaseStatus.PENDING_STORAGE) {
    throw new BusinessException("The current case cannot be stored");
}
caseInfo.setCabinetId(cabinetId);
caseInfo.setStatus(CaseStatus.STORED);
```

If the project already has a separate behavioral business model, a clearer expression might be:

```java
caseInfo.store(cabinetId);
```

Typical rules worth encapsulating include:

```text
state transitions
object invariants
stable business calculations
behavior constraints that must hold regardless of entry point
```

Do this only when it provides real value.

Do not create a new business object merely because “rich domain models are better” when:

* the feature is simple CRUD;
* the rule appears only in one small use case;
* the new object would create extensive meaningless DO ↔ Domain conversion;
* the target project has no separate domain model and the existing structure is already clear;
* the proposed “behavior” is only a wrapper around getters / setters;
* the invariant cannot be confirmed from requirements, tests, or stable existing code.

Principle:

> Eliminate real duplication and bypassable invariants. Do not create another model layer merely to “eliminate anemic models.”

### 3.1 Shared Domain Semantics May Be Extracted into a Base Class

When multiple domain or behavioral business objects repeatedly carry the same properties, those properties may be extracted into a base class when they represent one stable shared domain concept.

A base class is appropriate when the subclasses have a real `is-a` relationship and the inherited state has the same business meaning, invariants, and lifecycle across those subclasses.

For example, if several concrete business objects are all kinds of the same business concept and consistently share identity and behavior, a domain base type may express that common meaning instead of duplicating it in every subtype.

Do not introduce inheritance merely because several classes happen to contain fields with the same names. Before extracting a base class, confirm:

```text
Do the subclasses represent specializations of the same business concept?
Do the shared properties have the same meaning and lifecycle in every subtype?
Would a rule or invariant defined on the base type be valid for every subtype?
Can callers safely reason about the subtype through the base-type contract?
```

If the answer is mainly “these fields are duplicated,” inheritance is usually too strong a relationship. Prefer keeping the models separate or extracting a value object / composition when that better represents the domain.

In particular, do not create a universal domain superclass merely to centralize technical or persistence metadata such as:

```text
id
createTime
updateTime
deleted
version
```

unless those fields genuinely form part of the shared domain abstraction. Persistence or audit metadata should remain with the boundary that owns those semantics rather than forcing unrelated business concepts into one inheritance hierarchy.

Likewise, Request / Query / DTO / BO / DO / VO models should not inherit from a domain base class merely to reuse fields when their responsibilities and contracts differ.

Principle:

> Extract a domain base class to express a real shared business abstraction, not merely to remove repeated fields. Inheritance models substitutable domain meaning; composition is often better for shared data without a true `is-a` relationship.

---

## 4. A Clean Architecture Entity Is Not the Same as a Persistence DO

The concepts must be distinguished.

A Clean Architecture Entity is closer to:

```text
critical business data
+
critical business rules
```

In this Skill Pack, however:

```text
*DO
<module>.domain
```

still represents a **database persistence model** by default.

Therefore, the presence of the `Entity` concept in this document does not justify mechanically doing any of the following:

```text
add business methods to every DO
interpret <module>.domain as a Clean Architecture Entity package
wrap every table in a rich domain object
add Repository wrappers around existing Mappers
```

If a project truly uses an independent domain model, it may contain:

```text
persistence DO
↔
behavioral business object
```

but the conversion must represent a real difference in responsibility and provide real value.

If no independent domain model exists, keeping stable rules in the correct Service / Manager is still better than introducing a valueless mapping layer for architectural appearance.

Principle:

> `DO` has a persistence responsibility; a Clean Architecture `Entity` has a business-rule responsibility. Similar naming—or a Package named `domain`—does not make them equivalent.

---

## 5. Core Business Objects Do Not Own I/O or Application Flow

Even when the project uses behavioral business objects, do not move all logic into them.

A core business object may express behavior such as:

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
query the database
persist itself
open transactions
send HTTP / RPC requests
invoke third-party SDKs
write to a message queue
send notifications
read the current Web user
construct an HTTP Response
```

Avoid designs such as:

```java
caseInfo.store();
caseInfo.saveDatabase();
caseInfo.sendMessage();
caseInfo.notifyPolice();
```

External collaboration belongs to boundaries such as Service / Manager / Mapper / Client.

Principle:

> Business objects maintain their own rules. The application layer coordinates collaboration among business objects and between those objects and external systems.

---

## 6. Dependencies Flow from Application Flow Toward Core Rules

If the project has independent behavioral business objects, the recommended dependency direction is:

```text
Controller / Consumer / RPC
          ↓
       Service
   (application use case)
          ↓
   core business object
```

The Service may also depend on:

```text
Manager
Mapper / DAO
Client / Adapter
```

A core business object should not depend back on:

```text
Controller
Service / UseCase
Spring MVC
MyBatis / MyBatis-Plus
Rabbit-SQL
PostgreSQL
third-party SDKs
HTTP Request / Response
```

If a supposed “core business object” must know SQL, the current Controller, a Spring Bean, or a vendor response in order to work, technical details have leaked back into the business core and the boundary should be reconsidered.

---

## 7. Service Code Should Read Like a Business Flow, Not a Database Script

For a complex use case, high-level Service code should preferably express:

```text
load required business objects
→ check conditions required by the use case
→ invoke stable business behavior
→ coordinate other objects or external capabilities
→ persist results
→ build the business output
```

Conceptually:

```java
CaseDO caseDO = caseMapper.getById(caseId);
CabinetDO cabinetDO = cabinetMapper.getById(cabinetId);

// Depending on whether the project has a separate behavioral model,
// invoke business behavior or enforce the rule in the application layer.

storageRecordMapper.insert(record);
```

The point is not that `Case` / `Cabinet` Entity classes must exist. The point is to avoid degrading Service code into:

```text
query table
→ if magic status
→ set field
→ update
→ query another table
→ assemble protocol object
```

while hiding the actual business meaning inside database field manipulation.

---

## 8. Rules That Depend on Current Database State Still Require Application-Layer Coordination

Not every “business rule” belongs inside one object.

Examples:

```text
whether a code is unique
whether the current caller has permission
whether a cabinet slot currently has enough remaining capacity
whether an unfinished record already exists in the database
whether two objects must be updated atomically
```

These rules depend on:

```text
current database state
other objects
caller context
transaction consistency
```

They should be coordinated by Service / Manager within the correct transaction boundary, while object-local rules can still be invoked as needed.

Do not make an Entity access Mapper / Repository / Client merely to keep the Entity “pure.”

Continue to use `transactions.md` to determine transaction scope.

---

## 9. Isolate Input and Output Models by Meaning, Not by Layer Count

Clean Architecture encourages decoupling use-case input/output from external protocol models. That direction can be useful, but this Skill Pack does not require a fixed chain such as:

```text
Request
→ Command
→ DTO
→ UseCase
→ Result
→ VO
```

Before adding an application input or output model, first ask whether a real semantic difference exists.

Separate models are useful when:

* HTTP / RPC / Message entry points reuse the same application use case;
* an external Request contains protocol fields the application layer should not know about;
* trusted server-side context must not come from the client Request;
* a use case needs a stable internal input contract that clearly differs from the current interface model;
* the public VO differs materially from a core business object in data or lifecycle.

An extra conversion layer is usually unnecessary when:

* Request / Query is already clear business input data and leaks no protocol objects;
* there is only one entry point and Command would have exactly the same fields and meaning as Request;
* Result and VO are merely field-for-field copies;
* conversion adds boilerplate but isolates no source of change.

Concrete business output should still preferably use VO. Neither DO nor core business objects should be exposed directly through a public HTTP API.

Principle:

> Boundary models arise from semantic differences, not from a rule that every layer transition requires a new object.

---

## 10. Decision Flow for Rule Placement

When you encounter a business decision, ask in this order:

```text
Would this rule still hold if the HTTP / UI / database implementation changed?
        ↓
No → it is more likely a protocol, application-flow, or technical rule
        ↓
Yes
        ↓
Is it a state transition, invariant, or stable calculation of one business concept?
        ↓
Yes → if there is real reuse / anti-bypass value, consider encapsulating it in a behavioral business object
        ↓
No
        ↓
Does it depend on current database state, permissions, multiple objects, an external system, or a transaction?
        ↓
Yes → coordinate it in Service / Manager
        ↓
No → place it according to the target project's existing responsibilities; do not create a layer merely for classification
```

Then check:

```text
Is the same rule already duplicated across entry points / use cases?
Can callers bypass it and directly set state?
Would extraction make the business intent clearer?
Would extraction introduce valueless mapping or another layer?
```

---

## 11. An “Anemic Model” Is Not a Defect by Itself

None of the following shapes alone proves an architectural problem:

```text
DO contains only fields
Service contains business checks
there is no Entity class
there is no UseCase class
there is no Repository interface
```

Adjustment is warranted only when a concrete risk exists, for example:

```text
the same state invariant is duplicated in multiple Services and the semantics have already drifted
multiple callers can bypass a critical state check and directly mutate the object
one Service simultaneously owns business flow, protocol conversion, SQL, third-party SDK handling, and state rules
a core business object depends back on Spring / Mapper / Client
an external API exposes persistence or core internal models directly, allowing external requirements to pollute internal boundaries
```

Principle:

> Evaluate real responsibilities and change risks. Do not score the design according to “anemic model vs. rich domain model” ideology.

---

## 12. Codex Implementation Checklist

When business rules, state transitions, or domain objects are involved, check:

1. The current rule comes from explicit requirements, an existing contract, tests, or stable implementation—not from an Agent inventing behavior.
2. Core business rules and application use-case flows were distinguished first.
3. The same stable invariant is not duplicated across multiple entry points / Services.
4. If a behavioral object is extracted, it truly encapsulates a state transition, invariant, or stable calculation rather than wrapping setters.
5. If common domain properties are extracted into a base class, the subclasses share one stable business abstraction and a real `is-a` / substitutability relationship; field duplication alone is not sufficient.
6. A persistence DO is not incorrectly treated as a Clean Architecture Entity.
7. Service still expresses the business use case rather than degrading into protocol or SQL scripting.
8. Core business objects remain unaware of Spring, persistence frameworks, HTTP, and vendor SDKs.
9. Database state, permissions, multi-object collaboration, and transaction rules are still coordinated correctly by Service / Manager.
10. `Entity / UseCase / Repository / Command / Result` types are not added for form alone without a responsibility benefit.
11. Request / Query / DTO / BO / DO / VO conversions each represent a real semantic change.
12. If a core rule is extracted, corresponding unit tests are added or adjusted; if an application flow changes, relevant use-case tests cover it.
13. The target project's stable structure is preserved rather than using the task as an excuse for a wholesale architectural migration.

Final principle:

> First identify the most stable business rules, then identify how the current application coordinates them. Protect business meaning from Web, database, and framework details—but isolate only real sources of change, and do not mechanically add architectural layers.
