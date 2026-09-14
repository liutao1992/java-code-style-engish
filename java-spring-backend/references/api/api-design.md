# API Design Standard

This document defines external HTTP API contracts, compatibility, and interface semantics.

This document answers:

> How should URL, HTTP Method, Request / VO, unified responses, pagination, error codes, compatibility, idempotency, and external security boundaries be designed?

For application layering, model responsibilities, and Package placement, read:

- [Application Layering and Model Boundaries](../architecture/layering.md)

For Spring MVC annotations, Validation, and Advice mechanisms, read:

- [Spring](../coding/spring.md)

For exception flow across layers, read:

- [Exception Handling and Error Boundaries](../architecture/error-handling.md)

Core principles:

> An API is a stable contract and should not change casually when database structures or internal implementation details change.

> Use VO for concrete business output. The default recommendation for a unified HTTP response wrapper is `ApiResponse<T>`. If the target project already has another unified wrapper, a historical API contract, or an established serialization contract, follow the existing project convention.

---

## 1. API Boundary

An API is responsible for:

* receiving client input;
* expressing business requests and responses;
* defining HTTP semantics;
* structural input validation;
* external error representation;
* maintaining interface compatibility.

An API should not expose:

* Mapper / DAO;
* database DOs;
* physical database column names;
* Java stack traces;
* internal exception types;
* third-party SDK objects;
* internal technical implementation details.

Controller architecture responsibilities are defined in `layering.md`, not duplicated here.

---

## 2. URL, Business Actions, and HTTP Methods

URLs should express resources and clear business semantics.

Common resources:

```text
/places
/cases
/equipments
```

Common resource operations:

```text
GET    /places
GET    /places/{id}
POST   /places
PUT    /places/{id}
PATCH  /places/{id}
DELETE /places/{id}
```

Business actions that do not map naturally to CRUD may use an explicit action path:

```text
POST /places/{id}/audit
POST /places/{id}/activate
POST /cases/{id}/submit
POST /cases/{id}/cancel
```

Avoid:

```text
/places/doSomething
/places/process
/places/handle
```

As a rule:

```text
GET    → query
POST   → create or business command
PUT    → complete or explicit update
PATCH  → partial update
DELETE → delete
```

Do not design every endpoint as POST merely because it is convenient to implement.

If the project already has a stable URL / Method style, preserve compatibility.

---

## 3. GET Requests

GET should normally be used for reads that:

* do not change business state;
* do not create persistent data;
* do not produce business write side effects.

Do not use GET for state-changing operations such as delete, audit, or submit.

---

## 4. Request Models

Use responsibility-specific Request models for complex interface input, for example:

```text
PlaceCreateRequest
PlaceUpdateRequest
PlaceAuditRequest
CaseRegisterRequest
```

Avoid overly broad names such as:

```text
PlaceRequest
CommonRequest
DataRequest
```

Do not rely heavily on:

```java
Map<String, Object>
```

to represent ordinary business requests merely to avoid creating a model.

For Request responsibilities and Package placement, read:

- [layering.md](../architecture/layering.md#91-request)

For Java implementation details, read:

- [java.md](../coding/java.md)

---

## 5. Parameter Location and Structural Validation

Path parameters identify resources:

```text
/places/{id}
```

Query parameters express filtering, pagination, and sorting:

```text
/places?status=ACTIVE&pageNum=1&pageSize=20
```

Body is used for complex business input.

Structural constraints are suitable for Bean Validation, for example:

```text
@NotNull
@NotBlank
@NotEmpty
@Size
@Min
@Max
@Pattern
```

Business validation—such as whether the current status permits an audit or whether the current caller may perform an action—belongs in the business layer. Do not replace business flow with complex Bean Validation.

For Spring-specific usage, read `spring.md`.

---

## 6. Query Objects

When query conditions become numerous, use a responsibility-specific Query object, for example:

```java
public class PlaceQuery {

    private String placeName;

    private PlaceStatus status;

    private String centerCode;
}
```

For Query responsibilities and Package placement, read:

- [layering.md](../architecture/layering.md#92-query)

Do not use `Map<String, Object>` instead of an ordinary business query model.

---

## 7. Isolate Database Naming from the API

The API uses English Java business semantics and does not expose physical database column names directly.

For example, an existing database may use:

```text
zjhm
rqsj
csbh
```

while the API exposes:

```text
identityNumber
entryTime
placeCode
```

If a historical API already uses other field names, preserve compatibility instead of renaming them without authorization merely to satisfy this standard.

Database mapping details are maintained by the persistence-framework reference.

---

## 8. VO and Unified Responses

Use VO for concrete business output, for example:

```text
PlaceVO
PlaceDetailVO
PlaceStatsVO
PlaceTreeNodeVO
```

For VO responsibilities and Package placement, read:

- [layering.md](../architecture/layering.md#96-vo)

Do not add another model layer merely to distinguish “business output” from “HTTP output” when the responsibility is the same. Convert only when output responsibility, contract, or data semantics actually change.

The default recommendation for a unified HTTP response is:

```text
ApiResponse<T>
```

Typical flow:

```text
PlaceVO
   ↓
ApiResponse<PlaceVO>
```

`ApiResponse<T>` is an outer HTTP transport wrapper. It is not a concrete business-output model and is not part of the Request / Query / DTO / BO / DO / VO responsibility classification.

`ApiResponse<T>` is this Skill Pack's default recommendation, not a mandate that overrides the target project's existing HTTP response contract.

If the target project already has:

* another unified response wrapper;
* a fixed JSON field structure;
* a global error response format;
* published API contracts;

reuse the project's existing HTTP contract and semantics. Do not create a parallel wrapper system or bulk-change historical APIs merely to apply this Skill Pack.

Do not expose database DOs directly as interface output.

> Use VO for concrete business output; unified HTTP wrappers such as `ApiResponse<T>` are responsible only for transport-layer response structure.

---

## 9. Returned Fields and Sensitive Information

Return only data the client actually needs. Do not copy every DO field into a VO simply because it exists.

Pay particular attention to:

* internal states and technical fields;
* deletion markers;
* data-scope fields;
* audit fields;
* password / Token / Secret;
* full identity documents;
* biometric information;
* internal notes and server information.

API fields are determined by the interface contract, not by the database schema.

---

## 10. Null, Boolean, Enums, Time, and IDs

The API must maintain stable Null semantics among:

```text
null
empty string
empty array
field absent
```

For collection results, `[]` is normally preferred, but an existing API contract takes precedence.

Boolean fields should use names with clear business meaning, for example:

```text
enabled
deleted
editable
auditable
```

Actual serialized names follow the existing contract.

Enum values must remain stable. Do not invent aliases, compatibility values, or statuses.

Time-field names and formats should express clear business meaning. For cross-time-zone APIs, make UTC / offset / time-zone conventions explicit.

Once a resource ID is published as String / Long or another type, do not casually change the API type merely because the internal database type changes.

---

## 11. Pagination and Sorting

Pagination parameters and pagination-result structures must follow the project's unified convention.

For example, if the project already uses:

```text
pageNum
pageSize
```

do not casually mix in:

```text
page
pageIndex
current
```

Pagination results must have stable ordering.

If the client may specify sort fields, map them through an allow-list. Never concatenate raw user input directly into SQL.

A pagination wrapper is a generic interface structure, not a concrete business VO.

---

## 12. Error Codes, Messages, and HTTP Status

Public API errors should form a stable contract, for example:

```text
error code
error message
necessary request-tracing identifier
```

Error codes should be stable, have clear meaning, and reuse the target project's existing system.

Do not invent an error-code scheme or change existing error-code meanings without a real requirement.

Error messages must not directly expose:

* SQL;
* database table names;
* Java stack traces;
* Java exception classes;
* internal server paths or addresses;
* Token / Secret;
* third-party internal exception details.

Do not unconditionally return `exception.getMessage()` to the client.

HTTP Status policy follows the existing project convention. Service / Manager is not responsible for choosing HTTP Status.

For exception flow across layers, read:

- [error-handling.md](../architecture/error-handling.md)

For Spring Advice implementation, read:

- [spring.md](../coding/spring.md)

---

## 13. API Compatibility

Without an explicit requirement, do not casually change a published:

* URL;
* HTTP Method;
* Request field;
* VO / output field;
* unified response wrapper;
* field type;
* Null semantic;
* enum value;
* time format;
* error code;
* pagination structure.

Naming consistency is not a justification for breaking compatibility.

Adding a field is usually lower risk than deleting, renaming, or changing a type, but still requires checking sensitive information, serialization, and client compatibility.

Do not mechanically create `/v2`, `/v3`, and so on for small changes. Consider API versioning only for major contract changes that cannot remain compatible.

---

## 14. Idempotency

For write endpoints such as:

```text
submit
audit
payment
state transition
external callback
write operations that may be retried
```

evaluate the behavior of repeated requests based on the real business contract.

Check whether repeated calls may cause:

* duplicate inserts;
* repeated deductions;
* duplicate sends;
* repeated state changes.

Depending on existing business conventions, use mechanisms such as unique constraints, state checks, idempotency keys, conditional UPDATE, or processed-record tracking.

Do not assume every POST is inherently “non-idempotent,” and do not invent an idempotency strategy without evidence.

---

## 15. Authentication, Authorization, and Client Input

Client-supplied values such as:

```text
userId
deptCode
tenantId
dataScope
```

must not be trusted by default.

Identity, tenant, and data-scope information should preferably come from the target project's trusted server-side context.

Do not allow client parameters to bypass authorization or expand data scope.

Concrete security implementation follows the target project's existing security standards and code.

---

## 16. Batch, File, and Internal APIs

Batch APIs should make existing business constraints explicit, such as:

* item-count limits;
* partial success vs. all-or-nothing failure;
* transaction scope;
* per-item errors;
* duplication and idempotency.

Do not invent a fixed batch limit without evidence.

File uploads must consider file size, type, count, file name, storage, authorization, and security checks. Do not trust client-supplied file names or MIME types directly.

Internal APIs are still contracts. “Internal use” is not a reason to expose DOs, database field names, or ignore authorization boundaries.

For architecture rules governing cross-module calls within the same application, read `layering.md`; this document does not duplicate Service / Facade / Mapper call rules.

---

## 17. Controller Examples

Command-style endpoint:

```java
@PostMapping("/{id}/audit")
public ApiResponse<Void> audit(
        @PathVariable
        @NotBlank
        String id,
        @Valid @RequestBody PlaceAuditRequest request) {

    placeService.audit(id, request, currentOperator());
    return ApiResponse.success();
}
```

Query-style endpoint:

```java
@GetMapping("/{id}")
public ApiResponse<PlaceVO> detail(
        @PathVariable
        @NotBlank
        String id) {

    PlaceVO place = placeService.getById(id);
    return ApiResponse.success(place);
}
```

`ApiResponse.success(...)` is only an example of the default construction style. Actual method names, field structure, and serialization contracts follow the target project's implementation.

If the project already has another unified HTTP wrapper or a historical API contract, continue to use the existing design.

Controller layering responsibilities are defined in `layering.md`; Spring annotation and Validation usage is defined in `spring.md`.

---

## 18. Codex API Change Workflow

Before adding or modifying an API, check:

1. Whether a similar endpoint or unified response structure already exists.
2. Whether URL and HTTP Method match the existing contract.
3. Whether existing Request / Query / VO models can be reused.
4. When a new model is needed, whether responsibility and Package were determined according to `layering.md`.
5. Whether concrete business output uses VO and the unified response follows project conventions.
6. Whether field, Null, enum, time, pagination, or error-code semantics are being changed without authorization.
7. Whether structural validation and business validation are located at the correct boundaries.
8. Whether trusted identity / authorization context is used.
9. Whether a write endpoint has a real idempotency concern.
10. Whether database details, internal technical objects, or sensitive information are leaked.
11. Whether behavior changes require tests.

After the change, check:

* whether the API remains compatible;
* whether DOs / physical database names leak out;
* whether concrete business output uses the project's VO convention;
* whether `ApiResponse<T>` remained only a default recommendation rather than overriding an existing unified HTTP wrapper;
* whether dynamic sorting is safe;
* whether error codes and messages are stable and safe;
* whether client-provided identity, tenant, or data-scope parameters are being trusted incorrectly;
* whether any status, error code, batch threshold, or compatibility behavior was invented without evidence.

Final principle:

> The API reference maintains only the external contract. Concrete business output uses VO; unified HTTP wrappers and business-output models have separate responsibilities. Layering, Spring implementation, and exception flow are maintained by their own dedicated references.
