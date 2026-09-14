---
name: java-spring-backend
description: Develop, fix, and refactor Java, Spring Boot, MyBatis, MyBatis-Plus, Rabbit-SQL, and PostgreSQL code according to team backend standards. Use it for backend implementation work. Use backend-code-review for formal code review and review-and-test for executable verification.
---

# Java Spring Backend

Turn the current backend task into the smallest implementation change that fits the target project's existing design.

This Skill owns:

```text
implementation context
responsibility analysis
standards selection
minimum code change
implementation reporting
```

It does **not** own the formal code-review workflow or the executable verification pipeline. Those belong to the sibling `backend-code-review` and `review-and-test` Skills.

Detailed backend rules live under `references`. Use [references/routing.md](references/routing.md) as the single routing index and load only the standards needed by the current task.

---

## Before You Start

- Read the applicable `AGENTS.md` in the target project and follow the user's current requirements and execution permissions.
- Inspect relevant code, similar implementations, tests, build configuration, and available recent Git history before changing code. If evidence is unavailable, state that honestly.
- Continue using stable project contracts, framework mechanisms, directory structures, and naming conventions when they are already established.
- Do not bulk-migrate historical code merely because this Skill Pack recommends a different default.
- Route pure review work to [backend-code-review](../backend-code-review/SKILL.md).
- Route executable validation work to [review-and-test](../review-and-test/SKILL.md).

---

## Workflow

1. **Clarify the task.** Determine the requested behavior, affected modules, acceptance criteria, and constraints. Ask only about critical business intent that cannot be established from requirements, code, contracts, tests, or stable project conventions.
2. **Build context.** Read the relevant implementation, callers, models, persistence code, configuration, tests, and available recent history.
3. **Select standards.** Use [references/routing.md](references/routing.md) to load only the references actually involved. If the task expands into another domain, load that reference then.
4. **Determine responsibilities.** Before adding or moving files, determine the business module, logical responsibility, model type when applicable, dependency direction, and responsibility Package. Then determine the physical location.
5. **Implement the minimum change.** Reuse existing capabilities. Do not invent business states, codes, defaults, compatibility behavior, permissions, transaction semantics, database constraints, or abstractions.
6. **Inspect the complete implementation diff.** Confirm that the intended files changed, unrelated edits were not introduced, new files are necessary, and the implementation remains within the authorized scope. This is an implementation-scope inspection, not a replacement for `backend-code-review`.
7. **Hand off completion stages.** For non-trivial completed changes, follow the pipeline defined by `AGENTS.md`: `backend-code-review` first, fix confirmed issues within scope, then `review-and-test`. Use `rework-commits` only when explicitly requested.
8. **Report the implementation.** Explain what changed, why, important file locations, unresolved business questions, and which companion stages were or were not executed. Never report review or verification as passed unless the owning stage actually produced that result.

---

## Standards Selection

Use [references/routing.md](references/routing.md) rather than maintaining another routing table here.

Typical examples:

```text
New business module
→ Project structure + Layering

New Request / Query / DTO / BO / DO / VO
→ Layering + Java

Repeated stable business-state rule
→ Business rules + Layering + Java

Controller HTTP-contract change
→ API + necessary Spring

Mapper XML with changed SQL semantics
→ MyBatis + SQL

@XQLMapper / .xql with changed SQL semantics
→ Rabbit-SQL + SQL

Table-column and persistence-model change
→ Database design + matching persistence reference

Transaction boundary change
→ Transactions + necessary Layering / Spring

CompletableFuture database work
→ Concurrency + Transactions

Bug fix that requires regression coverage
→ relevant domain reference + Testing
```

Principle:

> Load the smallest necessary union. Do not load all references "just to be safe."

---

## Responsibility Check Before Creating a File

Before adding a file, answer in order:

```text
Which business module owns it?
        ↓
What logical responsibility does it have?
        ↓
Can an existing implementation be reused?
        ↓
If it is a model, what model responsibility does it have?
        ↓
Which responsibility Package should contain it?
        ↓
Where is that Package physically located in the target project?
        ↓
Is a new file actually necessary?
```

Ownership of these decisions is explicit:

```text
physical business-module location
→ references/architecture/project-structure.md

logical responsibilities, dependency direction, model classification, responsibility Packages
→ references/architecture/layering.md

core business rules vs application use-case rules
→ references/architecture/business-rules.md
```

---

## Implementation Guardrails

- Make only changes necessary for the current task.
- Reuse project mechanisms before introducing new files, abstractions, dependencies, or design patterns.
- Preserve architectural boundaries and dependency direction.
- Do not speculate about concurrency, caching, async execution, compatibility behavior, or future architecture.
- Never bypass authentication, authorization, tenant isolation, data scope, or other existing security controls.
- Do not hard-code or log credentials, Tokens, Secrets, or private keys.
- Do not disable checks, delete failing tests, weaken assertions, or change expected output merely to accommodate an implementation defect.

These guardrails do not replace their detailed owning references or the target project's own contracts.

---

## Delivery Requirements

An implementation-stage report should state:

- the purpose of the change;
- the key implementation behavior;
- important files added or modified;
- any business assumptions that could not be confirmed;
- whether `backend-code-review` was executed and its actual result, if available;
- whether `review-and-test` was executed and its actual result, if available;
- remaining real risks or unverified items.

Do not reproduce the review finding format or verification PASS/FAIL format here. Those formats belong to their owning Skills.

Final principle:

> This Skill implements the change; references define detailed backend rules; `backend-code-review` evaluates the completed implementation; `review-and-test` proves it with executable checks.
