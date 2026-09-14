# Spring Boot Coding Standard

This document defines usage standards for Spring Framework / Spring Boot.

This document answers:

> How should Spring MVC, Bean Validation, dependency injection, Bean lifecycle, configuration, Proxy behavior, transaction APIs, async annotations, and Web exception-handling mechanisms be used?

This document does **not** define:

```text
business responsibilities of Controller / Service / Manager / Mapper
HTTP URL / Method / VO / unified-response contracts
whether a transaction is needed and its consistency scope
whether concurrency is worth introducing
exception semantics across layers
```

Read these references instead:

- [Layering](../architecture/layering.md)
- [API](../api/api-design.md)
- [Transactions](../architecture/transactions.md)
- [Concurrency](../architecture/concurrency.md)
- [Exception Handling](../architecture/error-handling.md)

Core principle:

> Spring provides container, proxy, and protocol-framework mechanisms. It does not replace business layering or domain-contract design.

---

## 1. Spring MVC

Controller is an HTTP inbound adapter; read its responsibilities in `layering.md`.

Use the target project's existing style for:

```text
@RestController
@Controller
@RequestMapping
@GetMapping
@PostMapping
@PutMapping
@PatchMapping
@DeleteMapping
```

Choose request binding according to the real HTTP contract:

```text
@PathVariable
@RequestParam
@RequestBody
@ModelAttribute
```

URL, HTTP Method, Request, VO, unified responses, and compatibility are determined by `api-design.md`; the Spring reference does not redefine those contracts.

### 1.1 Placement of Mapping Annotations

This Skill Pack does not mechanically require `@RequestMapping` to appear only on methods, nor does it require it on every class.

Both styles can be reasonable:

```java
@RestController
@RequestMapping("/places")
public class PlaceController {

    @GetMapping("/{id}")
    public Object detail(@PathVariable String id) {
        ...
    }
}
```

or:

```java
@RestController
public class PlaceController {

    @GetMapping("/places/{id}")
    public Object detail(@PathVariable String id) {
        ...
    }
}
```

Choose based on:

* existing project style;
* whether a common prefix is stable;
* whether the final URL is easy to locate;
* whether inheritance / multi-level mappings make the path hard to understand.

When a more specific mapping annotation is available, prefer it instead of mechanically using broad `@RequestMapping` everywhere.

### 1.2 Keep Controllers Focused on the Protocol Boundary

A Spring MVC Controller should contain protocol-boundary code such as:

```text
parameter binding
Bean Validation
reading current request context
calling Service
protocol-layer response construction
```

Do not put database access, business state machines, complex business composition, or business transactions in Controllers.

For how request-specific data such as current user, tenant, or department enters the business layer, read `layering.md`.

### 1.3 OpenAPI / Swagger

If the project already uses OpenAPI / Swagger, keep the real interface contract documentation synchronized.

This Skill Pack does not universally require:

```text
every method to use one fixed documentation annotation
author names to be written in documentation annotations
```

Authors and change history should preferably be maintained by Git.

---

## 2. Bean Validation

Prefer the project's existing Bean Validation system for structural constraints:

```text
@NotNull
@NotBlank
@NotEmpty
@Size
@Min
@Max
@DecimalMin
@Pattern
```

Use:

```java
@Valid
```

for cascading validation of nested objects when needed.

Bean Validation is appropriate for:

```text
non-null / non-blank constraints
length
numeric range
format
structural constraints
```

Rules requiring business meaning or external state—such as business status, authorization, cross-source conditions, or database existence—belong in the business layer. Do not put an entire business flow into `ConstraintValidator`.

### 2.1 Avoid Duplicate Structural Validation

If the same structural constraint has already been reliably enforced at a trusted entry boundary through Bean Validation or an equivalent mechanism, do not mechanically repeat synonymous checks in Service / Manager for:

```text
null
blank
size
pattern
```

For example, if the entry boundary reliably guarantees that `placeCode` is non-blank, the business layer should not repeat `StringUtils.hasText` solely to enforce the same non-blank condition.

However, if a Service has multiple entry points, do not assume every caller has been validated merely because one HTTP Controller uses `@Valid`.

Check first:

1. whether every external entry point performs the required structural validation;
2. whether the project has a unified method-level Validation mechanism;
3. whether the Service method explicitly owns a public input contract.

When validation is missing, fix the real missing boundary instead of scattering duplicate defensive checks through business code.

### 2.2 Do Not Hide Invalid Input with Defaults

For a field already declared required or structurally constrained, do not silently convert without an explicit contract:

```text
null → ""
null → 0
blank → default code
invalid enum → default status
```

Default behavior must come from a real requirement or an established project contract.

### 2.3 `@Validated`

When method-level Bean Validation is needed, use `@Validated` according to the target project's existing approach.

Do not mechanically build three identical layers of validation:

```text
Controller @Valid
+
Service @Validated / @Valid
+
Service hand-written equivalent checks
```

---

## 3. Dependency Injection

Prefer constructor injection for new code.

For example:

```java
@Service
@RequiredArgsConstructor
public class PlaceService {
    private final PlaceManager placeManager;
}
```

An explicit constructor is also fine.

Avoid introducing field injection:

```java
@Autowired
private PlaceManager placeManager;
```

unless the current project has a clear established convention and the task is not an appropriate place to migrate it.

Do not bulk-change unrelated historical classes merely to convert them to constructor injection.

---

## 4. Spring Bean Lifecycle

Components that require container lifecycle, dependency injection, proxying, or framework collaboration should be managed by Spring, for example:

```text
@Service
@Component
@Repository
@Controller / @RestController
@Configuration
```

Ordinary Request / Query / DTO / BO / DO / VO objects, value objects, and pure Java algorithm classes should not be declared Beans without a container need.

Business code must not manually `new` a component that depends on Spring proxy behavior or lifecycle management.

Principle:

> A type becomes a Spring Bean only when it has a real need to collaborate with the container.

---

## 5. Configuration

Environment-specific and mutable configuration belongs in the target project's configuration system.

For structured configuration, consider:

```java
@ConfigurationProperties
```

instead of many scattered `@Value` fields when it better represents a related configuration group.

Do not hard-code in business code:

* environment URLs;
* usernames / passwords;
* Token / Secret;
* private keys;
* environment switches.

If the project uses a configuration center or another binding mechanism, follow the existing approach.

---

## 6. Spring Proxy

Capabilities commonly implemented through Spring Proxy include:

```text
@Transactional
@Async
@Cacheable
@CacheEvict
```

A direct call from one method to another method on the same object may bypass the proxy.

For example:

```java
public void process() {
    audit();
}

@Transactional
public void audit() {
}
```

The presence of the annotation on `audit()` alone does not prove that this self-invocation path goes through a transaction proxy.

When using proxy-based behavior, check:

* whether the Bean is managed by Spring;
* whether the call actually passes through the proxy;
* method visibility;
* the actual JDK / CGLIB / AspectJ mechanism in use;
* whether self-invocation is involved.

Do not mechanically split Beans merely to “make the annotation work.” First evaluate the real responsibility and the project's proxy mechanism.

---

## 7. Spring Transaction Mechanisms

Whether a transaction is needed, who owns the consistency boundary, propagation / isolation / locking, and rollback semantics are defined in `transactions.md`.

On the Spring side, this section only covers implementation mechanisms.

### 7.1 `@Transactional`

Suitable for a clear method-level transaction boundary.

Check:

* whether the call goes through a Proxy;
* whether self-invocation exists;
* whether `rollbackFor` / `noRollbackFor` matches the transaction standard and project contract;
* whether exceptions are swallowed and therefore allow an unintended commit.

Do not mechanically add `@Transactional` merely because a method executes write SQL.

### 7.2 `TransactionTemplate`

Suitable for explicit, local transaction blocks.

For example:

```java
transactionTemplate.executeWithoutResult(status -> {
    ...
});
```

Spring-specific notes:

* it does not depend on `@Transactional` method proxying;
* `rollbackFor` does not apply to `TransactionTemplate`;
* if an exception is caught and swallowed inside the callback, the transaction does not automatically know it should roll back merely because an exception occurred earlier;
* use `status.setRollbackOnly()` only when explicit recovery semantics require it;
* the selected `PlatformTransactionManager`, propagation, isolation, and timeout must match project configuration.

`TransactionTemplate` does not determine whether code belongs in Service or Manager. Transaction ownership is determined by the consistency boundary in `transactions.md`.

Principle:

> Spring determines how a transaction takes effect; the transaction reference determines whether a transaction should exist and what it covers.

---

## 8. `@Async`

When using `@Async`, also read `concurrency.md`.

On the Spring side, check:

* whether the call goes through a Proxy;
* which Executor is used;
* who receives asynchronous failures;
* whether the project has a reliable propagation mechanism for `SecurityContext` / MDC / ThreadLocal.

Do not assume transaction or request context propagates automatically to an async thread.

---

## 9. Cache Annotations

When using `@Cacheable` / `@CacheEvict` and related annotations, check:

* whether calls go through the Proxy;
* whether keys are stable;
* whether cache-consistency semantics have a real business basis;
* whether self-invocation prevents the annotation from taking effect.

Do not mechanically add caching for speculative performance gains.

---

## 10. Web Exception Handling

When the project already has unified Web exception handling, ordinary Controllers should not repeatedly write:

```java
try {
    service.execute();
} catch (Exception ex) {
    ...
}
```

Prefer to reuse:

```text
@RestControllerAdvice
@ControllerAdvice
@ExceptionHandler
HandlerExceptionResolver
```

For where exceptions are converted / logged, read `error-handling.md`; for HTTP error structures, read `api-design.md`.

---

## 11. HTTP Semantics Stop at the Web Boundary

Business Service / Manager should normally not depend directly on:

```text
HttpStatus
ResponseEntity
HttpServletRequest
HttpServletResponse
```

This is a layering rule; read `layering.md` for details.

If an infrastructure class is itself a Web technical component, judge it by its real responsibility rather than mechanically by type name.

---

## 12. Do Not Overuse Spring

Do not mechanically add for appearance:

```text
@Component
@Service
@Bean
@Configuration
Event / Listener
AOP
custom Starter
```

Spring solves container and framework-collaboration problems. It does not give every Java class a framework identity.

---

## 13. Codex Spring Checklist

When modifying Spring code, check:

1. Controller owns only protocol-boundary responsibilities.
2. Mapping annotations follow the project's existing mechanism and the final URL is clear.
3. Bean Validation expresses structural constraints and is not mechanically duplicated in business code.
4. In multi-entry scenarios, the real validation boundary is complete.
5. Invalid input is not hidden by defaults.
6. Dependency injection and Bean lifecycle are reasonable.
7. Environment configuration and credentials use the correct configuration mechanism.
8. `@Transactional` / `@Async` / Cache annotations actually go through a Proxy.
9. Layering is not being determined backward from the chosen Spring transaction API; `TransactionTemplate` remains an implementation mechanism only.
10. Web exceptions reuse unified Advice / Handler infrastructure.
11. Business layers do not depend on HTTP types without a real reason.
12. The task does not expand into unrelated changes merely to impose Spring style.

Final principle:

> The Spring reference maintains framework mechanisms only. The business meaning of layering, API, transactions, concurrency, and exceptions is uniquely maintained by the corresponding dedicated references.
