# Exception Handling and Error-Boundary Standard

This document defines how exceptions propagate, are translated, logged, and exposed across Mapper / Client / Manager / Service / Web / API boundaries.

It answers:

> Where should an exception be translated, where should business context be added, where should the full failure context be logged, and where should it be converted into an external error contract?

For Java `catch` / `throw` / logging APIs, read:

- [Java](../coding/java.md)

For Spring Web exception-handling mechanisms, read:

- [Spring](../coding/spring.md)

For HTTP error contracts, read:

- [API](../api/api-design.md)

Core principle:

> Handle exceptions only at boundaries that genuinely own recovery, abstraction translation, contextual enrichment, failure recording, or external representation. Do not mechanically add `catch + log + wrap` at every layer.

---

## 1. Typical Exception Flow

```text
Database / External System
        ↓
Mapper / Client / Adapter
        ↓ isolate technical exceptions when necessary
Manager (optional)
        ↓ translate application-capability semantics when necessary
Service
        ↓ add business-use-case context
Web / API Boundary
        ↓ convert to a safe, stable error contract
Client
```

Not every failure must pass through every layer. A simple flow may be:

```text
Mapper → Service → Web Error Handler
```

Handle an exception only where a real boundary exists.

---

## 2. Mapper / DAO

Mapper / DAO owns data access and does not own repeated logging of business-failure context.

In Spring + MyBatis projects, prefer the framework's existing data-access exception translation. Do not mechanically add this around every Mapper call:

```text
catch Exception
→ DAOException
```

Translate only when there is a real technical-isolation requirement, and preserve the original cause.

Mapper normally should not repeatedly log the full stack trace because upper layers usually have more valuable business context.

Principle:

> Mapper owns data access and necessary technical exception isolation; it does not need to prove that an exception passed through this layer.

---

## 3. Client / Adapter

Outbound adapters such as HTTP Client, RPC Client, SDK Adapter, and Object Storage Client should isolate vendor and protocol details.

For example:

```text
VendorSdkTimeoutException
        ↓
FaceRecognitionUnavailableException
```

The goal of translation is to let upper layers depend on stable project-owned semantics rather than vendor-specific:

```text
exception classes
status codes
Request / Response types
SDK types
```

Do not flatten every third-party exception into one generic exception that loses useful failure meaning.

Principle:

> Exception translation in Client / Adapter corresponds to a real technical abstraction boundary.

---

## 4. Manager

Manager is an optional application-capability layer inside the current process.

It may:

* recover a recoverable failure within its responsibility;
* translate an exception when the abstraction meaning actually changes;
* propagate the exception when it cannot handle it;
* preserve necessary context for an atomic application capability.

Do not mechanically create chains such as:

```text
RuntimeException
→ ManagerException
→ ServiceException
```

merely because an exception passed through a Manager.

If an application capability is later split into a remote service, it has become **another application boundary**. The caller should access that remote contract through Client / Adapter; the remote service then handles its own Service / Manager / Mapper / Web or RPC exception boundaries internally.

Do not continue treating a remote service as an "independently deployed Manager" inside the current application.

Principle:

> Manager represents an application capability inside the current process; once it crosses a process boundary, it becomes a new application boundary and an external-call contract.

---

## 5. Service

Service understands the current business use case best and is an important place to add business-failure context.

Necessary and safe context may include:

```text
business action
resource identifier
key business number
current processing stage
TraceId / RequestId, when the project has one
```

This does not mean every Service method should contain:

```java
catch (Exception ex) {
    log.error(..., ex);
    throw ex;
}
```

If the project already has a unified exception handler, AOP, or application boundary that records unhandled exceptions with enough context, Service should not log the same stack trace again.

Expected business failures should not mechanically be logged as system `error`s.

Principle:

> Preserve one sufficiently informative failure record at the boundary with the most useful business context, while avoiding duplicate logging.

---

## 6. Web / API Boundary

Java stack traces, internal exception classes, SQL, server paths, and vendor technical details must not cross the external protocol boundary directly.

In Spring MVC, prefer the project's shared mechanisms such as:

```text
@RestControllerAdvice
@ExceptionHandler
HandlerExceptionResolver
```

rather than handwritten `try/catch` in every Controller.

External error codes, messages, HTTP Status, and unified response structures are defined by `api-design.md`.

Principle:

> Exceptions may propagate internally; an external protocol boundary must convert them into a safe, stable error contract.

---

## 7. Exception Translation Must Preserve the Cause

Recommended:

```java
throw new StorageAccessException(
        "Failed to load attachment",
        ex);
```

Avoid:

```java
throw new StorageAccessException(
        "Failed to load attachment");
```

Do not create meaningless layer-by-layer wrapping:

```text
SQLException
→ DAOException
→ ManagerException
→ ServiceException
→ ApiException
```

Principle:

> Translate an exception to isolate abstractions, not to mirror the call stack.

---

## 8. Usually Log One Full Stack Trace per Exception Chain

Avoid:

```text
Mapper log.error
→ Manager log.error
→ Service log.error
→ ControllerAdvice log.error
```

which produces duplicate alerts and log noise.

Prefer a responsibility split such as:

```text
Mapper / Client / Manager
→ translate when necessary; do not mechanically print the full stack trace

Service / application boundary
→ add business context when necessary

unified exception handler
→ if the project logs centrally here, upstream layers do not repeat it
```

The exact logging location follows the target project's existing logging architecture.

---

## 9. Do Not Swallow Exceptions or Pretend Success

Forbidden:

```java
catch (Exception ex) {
}
```

Also forbidden without a business contract:

```java
catch (Exception ex) {
    log.error("failed", ex);
    return null;
}
```

or returning:

```text
empty collection
0
false
default object
default state
```

to turn a system failure into a normal business result.

For async fallback recovery semantics, read `concurrency.md`.

---

## 10. Transactions and Exceptions

Catching, translating, or swallowing an exception can change transaction rollback behavior.

When transactions are involved, confirm:

* whether the current exception should trigger rollback;
* whether the translated exception still satisfies the rollback contract;
* whether catching without rethrowing causes an unintended commit;
* whether a `TransactionTemplate` callback accidentally swallows failure.

Read detailed rules in `transactions.md`.

---

## 11. Sensitive Information

Exceptions and logs must not directly record or return:

* passwords;
* Tokens;
* Cookie / Session credentials;
* private keys / Secrets;
* full identity documents;
* biometric data;
* unmasked sensitive personal information;
* third-party authentication credentials.

Do not leak sensitive fields through an object's full `toString()` output.

---

## 12. Codex Exception Checklist

When exceptions are involved, check:

1. Whether the current layer genuinely owns recovery, translation, contextual enrichment, logging, or external representation.
2. Whether it is merely mechanical `catch + log + throw`.
3. Whether exception translation corresponds to a real abstraction change and preserves the cause.
4. Whether the same exception chain is logged with a full stack trace multiple times.
5. Whether Mapper / Client leaks low-level technical exceptions unnecessarily.
6. Whether Manager creates meaningless exception layers.
7. Whether Service preserves necessary and safe business context.
8. Whether Web / API converges exception handling through a shared boundary.
9. Whether SQL, stack traces, paths, vendor details, or sensitive information leak externally.
10. Whether catch logic changes failure semantics through defaults / fallback.
11. Whether exception handling changes transaction rollback or the caller's established exception contract.

Final principle:

> Lower layers perform necessary technical isolation; business boundaries add real business semantics; the application boundary records one sufficient failure context; the external protocol boundary converts failures into a safe, stable error contract.
