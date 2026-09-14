# Application Layering Standard

This document defines logical application layers, responsibility boundaries, dependency direction, model classification, and **responsibility Packages**.

This document answers:

> What responsibility does a class have in the application, what may it depend on, and which responsibility Package does it belong to?

For the **physical location and directory organization** of a project / module, read:

- [Project and Business Module Structure](project-structure.md)

Java implementation, Spring annotations, HTTP contracts, exceptions, SQL, transactions, and concurrency are maintained by their dedicated references and are not duplicated here.

Core principle:

> Determine responsibility first, then determine the Package. Isolate volatile protocols and technical details, and do not mechanically add call layers.

---

# 1. Default Logical Layers

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
                    Outbound Adapters
```

Responsibilities can be summarized as:

```text
Inbound adapter
→ how the outside world enters the application

Service
→ what the current business use case needs to do

Manager (optional)
→ reusable application capabilities, composite operations, atomic operations

Mapper / DAO
→ how the database is accessed

Client / Adapter
→ how external technical systems are accessed and adapted
```

Manager is optional. Simple business flows may use:

```text
Controller → Service → Mapper
```

or:

```text
Controller → Service → Client
```

Do not mechanically create a Manager merely to make the layering look “complete.”

Principle:

> Upper layers may depend on lower layers or stable abstractions; lower layers must not depend back on upper layers. The number of layers is determined by real responsibilities.

---

## 1.1 Inbound Adapters

Common inbound forms include:

```text
HTTP Controller
RPC Endpoint
Open API Endpoint
Message Consumer
Scheduled Task
Command / Job Handler
```

Shared responsibilities:

* receive external input or triggers;
* parse the protocol;
* perform structural validation required by the current protocol entry point;
* obtain trusted caller context;
* convert the request into an application call;
* invoke a Service / Facade;
* convert application output into the response or acknowledgement required by the protocol.

Adapters for different protocols should not call one another merely to reuse business logic.

Preferred:

```text
HTTP Controller ───┐
RPC Endpoint ──────┼→ PlaceManageService
Consumer ──────────┘
```

Avoid:

```text
RPC Endpoint → HTTP Controller → Service
Consumer → Controller
```

---

## 1.2 Outbound Adapters

Common outbound forms include:

```text
Mapper / DAO
HTTP Client
RPC Client
SDK Adapter
Object Storage Client
Message Producer
External Data Client
```

Database access and external technical calls are both outbound boundaries, but they have different responsibilities:

```text
Mapper / DAO
→ Database

Client / Adapter
→ External System / Vendor Protocol
```

Do not force HTTP, RPC, SDK, object storage, or message sending into Mapper / DAO merely to “unify the lower layer.”

---

## 1.3 The Business Core Should Not Know Protocol Details

Service / Manager should normally not depend directly on:

```text
HttpServletRequest / HttpServletResponse / ResponseEntity
RPC framework Request / Context
messaging middleware Record / Message
third-party SDK Request / Response / Exception
database physical column names
```

Protocol-specific and vendor-specific types should be converted at the appropriate inbound / outbound boundary into stable semantics understood by the application.

---

# 2. SOLID and Simple Design

Use SOLID to identify real problems in responsibility, substitutability, extensibility, interface design, and dependency direction—not to mechanically add design patterns.

## 2.1 SRP

A class should center on one primary responsibility and one main reason to change.

For example, `PlaceManageService` should not simultaneously own:

```text
business flow
+ HTTP response construction
+ SQL
+ third-party SDK protocol parsing
```

But SRP does not mean “one method per class,” nor does slightly more code automatically justify extracting a Manager.

## 2.2 OCP

Introduce extension points such as Strategy, Handler, or Factory only when a real, stable, recurring direction of variation exists.

Do not abstract in advance for speculative future changes.

## 2.3 LSP

An implementation must not violate the contract of its abstraction for inputs, returns, Null behavior, exceptions, side effects, or state changes.

## 2.4 ISP

Split interfaces around real consumer / implementer boundaries, not mechanically by method count.

## 2.5 DIP

High-level business code should not couple directly to volatile vendors or technical protocols. When there is a real need for replacement, isolation, or testing, isolate them behind stable boundaries such as Client / Adapter / SPI.

DIP does **not** mean:

```text
all Service → ServiceImpl
all Mapper → Repository → RepositoryImpl
```

## 2.6 Overengineering

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

> SOLID should reduce real complexity, not manufacture new complexity.

---

# 3. Controller / Web Layer

A Controller is an HTTP inbound adapter.

Primary responsibilities:

* bind HTTP parameters;
* perform structural input validation;
* obtain request-related caller context;
* invoke a Service;
* perform necessary protocol conversion.

Forbidden responsibilities:

* Controller → Mapper / DAO;
* writing SQL;
* defining business transactions;
* owning business state transitions;
* complex cross-data-source business composition in the Controller;
* using database DOs directly to execute a business flow.

If current user, department, tenant, or data-scope information comes from the Web Request, `SecurityContext`, or a request ThreadLocal, obtain it at the inbound boundary and explicitly pass a responsibility-specific object such as:

```text
Operator
CallerContext
the project's existing unified caller context
```

Do not make Service / Manager depend back on Web objects merely to obtain the “current request user.”

If the project already has a unified context mechanism that safely covers multiple entry points, reuse it rather than creating a second Context model.

For Spring MVC usage read `spring.md`; for URL, Method, Request, VO, and unified response contracts read `api-design.md`.

Principle:

> Controller should do only what the protocol boundary must do; business decisions belong in Service.

---

# 4. Service Layer

Service owns business use cases and business flows.

Primary responsibilities:

* implement business use cases;
* perform business validation;
* coordinate multiple business capabilities;
* orchestrate Managers or stable outbound capabilities;
* in simple cases, call Mapper / Client directly;
* organize business input and output.

Service does not own:

* HTTP Status / HTTP response protocol;
* SQL;
* database column mapping;
* third-party SDK protocol details.

Method names should preferably express clear business behavior:

```text
audit
register
approve
reject
bindEquipment
```

Avoid long-term use of meaningless names such as:

```text
handle
process
doSomething
```

## 4.1 Service Class Naming

When a Service primarily provides ordinary resource management for a business module—especially CRUD, query, list, count, create/save, update, and delete/remove capabilities—use the `*ManageService` suffix.

Preferred:

```text
PlaceManageService
CaseManageService
EquipmentManageService
```

For example, if one Place Service exposes capabilities such as:

```text
getPlace
listPlaces
countPlaces
savePlace
updatePlace
removePlace
```

then the class should be named:

```text
PlaceManageService
```

rather than the overly broad:

```text
PlaceService
```

The `Manage` qualifier makes the class responsibility explicit: it is the module-level management surface for ordinary create, read, update, delete, and query operations.

Do not apply `Manage` mechanically to every Service. A Service centered on a specific business use case or domain action should use the name that best expresses that responsibility, for example:

```text
PlaceAuditService
CaseRegistrationService
OrderDeliveryService
```

If the target project already has a stable historical naming contract, do not rename unrelated existing Services merely to satisfy this convention. Apply the convention to new Services and to Services being intentionally renamed or significantly reshaped by the current task.

Principle:

> CRUD / query / resource-management responsibility → `*ManageService`; focused business-use-case responsibility → a specific action- or capability-oriented Service name.

## 4.2 Service Method Naming

For ordinary CRUD / query-oriented business capabilities, when the target project has no more specific stable convention, prefer:

```text
get one object      → get
get multiple objects→ list
get a count         → count
create / save       → save
delete              → remove
modify              → update
```

For example, in `PlaceManageService`:

```java
PlaceVO getPlace(String id);

List<PlaceVO> listPlaces(PlaceQuery query);

long countPlaces(PlaceQuery query);

void savePlace(PlaceSaveRequest request);

void removePlace(String id);

void updatePlace(PlaceUpdateRequest request);
```

Collection-returning methods use the `list` prefix. When the method directly denotes a resource collection, prefer a plural noun:

```text
listPlaces
listCases
listEquipmentItems
```

If the focus is the filtering semantics, these forms are also valid:

```text
listByStatus
listByQuery
listAvailablePlaces
```

Do not sacrifice clearer business meaning just to satisfy a “plural suffix” preference.

`save` expresses a create / save business action at the Service layer. If creation and modification have different business meaning, use separate responsibility-specific methods instead of blurring all writes into `save`.

A real business action takes precedence over a CRUD template. For example:

```text
auditPlace
approveCase
rejectCase
registerCase
bindEquipment
```

When these names already express the use case accurately, do not mechanically rename them to:

```text
updatePlace
saveCase
```

Concrete data-access naming for Mapper / DAO is maintained by the persistence-framework references: read `mybatis.md` for MyBatis / MyBatis-Plus and `rabbit-sql.md` for Rabbit-SQL. Service should not use persistence terms such as `insert` / `delete` merely to mirror database operations one-to-one.

Principle:

> CRUD-oriented ManageServices use stable method prefixes to reduce cognitive cost; when a clear business action exists, express that business meaning instead of letting a naming template hide the real use case.

## 4.3 Splitting Services

A growing Service is only a signal. Split by business capability only when independently nameable and independently changing use cases emerge, for example:

```text
OrderQueryService
OrderCreateService
OrderDeliveryService
```

Do not create these merely because of line or method count:

```text
OrderHelperService
OrderCommonService
OrderValidatorService
```

unless they truly have independent, stable responsibilities.

Principle:

> Split Services by use case and reason to change, not mechanically by file length.

---

# 5. Manager Layer

Manager is an **optional application-capability layer**.

Appropriate uses include:

* meaningful composition of multiple Mappers / Clients;
* reusable data operations;
* atomic multi-table operations;
* application-level composition of cache + data access;
* application-level composition of multiple external capabilities;
* complex data assembly required by multiple Service use cases.

The focus of a Manager is application-level reuse, composition, and atomic capabilities—not adaptation of a third-party protocol.

For example:

```text
PlaceManageService
    ↓
FaceRecognitionManager
    ↓
FaceRecognitionClient
    ↓
Vendor HTTP / SDK
```

A simple case may use:

```text
PlaceManageService → FaceRecognitionClient
```

Do not create a pass-through Manager merely because “Service must not call Client.”

Whether a transaction belongs in Manager is determined by the consistency scope; read `transactions.md`. Do not create a Manager merely because you want somewhere to place a transaction annotation.

Principle:

> Manager exists because a real application capability exists—not because a layer or annotation needs a home.

---

# 6. Mapper / DAO Layer

Mapper / DAO is the outbound boundary for database access. It is responsible for:

* SELECT;
* INSERT;
* UPDATE;
* DELETE;
* parameter and result mapping;
* SQL execution.

It is not responsible for:

* business permission decisions;
* business state transitions;
* complete business flows;
* third-party service calls;
* business transaction orchestration.

Choose framework-specific rules based on the persistence technology actually in use: read `mybatis.md` for MyBatis / MyBatis-Plus and `rabbit-sql.md` for Rabbit-SQL. SQL itself is governed by `sql.md`. Do not assume a framework merely because an interface is named Mapper / DAO.

---

# 7. Client / Adapter Layer

Client / Adapter is the outbound boundary for external technical systems. It is a peer of Mapper / DAO, not a sublayer of Mapper.

Responsibilities:

* HTTP / RPC / SDK calls;
* vendor authentication and protocol parameters;
* vendor Request / Response conversion;
* isolation of external error codes and exceptions;
* normalization of external nullable values or other protocol-specific differences;
* technical details such as timeout, connection, and serialization required by the integration.

It is not responsible for:

* orchestrating the current application's business use case;
* constructing HTTP Controller responses;
* coordinating multiple business state transitions;
* absorbing unrelated business decisions merely to reduce Service code.

Depending on the project's existing terminology, names may include:

```text
Client
Adapter
Gateway
Integration
```

Do not mechanically migrate one naming convention to another.

Principle:

> Client / Adapter isolates volatile technical protocols; Manager composes application capabilities; Service expresses business use cases.

---

# 8. Call and Dependency Rules

Default recommendation:

```text
Inbound adapter → Service
Service → Manager / Mapper / Client
Manager → Mapper / Client
Mapper → Database
Client / Adapter → External System
```

Forbidden:

```text
Controller → Mapper
Mapper → Service
Client → Service
lower layer → Controller
```

Simple flows may skip optional layers, but must not tunnel through technical details that should remain hidden.

For example:

```text
Service → Mapper
```

is allowed;

```text
Controller → Mapper
```

is not.

---

# 9. Model Classification and Package Placement

Classify models by responsibility instead of naming all data objects DTOs.

The default model system first groups by Package / boundary, then uses class names to express finer semantics:

```text
Request / Query → inbound request models, both under <module>.request by default
  Request       → operation input from an external interface
  Query         → query conditions and filtering semantics

DTO             → application-internal data transfer, default <module>.dto
BO              → intermediate / composed business-processing semantics, default <module>.bo
DO              → database persistence model, default <module>.domain
VO              → concrete business interface / view output, default <module>.vo
```

`Request` and `Query` are two semantic names within the same inbound-request model group; they do not imply separate Packages. The default mapping is:

```text
Request ─┐
         ├→ <module>.request
Query   ─┘

DTO     → <module>.dto
BO      → <module>.bo
DO      → <module>.domain
VO      → <module>.vo
```

For example:

```text
module.place.request.PlaceSaveRequest
module.place.request.PlaceAuditRequest
module.place.request.PlaceQuery
module.place.dto.PlaceDTO
module.place.bo.PlaceAuditBO
module.place.domain.PlaceDO
module.place.vo.PlaceDetailVO
```

Do not mechanically create:

```text
module.place.query
```

merely because the model type is named `Query`.

Model responsibility and the project's physical directory are still separate concerns:

```text
business module location
→ project-structure.md

model semantics and responsibility Package
→ this document
```

If the target project already has a clear and stable Package structure, follow it rather than bulk-migrating historical code to this default.

---

## 9.1 Request

Request represents operation-oriented interface input submitted by an external caller.

Examples:

```text
PlaceSaveRequest
PlaceAuditRequest
```

Default Package:

```text
<module>.request
```

A Request should not carry server-side identity data that a client cannot provide trustworthily, such as the current Operator or tenant permission context.

---

## 9.2 Query

Query represents query conditions and filtering semantics.

Examples:

```text
PlaceQuery
CaseQuery
```

Query remains semantically distinct from an ordinary Request, but by default lives with Request under:

```text
<module>.request
```

Use the `*Query` class name to express its query responsibility; do not create a separate `query` Package.

When ordinary query conditions grow, prefer a Query object rather than an unbounded method parameter list or `Map<String, Object>`.

---

## 9.3 DTO

DTO is used for an explicit application-internal data-transfer boundary.

Appropriate when:

* a stable group of data is passed across layers;
* an internal capability's input / output is not equivalent to Request, DO, or VO;
* an operation forms a clear internal data contract.

Do not use DTO merely because “we do not know what else to call it.”

---

## 9.4 BO

BO represents an intermediate result, calculation result, or composite object with independent meaning during business processing.

Create one only when a real intermediate business semantic exists; do not create a BO for every Service method.

---

## 9.5 DO

DO represents the database persistence structure.

DO fields use English Java business semantics. Map physical database names explicitly through the persistence layer, such as SQL column aliases or MyBatis ResultMap, so physical database naming does not leak into business models.

A DO is not exposed directly through an external API.

---

## 9.6 VO

VO represents concrete business-interface or view output, for example:

```text
PlaceVO
PlaceDetailVO
PlaceStatsVO
```

The unified outer HTTP response wrapper is not a VO; its rules are maintained by `api-design.md`.

Do not mechanically create another model layer merely to distinguish “business output” from “HTTP output” when the responsibilities are identical.

---

## 9.7 Model Conversion

Convert models only when responsibility, contract, or data semantics actually change.

Avoid valueless chains such as:

```text
DO → DTO → BO → VO
```

If a model layer has no independent responsibility, skip it.

A simple one-off DO → VO field mapping does not require a Converter / Assembler. Extract a dedicated mapping capability only when mapping is complex, reused in several places, or contains real business transformation rules.

---

# 10. Cross-Module Calls

Within the same application, cross-module calls should preferably go through the other module's stable Service / Facade capability.

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

A module boundary should not be penetrated to reach the other module's Mapper, Client, or internal Manager merely because everything runs in the same JVM.

Whether a Facade is necessary depends on a real module boundary and public calling surface; do not mechanically create one.

---

# 11. Responsibility Decision Flow

When adding or moving a class, decide in this order:

```text
What business / technical problem does it solve?
        ↓
Is it inbound, a business use case, an application capability, database access, or external technical adaptation?
        ↓
Does an implementation with the same responsibility already exist?
        ↓
Are dependencies one-way?
        ↓
If it is a model, first decide whether it belongs to the Request / Query inbound-request group, or to DTO / BO / DO / VO
        ↓
Request / Query go to request by default; other models map to Packages by their own responsibility
        ↓
Then use project-structure.md to determine the physical business-module location
```

Final principle:

> `project-structure.md` determines “which business module and physical directory this belongs to”; `layering.md` determines “what this class logically is and what it may depend on.” Request and Query share the `request` Package by default and are distinguished by class name. Responsibility comes before Package, and Package comes before file creation.
