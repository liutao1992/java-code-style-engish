# Java Coding Standard

This document defines implementation standards at the Java-language level.

This document answers:

> How should Java classes, methods, parameters, collections, Null handling, exceptions, logging, comments, and formatting be written?

For model responsibilities and Package placement, read:

- [Application Layering and Model Boundaries](../architecture/layering.md)

Spring, API, transactions, concurrency, MyBatis, and other specialized behavior are maintained by their own references and are not duplicated here.

Core principle:

> Prefer clear semantics, correct behavior, and consistency with the target project. Rules exist to reduce cognitive load and common errors, not to mechanically rewrite reasonable existing code.

---

## 1. Naming

### 1.1 Class Names

Use `UpperCamelCase` and express the real responsibility:

```java
PlaceService
PlaceQuery
PlaceDetailVO
PlaceNotFoundException
```

Avoid vague names such as:

```text
CommonService
DataHandler
BizUtil
ProcessHelper
```

Create abstract classes, interfaces, and implementation classes only when the responsibility genuinely exists. Do not mechanically generate `Interface + Impl` from a naming template.

### 1.2 Methods, Fields, Parameters, and Local Variables

Use `lowerCamelCase` and prefer complete English business semantics:

```java
identityNumber
expirationTime
queryPlaceDetail()
```

Avoid arbitrary abbreviations without industry consensus, pinyin, Chinese identifiers, and pinyin-English mixtures.

When collection or type semantics need to be expressed, names such as these are reasonable:

```text
nameList
placeMap
workQueue
```

Do not mechanically append type names such as `String` or `Integer` when they add no value.

### 1.3 Constants and Magic Values

Constants use `UPPER_SNAKE_CASE`:

```java
private static final int MAX_RETRY_COUNT = 3;
```

**Required:** business or technical magic values must not appear directly in code without a named meaning.

A magic value is a fixed literal that carries real semantics—such as status, type, threshold, timeout, retry count, cache duration, or business code—but whose meaning can only be inferred from surrounding context.

Avoid:

```java
if ("1".equals(status)) {
    ...
}

if (retryCount >= 3) {
    ...
}

cache.put(key, value, 300);
```

Prefer named constants or Enums:

```java
if (PlaceStatus.ENABLED.getCode().equals(status)) {
    ...
}

if (retryCount >= MAX_RETRY_COUNT) {
    ...
}

cache.put(key, value, CacheConsts.PLACE_DETAIL_TTL_SECONDS);
```

Values that normally need names include:

```text
business status / type codes
permission / source codes
cache TTL
timeouts
retry counts
batch thresholds
fixed business ratios
numbers or strings with business meaning
```

“Do not use magic values” does not mean extracting every Java literal into a constant. Literals without independent business / technical meaning can remain inline, for example:

```java
for (int i = 0; i < items.size(); i++) {
    ...
}

if (name.isEmpty()) {
    ...
}
```

A useful test is:

> If this value changes, must a maintainer first understand what it represents in order to change it safely?

If yes, it should not remain as an anonymous literal scattered through the codebase.

### 1.4 Packages

Use lowercase English Package names. Common responsibility terms include:

```text
controller
service
manager
mapper
client
adapter
request
dto
bo
domain
vo
```

`Query` is a model semantic and does not require a separate `query` Package. By default it lives with Request under `request`. Responsibility and Package placement are defined in `layering.md`; physical project structure is defined in `project-structure.md`.

### 1.5 Expose Real Design-Pattern Roles in Naming

When a module / Package, interface, class, or method **genuinely has a specific design-pattern role**, naming should make that role visible so readers can understand the architectural intent without first opening the implementation.

Typical type names:

```text
PaymentStrategy
DefaultPaymentStrategy
NotificationFactory
StorageAdapter
PlaceBuilder
AuditHandler
AuditHandlerChain
ExportCommand
ExportCommandHandler
```

A technical submodule or responsibility Package organized around a real pattern may use names such as:

```text
strategy
factory
adapter
handler
command
```

But business modules should still primarily express business capabilities such as `place` or `casecenter`; do not rename an entire business module to `placeStrategy` merely because one Strategy appears inside it.

Method names should express the typical action of the real role rather than appending meaningless pattern suffixes:

```text
Builder      → build(...)
Factory      → create(...) / createXxx(...)
Command      → execute(...)
Handler      → handle(...)
Visitor      → visit(...) / accept(...)
```

Therefore, ordinary business methods should avoid context-free `handle()` or `execute()`. But if the containing type is clearly a pattern role such as:

```text
PlaceAuditHandler
ExportCommand
```

then:

```java
handler.handle(context);
command.execute();
```

is meaningful and appropriate.

Do not invent a design pattern merely to use its vocabulary. Names such as the following should be used only when the corresponding responsibility exists:

```text
Strategy
Factory
Adapter
Builder
Handler
Command
Observer
Visitor
Template
Facade
```

For example, a single ordinary branch does not justify `XxxStrategy` when there is no replaceable algorithm family; an object-returning method does not automatically justify `XxxFactory`.

For whether a design pattern is really needed or becomes overengineering, read `layering.md`.

Principle:

> First confirm the pattern and role are real, then let naming expose the architectural intent. Names explain design; they must not fabricate design.

---

## 2. Class Design and OOP

Design each class around a clear responsibility.

Before creating a new class, search for existing implementations and avoid:

* endlessly growing `Utils` classes;
* one class mixing unrelated responsibilities;
* premature abstractions for speculative future extension;
* interfaces, Factory, Strategy, or Repository wrappers created only for form.

Principle:

> Determine responsibility first, then class name and Package, and only then decide whether a new class is actually needed.

### 2.1 Use `@Override` Explicitly

When overriding a superclass or interface method, use:

```java
@Override
```

### 2.2 Access Static Members Through the Type Name

Preferred:

```java
Objects.equals(a, b);
PlaceConstants.MAX_NAME_LENGTH;
```

### 2.3 Keep Visibility as Narrow as Practical

Use the smallest visibility required by real callers. Do not make all internal implementation details `public` merely for convenience.

### 2.4 Do Not Mechanically Create `Interface + Impl`

Introduce an interface only when there is a real need such as:

* multiple implementations;
* SPI;
* replaceable capability;
* third-party isolation;
* a stable module boundary.

A normal single-implementation Service does not automatically require:

```text
PlaceService
PlaceServiceImpl
```

merely because the codebase is layered.

### 2.5 Keep Constructors and Accessors Simple

Constructors should not perform database, HTTP/RPC, file I/O, long-running business flows, or hidden state transitions.

Ordinary getters / setters should not hide business rules or side effects. When a constraint is business behavior, use an explicit business method such as:

```java
changeStatus(...)
approve(...)
activate(...)
```

---

## 3. Java Implementation of Model Objects

Responsibilities for Request, Query, DTO, BO, DO, and VO are uniquely maintained by `layering.md`; this section only defines Java implementation choices.

When the project has no stronger convention, ordinary models default to normal `class` types. Do not proactively convert models to `record` merely for style.

For ordinary accessors without business logic, prefer:

```java
@Getter
@Setter
```

over mechanically using `@Data`. `@Data` also affects:

```text
equals
hashCode
toString
constructors
```

Use it only after confirming those behaviors match the object's semantics.

### 3.1 JavaBean Naming

A JavaBean name should directly express its model responsibility, without adding a generic suffix that has no independent meaning.

Preferred:

```text
PlaceCreateRequest
PlaceQuery
PlaceDTO
PlaceAuditBO
PlaceDO
PlaceDetailVO
```

`PlaceQuery` still expresses “query conditions” through the class name, but its default Package is `<module>.request`; do not create a separate `query` Package merely because the class name ends in `Query`.

Avoid generic names when responsibility is already clear:

```text
PlaceBean
PlaceInfo
PlaceData
PlaceModel
CommonBean
```

If the target project has an established historical naming convention, preserve compatibility instead of bulk-migrating it for this rule.

JavaBean properties continue to use `lowerCamelCase` and full English business semantics, for example:

```java
private String placeCode;
private LocalDateTime entryTime;
private Boolean enabled;
```

Boolean properties should preferably describe state directly:

```text
enabled
deleted
editable
auditable
```

Do not mechanically name fields `isEnabled` or `isDeleted` merely for JavaBean style. Existing Jackson, MyBatis, RPC, or historical API serialization contracts take precedence where compatibility requires them.

Principle:

> A JavaBean name expresses “what responsibility this object has”; field names express “what business meaning it carries.” Do not use `Bean / Info / Data / Model` as substitutes for responsibility design.

### 3.2 `@Builder` and `@NoArgsConstructor`

For ordinary models that are assembled frequently in Java, have many fields, or become noticeably less readable with repeated setters, Lombok `@Builder` may be preferred.

Instead of:

```java
PlaceDetailVO detail = new PlaceDetailVO();
detail.setId(place.getId());
detail.setPlaceCode(place.getPlaceCode());
detail.setPlaceName(place.getPlaceName());
detail.setEnabled(place.getEnabled());
```

consider:

```java
PlaceDetailVO detail = PlaceDetailVO.builder()
        .id(place.getId())
        .placeCode(place.getPlaceCode())
        .placeName(place.getPlaceName())
        .enabled(place.getEnabled())
        .build();
```

For ordinary mutable models that need standard JavaBean no-argument instantiation for binding, mapping, or deserialization, `@NoArgsConstructor` may be used as well.

When class-level `@Builder` and `@NoArgsConstructor` are combined, ensure there is a usable all-arguments construction path for the builder. Recommended pattern:

```java
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor(access = AccessLevel.PRIVATE)
public class PlaceDetailVO {

    private String id;
    private String placeCode;
    private String placeName;
    private Boolean enabled;
}
```

Here:

```text
@NoArgsConstructor
→ preserves no-arg JavaBean / framework instantiation

@Builder
→ improves object assembly and reduces long setter chains

@AllArgsConstructor(access = AccessLevel.PRIVATE)
→ gives the class-level Builder a complete construction path without unnecessarily exposing a public all-args constructor
```

Do not add `@Builder` to every object merely to standardize Lombok style. First evaluate whether it is appropriate when:

* the object has only one or two fields;
* arbitrary field combinations should not be allowed;
* invariants must be established through an explicit constructor or factory;
* Builder could bypass necessary business validation or state constraints;
* the target project already has another stable construction approach.

Likewise, `@NoArgsConstructor` is not a default requirement for every class. Immutable objects or objects whose invariants must be established during construction should not receive an unjustified no-arg constructor merely to look like a JavaBean.

For Request / Query / DTO / BO / DO / VO POJOs, do not use `@Builder.Default` to establish field defaults; see section 3.5.

Builder exists for object construction, not to replace business behavior. A state transition such as:

```java
place.approve(operator);
```

must not be replaced merely to use Builder with unrestricted construction such as:

```java
PlaceDO.builder()
        .status(APPROVED)
        .build();
```

when doing so bypasses the existing state-transition rules.

Principle:

> `@Builder` improves object assembly; `@NoArgsConstructor` satisfies a real JavaBean / framework instantiation need. Neither may break invariants, business rules, or existing framework contracts.

### 3.3 Do Not Add Setters Mechanically

If an object should remain immutable or support only controlled mutation, expose only the access needed by its responsibility.

Do not replace business behavior with ordinary setters.

### 3.4 Primitives and Wrapper Types

Use wrapper types when the model must express:

```text
not provided
unknown
database NULL
```

Use primitives for local computations when no Null semantic exists.

Do not bulk-change existing API / database Null semantics merely for stylistic uniformity.

### 3.5 POJOs Do Not Define Field Defaults

**Required:** Request, Query, DTO, BO, DO, VO, and similar POJOs should not define property defaults in field declarations or Builders by default.

Do not introduce:

```java
private Boolean enabled = true;
private Integer sortOrder = 0;
private String status = "PENDING";
private LocalDateTime createTime = LocalDateTime.now();
```

or:

```java
@Builder.Default
private Boolean enabled = true;
```

to hide business defaults inside model construction.

A POJO carries data; its definition should not silently decide:

```text
business status
default switches
default codes
current time
default sort value
```

If a real default exists, establish it explicitly at the boundary that owns the meaning, for example:

```text
Controller / inbound conversion
→ request-default semantics explicitly defined by the protocol

Service / Manager
→ default status or value defined by a business rule

Database
→ a DEFAULT owned by the database and explicitly agreed upon
```

Do not repeat the same default across multiple boundaries.

If a historical model's field initialization is already part of a stable serialization, persistence, or business contract, do not remove it in bulk merely for this rule. Apply the rule to new models or when the current task explicitly changes that model's semantics.

Principle:

> A POJO carries values; it does not hide business default decisions. Defaults are created explicitly by the boundary that truly owns their meaning.

### 3.6 `toString` and Sensitive Information

Do not mechanically dump full objects for debugging. Passwords, Token, Secret, identity documents, biometric data, and other sensitive fields must not leak through generated `toString()` or logs.

---

## 4. Constants and Enums

### 4.1 Group Constants by Responsibility and Function

Do not maintain every project constant in one giant all-purpose class.

Avoid indefinitely growing classes such as:

```text
Constants
CommonConstants
GlobalConstants
SystemConstants
```

If a constant belongs only to one class's implementation, prefer a `private static final` field in that class.

For constants reused across classes, group them into responsibility-specific constant classes with real semantic ownership, for example:

```text
CacheConsts
SystemConfigConsts
FileUploadConsts
PlaceConsts
```

Typical mapping:

```text
cache-related constants
→ CacheConsts

system-configuration constants
→ SystemConfigConsts

technical file-upload limits
→ FileUploadConsts

stable Place-module business constants
→ PlaceConsts
```

Do not move a constant into an unbounded shared dumping ground merely because “many places can use it.” Business constants should remain with the business module that owns their meaning; only truly cross-module and stable technical constants belong in a shared technical boundary.

Conversely, do not create one class per constant merely to “categorize” them. Choose a grouping granularity that can be explained by one clear responsibility name.

Principle:

> Constants follow their semantic owner. Prefer a small number of clear responsibility-based groups over one global constant warehouse.

### 4.2 Prefer Enum for a Fixed Finite Value Domain

When a variable's valid values form a clear, fixed, finite set, prefer Enum over scattered string or numeric constants.

Example:

```java
public enum PlaceStatus {
    DRAFT,
    ENABLED,
    DISABLED
}
```

Typical use cases include:

```text
business status
audit result
fixed business type
fixed source type
finite operation type
```

If the database or an external API already uses stable codes, let the Enum carry and convert those codes explicitly rather than scattering values such as:

```text
"0"
"1"
"PENDING"
"FORMAL"
```

through business code.

Enum is not appropriate when the domain is truly open-ended, dynamically extended by configuration, or controlled by an external system whose complete value set the application does not own. Model those cases according to the real contract rather than pretending the domain is fixed.

Do not invent business statuses, codes, or compatibility mappings that the target project does not define.

Principle:

> When the value domain is genuinely fixed, express the range in the type system. When it is dynamically controlled by configuration or an external contract, respect that real contract.

### 4.3 Numeric Literals

Use uppercase `L` for `long` literals:

```java
1000L
```

For numbers with business / technical meaning, also follow the magic-value rule in 1.3.

---

## 5. Method Design and Readability

A method should perform one clear operation. Prefer explicit business names, Guard Clauses, and shallow nesting.

Recommended:

```text
auditPlace
registerCase
bindEquipment
```

Avoid:

```text
handle
process
doSomething
```

When the containing type genuinely has a Handler, Command, Visitor, or similar design-pattern role, conventional actions such as `handle`, `execute`, or `visit` may be accurate; see section 1.5.

### 5.1 Control Parameter Count

For newly created or significantly modified ordinary business methods, default to no more than **5 parameters**. This number is a design signal, not a mechanical target.

When parameters are numerous, decide in this order:

```text
Does the method own too many responsibilities?
        ↓
Do these parameters collectively describe one complete operation?
        ↓
Do they share the same source, lifecycle, and trust boundary?
        ↓
Choose a semantic parameter object or keep independent parameters
```

If a group of parameters collectively describes one complete operation and shares source, lifecycle, and trust boundary, prefer a responsibility-specific object.

For example:

```java
public String signEnvelope(
        RequestPayload payload,
        String password,
        String privateCertificate,
        String publicCertificate,
        String username,
        String ip,
        String userAgent) {
    ...
}
```

If these fields together form the complete internal input of one signing operation, consider:

```java
public String signEnvelope(SignEnvelopeDTO signEnvelope) {
    ...
}
```

Do not combine data with clearly different sources or trust boundaries merely to reach one parameter. For example:

```java
public void audit(PlaceAuditRequest request, Operator operator) {
    ...
}
```

Here:

```text
PlaceAuditRequest
→ external business input

Operator
→ trusted server-side caller context
```

The responsibilities and trust sources differ, so keeping them separate is usually clearer. Never mechanically put trusted server-side identity inside a client Request.

Use real semantic parameter models such as the project's existing:

```text
Request
Query
DTO
BO
Context
Command (when the project already has this model system)
```

Do not create meaningless parameter bags merely to reduce a number:

```text
XxxParam
CommonParam
Object[]
Map<String, Object>
```

For ordinary query conditions, when more than **3** conditions are needed, prefer evaluating a Query object; this is a more specific query-oriented rule.

Framework callbacks, interface overrides, third-party APIs, and stable public APIs are constrained by external contracts; do not break compatibility to satisfy the parameter-count guideline.

Principle:

> Parameter encapsulation should express the complete semantics of an operation. Merge or separate parameters based on responsibility, source, lifecycle, and trust boundary—not the number alone.

### 5.2 Method Length

New or significantly modified methods should normally remain around **80 lines** or less. When a method exceeds that, inspect:

* whether it owns too many responsibilities;
* whether branching is too complex;
* whether abstraction levels are mixed;
* whether logic is duplicated;
* whether independently nameable business steps exist.

80 lines is not a mechanical split command. Do not manufacture many meaningless private methods merely to reduce line count.

### 5.3 Guard Clauses

Prefer returning or throwing early for invalid / exceptional paths to avoid deep nesting, while avoiding fragmentation of simple code merely for style.

---

## 6. Null, Optional, and Return Contracts

Null semantics should be explicit in contracts rather than guessed by callers.

### 6.1 Prefer Empty Collections for No Results

For “zero to many” collection returns, use an empty collection by default to represent no data, not `null`.

When a lower layer already guarantees a non-Null collection, callers should depend on that contract rather than mechanically adding:

```java
list == null ? new ArrayList<>() : list
```

```java
Optional.ofNullable(list).orElseGet(Collections::emptyList)
```

```java
if (list != null) {
    ...
}
```

If a third-party SDK, legacy interface, or other source genuinely permits Null, normalize it once at the boundary closest to the source, then expose a stable contract upward.

Principle:

> Normalize once at the boundary, then let upper layers depend on the stable contract. Do not assume every lower layer is unreliable.

For MyBatis `List<T>` specifics, read `mybatis.md`.

### 6.2 Single-Object Returns Follow Project Contract

A missing single object may be represented by project convention as:

```text
null
Optional
exception
```

Do not mechanically apply collection rules to single-object returns.

### 6.3 Optional

Use `Optional` primarily to express that a return value may be absent. Do not make it the default wrapper for every field, parameter, or collection.

Avoid:

```text
Optional<List<T>>
```

merely to express the ordinary condition that a list can be empty.

### 6.4 Do Not Hide Errors with Defaults

Without a business contract, do not transform:

```text
null → ""
null → 0
exception → default success result
invalid enum → default status
```

under the label of defensive programming.

---

## 7. Equality and Boolean

Prefer:

```java
Objects.equals(a, b)
```

for object equality.

For wrapper `Boolean`, prefer:

```java
Boolean.TRUE.equals(enabled)
Boolean.FALSE.equals(enabled)
```

to avoid unsafe auto-unboxing of possible Null values.

Use `equals`, not `==`, for business string equality.

---

## 8. Collections and Generics

Do not use raw types:

```java
List list;
Map map;
```

Use explicit generics.

Choose a collection implementation based on real use. Do not default to concurrent collections merely for “thread safety”; read `concurrency.md` when state is actually shared across threads.

Collection mutability should follow the project contract. Do not change existing calling semantics without authorization merely to pursue immutability.

Avoid repeated database / remote calls inside loops that create N+1 behavior; read `sql.md` when SQL is involved.

---

## 9. BigDecimal and Exact Numeric Values

Use `BigDecimal` for money, ratios, and other exact business values rather than `double` / `float`.

Prefer constructing decimals with:

```java
BigDecimal.valueOf(0.1)
new BigDecimal("0.1")
```

Avoid:

```java
new BigDecimal(0.1)
```

Division must use an explicit precision / rounding policy rather than depending on accidental behavior.

Use `compareTo` for numeric comparison in most cases. If the business contract cares about scale, use `equals` accordingly.

---

## 10. Time

For new code, prefer `java.time` types:

```text
Instant
LocalDate
LocalDateTime
OffsetDateTime
ZonedDateTime
Duration
```

Choose the type according to whether the business semantics involve a time zone.

Do not express date boundaries with string slicing or manual millisecond arithmetic. Cross-time-zone API and database behavior must follow the project's established contract.

For tests involving current time, prefer a controllable `Clock` or the project's existing time abstraction.

---

## 11. Java Exception Implementation

Cross-layer exception responsibility is defined in `error-handling.md`; this section covers Java implementation details only.

Do not use an empty catch:

```java
catch (Exception ex) {
}
```

Preserve the cause when translating an exception:

```java
throw new StorageAccessException("Failed to load attachment", ex);
```

Do not mechanically repeat:

```text
catch
→ log
→ wrap
→ repeat at every layer
```

Do not catch failures and return unjustified `null`, empty collections, default status, or a success result.

Prefer try-with-resources for resource management.

---

## 12. Logging

Use the target project's standard logging framework and existing format.

Log context that is useful for diagnosis rather than dumping entire objects.

Prefer parameterized logging:

```java
log.info("place created, placeId={}", placeId);
```

over unnecessary string concatenation.

A single exception chain normally needs one full stack trace. Read `error-handling.md` to decide which boundary should log it.

Do not log passwords, Token, Cookie, Secret, private keys, full identity documents, biometric data, or other sensitive information.

Do not mechanically log expected business failures at `error` level.

---

## 13. Comments and Javadoc

Comments should explain:

```text
why something is done
critical business constraints
non-obvious algorithms
external contracts
compatibility reasons
```

Do not use comments to repeat the literal meaning of the code.

Public APIs, complex algorithms, or easy-to-misuse capabilities may deserve Javadoc. Do not mechanically add Javadoc to obvious getters / setters or simple private methods.

Remove stale comments and long-commented-out old code; Git preserves history.

---

## 14. Formatting and Readability

Prefer the target project's existing formatter, Checkstyle, Spotless, IDE configuration, and nearby code style.

If the project has no more specific automated formatting rules, use the defaults below.

### 14.1 Keep a Blank Line Between Adjacent Methods

By default, keep one blank line between adjacent method declarations or implementations in a class, interface, enum, or similar type.

Avoid:

```java
public interface PlaceQueryMapper {
    List<PlaceRecordDO> selectPage(PlaceQuery query);
    long count(PlaceQuery query);
    PlaceRecordDO selectDetail(@Param("id") String id, @Param("scope") String scope);
    PlaceStatsDO selectStats();
    List<PlaceTreeNodeDO> selectTree();
}
```

Prefer:

```java
public interface PlaceQueryMapper {

    List<PlaceRecordDO> selectPage(PlaceQuery query);

    long count(PlaceQuery query);

    PlaceRecordDO selectDetail(@Param("id") String id, @Param("scope") String scope);

    PlaceStatsDO selectStats();

    List<PlaceTreeNodeDO> selectTree();
}
```

Annotations directly attached to a method are part of the method declaration; do not insert meaningless blank lines between an annotation and the method.

Principle:

> Use blank lines to separate independent members and reading units, not to compress multiple declarations into a wall of text.

### 14.2 Prefer One-Line Method Signatures When They Fit Clearly

When a method declaration is clear within the project's line-width rules, keep it on one line. Do not mechanically split a normal two- or three-parameter signature merely because parameters have simple annotations.

Preferred:

```java
PlaceRecordDO selectDetail(@Param("id") String id, @Param("scope") String scope);
```

Wrap when:

* the project's formatter / convention line width would be exceeded;
* there are many parameters;
* parameter types, generics, or annotations are complex;
* a single line materially harms readability.

For example:

```java
int updateStatus(
        @Param("organizationCode") String organizationCode,
        @Param("placeCode") String placeCode,
        @Param("expectedStatus") String expectedStatus,
        @Param("targetStatus") String targetStatus);
```

Once wrapping is necessary, put one parameter per line by default with consistent indentation. Apply the same principle to method calls and constructor calls.

Principle:

> Keep a signature on one line when it remains clear; wrap only when it is genuinely long or complex, not to create vertical space for style alone.

If the project formatter has explicit maximum-width, continuation-indent, or wrapping rules, follow the automated output instead of fighting the formatter.

### 14.3 Use Braces for Control Statements

Use braces consistently. Avoid:

```java
if (condition) return;
```

in complex business code because it increases maintenance risk.

### 14.4 Do Not Expand Unrelated Formatting Changes

Modified code should follow the current file's format, but do not opportunistically format an entire file or module during an unrelated task and create noisy diffs.

---

## 15. Codex Java Change Checklist

When modifying Java code, check:

1. Names express real English business semantics; JavaBeans use responsibility-specific Request / Query / DTO / BO / DO / VO naming rather than generic `Bean / Info / Data / Model`; Query is not moved into a separate `query` Package without basis.
2. When Strategy / Factory / Adapter / Builder / Handler / Command / Visitor or another pattern is genuinely used, type, technical-submodule, and method names expose the real role; patterns are not invented merely to justify names.
3. Business / technical magic values are absent; fixed values needing explanation are represented by responsibility-specific constants or Enum.
4. Constants are grouped by function and semantics rather than placed in one giant `Constants / CommonConstants / GlobalConstants` class.
5. A fixed finite value domain uses Enum when appropriate, while dynamic configuration or external open domains are not forced into Enum.
6. Request / Query / DTO / BO / DO / VO POJOs do not hide defaults in field initializers or `@Builder.Default`.
7. Every new class has a real independent responsibility and existing implementations were searched first.
8. Models follow the project's `class` / Lombok style rather than mechanically adopting `@Data` / `record`.
9. `@Builder` / `@NoArgsConstructor` are used for real construction / framework needs; a class-level Builder has a valid construction path and does not break invariants or business rules.
10. Method parameters are not excessive; encapsulation is based on complete semantics, source, lifecycle, and trust boundaries rather than just reducing a count.
11. Ordinary query conditions are moved to a Query object when appropriate.
12. Long methods are examined for mixed responsibilities rather than split mechanically by line count.
13. Existing non-Null collection contracts are not followed by redundant Null defenses.
14. Defaults or fallbacks do not hide errors.
15. Optional, generics, BigDecimal, and time semantics are correct.
16. catch / throw preserves failure semantics and cause, without duplicate exception logging.
17. Logs do not leak sensitive data.
18. Adjacent methods have clear blank-line separation; signatures remain one line when clearly readable and wrap reasonably only when genuinely long or complex.
19. Formatting and comments follow existing project mechanisms without expanding unrelated diff scope.

Final principle:

> The Java reference implements already-determined responsibilities and contracts clearly and reliably. Architecture, API, transactions, and business semantics are determined by their dedicated references.
