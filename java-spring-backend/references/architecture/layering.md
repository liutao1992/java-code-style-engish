# Application Layering Standard

This document defines **logical application responsibilities**, dependency direction, model classification, responsibility Packages, and SOLID-related boundaries.

It answers:

> What is this class logically responsible for, what may it depend on, and which responsibility Package should contain it?

It does **not** define the project's physical module/root-directory layout; use [project-structure.md](project-structure.md) for that. It also does not decide whether a business rule is a stable core invariant or an application use-case rule; use [business-rules.md](business-rules.md) for that.

Java implementation, Spring annotations, HTTP contracts, exceptions, SQL, transactions, concurrency, and testing are maintained by their dedicated references.

Core principle:

> Determine logical responsibility first, then responsibility Package, then use `project-structure.md` to locate that Package physically.

---

## 1. Default Logical Layers

```text
                    Inbound Adapters
       ┌──────────────┼──────────────┐
 Controller / Web   RPC / Open API   Consumer / Scheduled Task
       └──────────────┼──────────────┘
                      ↓
                   Service
                      ↓
                 Manager (optional)
                 ↙            ↘
          Mapper / DAO      Client / Adapter
               ↓                  ↓
            Database      Third-party / External System
```

Logical responsibilities:

```text
Inbound adapter
→ accept an external protocol and enter the application

Service
→ express a business use case / application flow

Manager (optional)
→ reusable application capability, composition, or atomic operation

Mapper / DAO
→ access the database

Client / Adapter
→ access and adapt external technical systems
```

Simple flows may skip optional layers:

```text
Controller → Service → Mapper
Controller → Service → Client
```

Do not create a Manager, Facade, Repository wrapper, or other layer merely to make the diagram complete.

---

## 2. Inbound and Outbound Boundaries

Typical inbound adapters include:

```text
HTTP Controller
RPC Endpoint
Open API Endpoint
Message Consumer
Scheduled Task
Command / Job Handler
```

They may receive external input, perform protocol-level binding and structural validation, obtain trusted caller context, invoke a Service / Facade, and convert application output to the required protocol response.

Different inbound adapters should not call one another merely to reuse business logic:

```text
HTTP Controller ───┐
RPC Endpoint ──────┼→ PlaceManageService
Consumer ──────────┘
```

Typical outbound adapters include:

```text
Mapper / DAO
HTTP Client
RPC Client
SDK Adapter
Object Storage Client
Message Producer
External Data Client
```

Database access and external-system access are separate outbound responsibilities. Do not force HTTP, RPC, SDK, object storage, or message sending into Mapper / DAO.

Service / Manager should normally not depend directly on protocol-specific types such as:

```text
HttpServletRequest / HttpServletResponse / ResponseEntity
RPC framework Request / Context
messaging middleware Record / Message
vendor SDK Request / Response / Exception
database physical column names
```

Convert volatile protocol details at the appropriate boundary.

---

## 3. SOLID and Simple Design

Use SOLID to identify concrete responsibility and dependency problems, not to manufacture abstractions.

### SRP

A class should center on one primary responsibility and reason to change. `PlaceManageService` should not simultaneously own business flow, HTTP response construction, SQL, and vendor SDK parsing.

SRP does not mean one method per class.

### OCP

Introduce Strategy, Handler, Factory, or other extension points only when a real, stable direction of variation exists.

### LSP

An implementation must preserve the abstraction's contract for inputs, returns, Null behavior, exceptions, side effects, and state changes.

### ISP

Split interfaces around real consumers and implementers, not mechanically by method count.

### DIP

High-level business code should not couple directly to volatile vendors or protocols when a stable boundary provides real isolation value.

DIP does not require:

```text
all Service → ServiceImpl
all Mapper → Repository → RepositoryImpl
```

### Overengineering

Without a real need for replacement, extension, reuse, or isolation, do not mechanically create:

```text
Interface + Impl
Strategy
Factory
Repository wrapper
Adapter
Facade
Manager
```

Principle:

> SOLID should remove concrete complexity, not create ceremonial layers.

---

## 4. Controller / Web Responsibility

A Controller is an HTTP inbound adapter.

It may:

- bind HTTP parameters;
- perform structural input validation;
- obtain trusted request/caller context;
- invoke a Service;
- perform necessary protocol conversion.

It must not own:

```text
Controller → Mapper / DAO
SQL
business transactions
business state transitions
complex business composition
persistence DOs as a public API contract
```

If caller identity, tenant, department, or data-scope context comes from Web infrastructure, convert it at the inbound boundary into the project's stable caller-context representation rather than making Service / Manager depend back on Web objects.

For Spring MVC mechanisms use `spring.md`; for external HTTP contracts use `api-design.md`.

---

## 5. Service Responsibility

Service is the default application use-case boundary in this Skill Pack.

It may:

- implement business use cases and flows;
- perform business validation;
- coordinate business capabilities;
- call Manager / Mapper / Client according to actual complexity;
- organize application input and output.

It must not own HTTP protocol details, SQL, physical database mapping, or vendor-specific SDK protocol handling.

Whether a rule should remain in Service / Manager or move into a behavioral business object is decided by [business-rules.md](business-rules.md).

### 5.1 Service Class Naming

When a Service primarily provides ordinary resource management—CRUD, query, list, count, create/save, update, delete/remove—prefer the `*ManageService` suffix when the target project has no stronger existing convention.

Preferred:

```text
PlaceManageService
CaseManageService
EquipmentManageService
```

Use focused capability names for focused business use cases:

```text
PlaceAuditService
CaseRegistrationService
OrderDeliveryService
```

Do not rename unrelated stable Services merely to satisfy this convention.

### 5.2 Service Method Naming

For ordinary CRUD/query capabilities, prefer stable business-oriented prefixes when no project convention overrides them:

```text
get one       → get
get many      → list
count         → count
create / save → save
delete        → remove
modify        → update
```

Example:

```java
PlaceVO getPlace(String id);
List<PlaceVO> listPlaces(PlaceQuery query);
long countPlaces(PlaceQuery query);
void savePlace(PlaceSaveRequest request);
void removePlace(String id);
void updatePlace(PlaceUpdateRequest request);
```

Real business actions take precedence over CRUD templates:

```text
auditPlace
approveCase
rejectCase
registerCase
bindEquipment
```

Do not mirror persistence terminology such as `insert` / `delete` one-to-one at the Service boundary when business wording is clearer.

### 5.3 Splitting Services

Split a growing Service only when independently nameable and independently changing business capabilities emerge. File length or method count alone is insufficient.

Avoid vague extra layers such as `CommonService`, `HelperService`, or `ValidatorService` unless they truly represent a stable independent responsibility.

---

## 6. Manager Responsibility

Manager is an optional application-capability layer for real reuse, composition, or atomic operations, for example:

- meaningful composition of multiple Mappers / Clients;
- reusable data operations;
- atomic multi-table operations;
- cache + data-access composition;
- reusable complex data assembly.

Manager is not the vendor-protocol boundary; Client / Adapter owns that.

A simple flow may call Mapper or Client directly from Service. Do not create a pass-through Manager merely because another layer seems desirable.

Transaction placement follows the consistency scope defined by `transactions.md`, not the existence of a Manager class.

---

## 7. Mapper / DAO Responsibility

Mapper / DAO is the outbound database boundary for:

```text
SELECT
INSERT
UPDATE
DELETE
parameter/result mapping
SQL execution
```

It does not own business permission decisions, business state transitions, complete use cases, vendor calls, or business transaction orchestration.

Choose framework-specific rules only after identifying the actual persistence technology: `mybatis.md` for MyBatis/MyBatis-Plus, `rabbit-sql.md` for Rabbit-SQL, and `sql.md` for SQL semantics.

---

## 8. Client / Adapter Responsibility

Client / Adapter isolates external technical systems and may own:

- HTTP / RPC / SDK calls;
- vendor authentication and protocol parameters;
- vendor Request / Response conversion;
- external error-code normalization;
- technical timeout, connection, and serialization details required by the integration.

It should not orchestrate the application's business use case or absorb unrelated business decisions merely to reduce Service code.

Depending on project terminology, names may include `Client`, `Adapter`, `Gateway`, or `Integration`. Do not rename a stable convention mechanically.

---

## 9. Dependency Direction

Default logical dependencies:

```text
Inbound adapter → Service
Service → Manager / Mapper / Client
Manager → Mapper / Client
Mapper → Database
Client / Adapter → External System
```

Forbidden directions include:

```text
Controller → Mapper
Mapper → Service
Client → Service
lower layer → Controller
```

Simple flows may skip optional layers but must preserve responsibility boundaries.

### 9.1 Cross-Module Dependencies Must Remain Acyclic

Cross-module dependencies must form a one-way acyclic graph. Before introducing a new dependency from one business module to another, inspect the existing dependency path and confirm that the new edge does not create a cycle.

For example, avoid structures such as:

```text
case → place
place → organization
organization → case
```

Do not treat the following as architectural fixes for a module cycle:

```text
@Lazy
static helpers
ServiceLocator-style lookup
calling another module's Mapper / Client / private Manager
moving unrelated business logic into common
```

If a real cycle would be created, reconsider ownership first. When justified by a real stable contract, extract or invert the dependency at the appropriate boundary rather than hiding the cycle through framework mechanisms.

Principle:

> A dependency cycle means the participating modules no longer change independently; remove the cycle at the responsibility boundary rather than at the wiring layer.

---

## 10. Model Classification and Responsibility Packages

Classify models by responsibility instead of naming every data object DTO.

Default logical mapping:

```text
Request / Query → inbound request models, both under <module>.request by default
DTO             → application-internal data transfer, default <module>.dto
BO              → intermediate/composed business-processing semantics, default <module>.bo
DO              → database persistence model, default <module>.domain
VO              → concrete business interface/view output, default <module>.vo
```

`Request` and `Query` are semantically different class types but share the `request` Package by default:

```text
module.place.request.PlaceSaveRequest
module.place.request.PlaceAuditRequest
module.place.request.PlaceQuery
```

Do not create a separate `query` Package merely because the class suffix is `Query`.

Physical placement of the business module containing these Packages belongs to [project-structure.md](project-structure.md).

### Request

Operation-oriented external input. It must not contain trusted server-side identity or permission information that a client can spoof.

### Query

Query conditions and filtering semantics. Prefer a Query object when ordinary query parameters become numerous or cohesive.

### DTO

Use only for a real internal transfer contract that is not already adequately represented by Request, DO, VO, or another existing model.

### BO

Use for an intermediate or composed business-processing concept with independent meaning. Do not create a BO for every Service method.

### DO

Database persistence model. It must not be exposed directly as a public API contract. A DO is not automatically a Clean Architecture Entity; see `business-rules.md`.

### VO

Concrete business interface or view output. The unified outer HTTP response wrapper is governed by `api-design.md`, not classified as a VO.

### Conversion

Convert models only when responsibility, contract, or data semantics actually change. Avoid ceremonial chains such as:

```text
DO → DTO → BO → VO
```

A one-off simple mapping does not require a Converter / Assembler unless complexity, reuse, or semantic transformation justifies it.

---

## 11. Cross-Module Logical Calls

Within one application, cross-module collaboration should normally use the other module's stable Service / Facade capability rather than its Mapper, Client, or private Manager.

Preferred:

```text
CaseManageService
    ↓
PlaceManageService / PlaceFacade
```

Avoid:

```text
CaseManageService
    ↓
PlaceMapper
```

A Facade is optional and should exist only when a real module-facing contract benefits from it.

---

## 12. Responsibility Decision Flow

When adding or moving a class:

```text
What business / technical problem does it solve?
        ↓
Is it inbound, use-case orchestration, reusable application capability, database access, or external adaptation?
        ↓
Does an equivalent responsibility already exist?
        ↓
Are dependencies one-way?
        ↓
Would a new cross-module dependency create a cycle?
        ↓
If it is a model, is it Request / Query / DTO / BO / DO / VO?
        ↓
Which responsibility Package owns it?
        ↓
Use project-structure.md to determine the physical business-module location
```

Final principle:

> `layering.md` owns logical responsibilities, dependency direction, model classification, and responsibility Packages. `project-structure.md` owns physical module layout. `business-rules.md` owns core-rule versus use-case-rule placement. Keep those decisions distinct.
