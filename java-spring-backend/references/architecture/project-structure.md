# Project and Business-Module Structure Standard

This document defines the physical directory structure, business-module organization, and internal Package structure of Java backend projects.

It answers:

> Where should a new module live, by what dimension should business code be organized, and what should directories such as `module` / `common` contain?

For logical class responsibilities, dependency direction, model classification, and concrete Package semantics, read:

- [Application layering and model boundaries](layering.md)

For Java implementation details, read:

- [Java coding](../coding/java.md)

Core principle:

> Organize code by business capability first, then layer by responsibility inside each business module. Directory structure exists to improve discoverability, boundaries, and collaboration. Do not create empty Packages for formality, and do not migrate an existing stable project structure without authorization.

---

## 1. The Target Project Structure Takes Priority

This standard provides a default organization when a new project or new module lacks an explicit convention.

If the target project already has a stable structure such as:

```text
feature/<module>
modules/<module>
business/<module>
<module>/controller
```

continue using it. Do not bulk-migrate historical code merely to introduce a `module` directory.

Evaluate structural changes only when:

* the current task explicitly requires modular refactoring;
* a new module has not yet been implemented and the project has no unified convention;
* the existing structure already creates clear responsibility confusion or dependency problems;
* the scope and compatibility impact can be controlled.

Principle:

> A recommended directory is a default, not a migration command.

---

## 2. Prefer Business-First Organization for New Projects

When the target project has no existing convention, prefer business-first modular organization.

Example:

```text
src/main/java/com/example/app/
├── common/                 genuinely cross-business shared capabilities
├── config/                 application-level framework configuration
├── module/                 business modules
│   ├── place/
│   ├── casecenter/
│   └── equipment/
└── Application.java
```

Resources continue to follow Maven / Gradle and the project's existing layout, for example:

```text
src/main/resources/
src/test/java/
src/test/resources/
```

`module` expresses an internal business-organization boundary. It does not imply that the module will later become a microservice, and it must not be used as a reason to introduce remote-call abstractions for speculative future decomposition.

If the project already uses another clear business directory name, continue using it. Do not force-renaming to `module`.

---

## 3. Prefer Business-First to Technology-First Organization

When a new project or new business area lacks an existing convention, prefer:

```text
module.place.controller
module.place.service
module.place.mapper

module.casecenter.controller
module.casecenter.service
module.casecenter.mapper
```

rather than first creating global technical directories and scattering every business across them:

```text
controller.place
controller.casecenter
service.place
service.casecenter
mapper.place
mapper.casecenter
```

Business-first organization helps:

* locate a business's complete implementation quickly;
* make module boundaries and cross-module dependencies visible;
* control growth of common code;
* keep related Controllers, Services, Mappers, models, and tests close together.

However, if the target project already stably uses a technology-first layout, do not migrate it without authorization.

---

## 4. Organize a `module` by Real Responsibilities

A business module may contain only the responsibilities it actually needs:

```text
module/place/
├── controller/             HTTP inbound adapters
├── service/                business use cases and flows
├── manager/                optional: reusable application capabilities / atomic compositions
├── mapper/                 database access
├── client/                 optional: external technical calls
├── adapter/                optional: external protocol adaptation
├── request/                API input: Request / Query
├── dto/                    optional: internal data transfer
├── bo/                     optional: intermediate business objects
├── domain/                 persistence DO
└── vo/                     concrete business output
```

`Request` and `Query` have different model semantics but share the `request` Package by default:

```text
module.place.request.PlaceSaveRequest
module.place.request.PlaceAuditRequest
module.place.request.PlaceQuery
```

Express the input type through `*Request` / `*Query` class names. Do not mechanically create a separate `query` Package merely to mirror the model classification.

This is a responsibility map, not a requirement that every module contain every directory.

A simple module may contain only:

```text
place/
├── controller/
├── service/
├── mapper/
├── request/
├── domain/
└── vo/
```

Add these only when the responsibility genuinely exists:

```text
manager
dto
bo
client
adapter
```

Do not pre-create large numbers of empty Packages or placeholder classes merely for "directory completeness."

The single detailed source of truth for model responsibilities is `layering.md`. Request and Query share `request` by default but remain semantically distinct through their class names. DTO, BO, DO, and VO continue to live according to their real responsibilities and must not be mechanically stuffed into generic `domain` or `dto` directories.

---

## 5. `common` Is Not a Shared Trash Bin

`common` contains only capabilities that genuinely satisfy all of the following:

* unrelated to any specific business module;
* have clear cross-module reuse value;
* have stable semantics;
* do not depend on the internal implementation of a business module.

Suitable examples may include:

```text
common.mybatis.handler
common.mybatis.interceptor
common.json
common.validation
common.web
```

Concrete names still follow the target project's conventions.

Do not move business code into:

```text
common
util
shared
```

merely because "multiple places can call it."

For example, a Place audit rule remains a Place business capability even when reused by multiple entry points. It should not become `common.util.PlaceAuditUtils`.

Principle:

> Extract common code according to stable cross-module responsibility, not merely because it looks reusable.

---

## 6. Do Not Mechanically Create Global `constant` / `util` / `handler` / `third`

The project root is not required to contain:

```text
constant
util
handler
interceptor
listener
third
```

Such directories must be justified by real responsibilities.

For example:

```text
MyBatis TypeHandler
→ common.mybatis.handler

Web Interceptor
→ common.web.interceptor or the project's existing Web infrastructure Package

Place business constants
→ a location inside the Place module that owns those business semantics
```

Avoid indefinitely growing catch-all types such as:

```text
GlobalConstants
CommonUtils
ThirdUtils
```

A directory name is not a substitute for responsibility design.

---

## 7. Own Third-Party Integrations by Boundary, Not by a Universal `third` Package

External capabilities dedicated to one module should normally follow that business module, for example:

```text
module.place.client.FaceRecognitionClient
module.place.adapter.FaceRecognitionAdapter
```

Genuinely shared external infrastructure may live under a stable common boundary according to the project's conventions, for example:

```text
common.storage
common.integration
common.client
```

Do not put every SDK, HTTP Client, Redis integration, OSS integration, or messaging integration into one giant `third` Package merely because the dependency comes from a third party.

For the difference between technical adapters and Manager responsibilities, read:

- [layering.md](layering.md#5-manager-layer)

Principle:

> Own an external dependency according to who owns the technical capability and whether it is reused across modules, not simply according to whether it is third-party.

---

## 8. Keep MVC / Application Layering Unidirectional Inside a Module

Business-module directories still obey logical layering:

```text
Controller / other inbound adapters
        ↓
      Service
        ↓
    Manager (when needed)
      ↙       ↘
   Mapper    Client / Adapter
```

Directory structure is not a reason to bypass dependency rules.

Even when everything lives under:

```text
module.place
```

these are still forbidden:

```text
Controller → Mapper
Mapper → Service
Client → Service
```

Read detailed responsibilities in:

- [layering.md](layering.md)

---

## 9. Cross-Module Calls Must Not Pierce the Data-Access Layer

Business modules should normally collaborate through the other module's stable Service / Facade capability.

Recommended:

```text
module.casecenter.CaseService
        ↓
module.place.PlaceService / PlaceFacade
```

Avoid:

```text
CaseService
    ↓
PlaceMapper
```

One purpose of the `module` directory is to make cross-module dependencies easier to identify, not to let any code reach into another module's internals merely because everything runs in the same JVM.

---

## 10. Decision Flow Before Creating a Module / Package

Before adding a directory or class, decide in order:

```text
Does the current project already have an equivalent structure?
        ↓
Which business module owns it?
        ↓
What responsibility does this class have?
        ↓
Does an implementation of that responsibility already exist?
        ↓
Which Package should own it?
        ↓
Is a new directory / class actually necessary?
```

Do not reverse the process:

```text
create controller/service/manager/mapper directories first
        ↓
then look for classes to put in them
```

---

## 11. Codex Directory-Structure Checklist

When adding a new module, adding new Packages, or moving code broadly, check:

1. Whether the target project's existing directories and similar business modules were inspected first.
2. Whether an existing project was forced into a `module` layout without authorization.
3. Whether new business code remains cohesive instead of being scattered across global technical directories.
4. Whether only genuinely needed responsibility Packages were created inside a `module`.
5. Whether Query was mechanically split into a separate `query` Package merely because of its model semantics; by default it belongs in `request` together with Request.
6. Whether all models were mechanically placed into `domain` / `dto`.
7. Whether `common` contains specific business semantics or has become a common trash bin.
8. Whether giant fallback directories such as `util`, `constant`, or `third` were created without justification.
9. Whether third-party integrations live at the correct Client / Adapter or shared technical boundary.
10. Whether cross-module calls reach through to another module's Mapper.
11. Whether Package choice follows the class's real responsibility rather than current file location or call convenience.

Final principle:

> Business modules provide business cohesion; responsibility Packages express boundaries. Request and Query share the `request` Package by default and are distinguished by class names. `common` contains only genuinely shared capabilities. Clear structure matters more than having many directories.
