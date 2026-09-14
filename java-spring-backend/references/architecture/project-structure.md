# Project and Business-Module Structure Standard

This document defines the **physical project structure**, business-module organization, and where responsibility Packages are located.

It answers:

> Where should a business module live, how should code be grouped physically, and when should shared directories such as `common` exist?

It does **not** define the semantics of Controller / Service / Manager / Mapper / Client, model classification, dependency direction, or business-rule placement. Those belong to:

- [Application layering and model boundaries](layering.md)
- [Business rules and use-case boundaries](business-rules.md)

For Java implementation details, read:

- [Java coding](../coding/java.md)

Core principle:

> Organize code by business capability first, then place responsibility Packages inside that business module. Physical structure should make ownership visible without redefining logical responsibilities.

---

## 1. The Target Project Structure Takes Priority

This standard provides defaults only when a new project or business area lacks a clear convention.

If the target project already has a stable structure such as:

```text
feature/<module>
modules/<module>
business/<module>
<module>/controller
```

continue using it. Do not bulk-migrate historical code merely to introduce a preferred directory name.

Evaluate structural changes only when:

- the current task explicitly requires modular restructuring;
- a new module has no established project convention;
- the current structure causes concrete ownership or dependency confusion;
- the compatibility and migration impact are controlled.

Principle:

> A recommended directory is a default, not a migration command.

---

## 2. Prefer Business-First Organization for New Projects

When no stable convention exists, prefer business-first organization:

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

Resources continue to follow the build tool and project convention, for example:

```text
src/main/resources/
src/test/java/
src/test/resources/
```

`module` is only an organizational boundary. It does not imply future microservices and must not be used to justify speculative remote-call abstractions.

If the project already uses another clear business directory name, keep it.

---

## 3. Prefer Business-First to Technology-First Organization

When a new project or business area lacks an existing convention, prefer:

```text
module.place.controller
module.place.service
module.place.mapper

module.casecenter.controller
module.casecenter.service
module.casecenter.mapper
```

rather than scattering each business across global technical directories:

```text
controller.place
controller.casecenter
service.place
service.casecenter
mapper.place
mapper.casecenter
```

Business-first organization improves discoverability, ownership, and module cohesion.

If the target project already stably uses a technology-first layout, do not migrate it without authorization.

---

## 4. Create Only Responsibility Packages That Actually Exist

A business module may physically contain responsibility Packages such as:

```text
module/place/
├── controller/
├── service/
├── manager/
├── mapper/
├── client/
├── adapter/
├── request/
├── dto/
├── bo/
├── domain/
└── vo/
```

This is a location map, not a semantic definition and not a required directory template.

The detailed meaning and default Package placement of Request / Query / DTO / BO / DO / VO and the responsibilities of Controller / Service / Manager / Mapper / Client are defined only by [layering.md](layering.md).

A simple module may contain only the directories it actually needs, for example:

```text
place/
├── controller/
├── service/
├── mapper/
├── request/
├── domain/
└── vo/
```

Do not pre-create empty Packages or placeholder classes for architectural completeness.

Principle:

> `project-structure.md` decides where a responsibility Package lives; `layering.md` decides what that responsibility means.

---

## 5. `common` Is Not a Shared Trash Bin

`common` contains only capabilities that are:

- unrelated to one specific business module;
- genuinely reused across multiple modules;
- semantically stable;
- independent of one module's private implementation.

Suitable examples may include:

```text
common.mybatis.handler
common.mybatis.interceptor
common.json
common.validation
common.web
```

Do not move business code into:

```text
common
util
shared
```

merely because multiple callers use it.

A Place audit rule remains Place business logic even if several entry points need it.

Principle:

> Extract shared code by stable cross-module responsibility, not by apparent reuse alone.

---

## 6. Do Not Mechanically Create Global Catch-All Directories

The project root does not automatically need:

```text
constant
util
handler
interceptor
listener
third
```

Each shared directory must represent a real responsibility.

Examples:

```text
MyBatis TypeHandler
→ common.mybatis.handler

Web Interceptor
→ common.web.interceptor or the project's existing Web infrastructure Package

Place business constants
→ remain inside the Place module that owns their semantics
```

Avoid indefinitely growing catch-all types such as:

```text
GlobalConstants
CommonUtils
ThirdUtils
```

A directory name is not a substitute for responsibility design.

---

## 7. Place Third-Party Integrations by Ownership

An external capability dedicated to one business module should normally remain physically inside that module, for example:

```text
module.place.client.FaceRecognitionClient
module.place.adapter.FaceRecognitionAdapter
```

A genuinely shared integration capability may live under a stable shared technical boundary according to project convention, for example:

```text
common.storage
common.integration
common.client
```

Do not create one universal `third` Package merely because dependencies come from external vendors.

The semantic difference between Client / Adapter / Manager belongs to [layering.md](layering.md).

---

## 8. Physical Structure Must Preserve Logical Boundaries

Directory organization does not redefine or override logical dependency rules.

For Controller / Service / Manager / Mapper / Client dependencies, cross-module calling rules, and responsibility boundaries, use [layering.md](layering.md).

A physical move is valid only when the resulting location still matches the class's logical responsibility.

---

## 9. Decision Flow Before Creating a Module or Package

Decide in this order:

```text
Does the target project already have an equivalent structure?
        ↓
Which business module owns the capability?
        ↓
What logical responsibility does the class have?  ← layering.md
        ↓
Does an implementation already exist?
        ↓
Which responsibility Package owns it?            ← layering.md
        ↓
Where is that Package physically located?         ← this document
        ↓
Is a new directory or class actually necessary?
```

Do not reverse the process by creating a directory template first and then looking for classes to fill it.

---

## 10. Directory-Structure Checklist

When adding modules, Packages, or broad physical moves, check:

1. The target project's existing structure and similar modules were inspected first.
2. An existing project was not forced into a new top-level layout without authorization.
3. New business code remains cohesive rather than scattered across global technical directories.
4. Only responsibility Packages that actually exist were created.
5. Model semantics and Package responsibility were taken from `layering.md`, not reinvented here.
6. `common` contains genuinely shared capabilities rather than business-specific code.
7. Catch-all directories such as `util`, `constant`, or `third` were not created without a stable responsibility.
8. Third-party integrations are physically owned by the correct module or shared technical boundary.
9. Package choice follows the class's logical responsibility rather than file proximity or call convenience.
10. A structural change does not silently become a broad architectural migration.

Final principle:

> Business modules provide physical cohesion. Logical responsibilities and model semantics come from `layering.md`; business-rule placement comes from `business-rules.md`. Keep those concerns separate.
