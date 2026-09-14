# AGENTS.md

This file contains only global guardrails and Skill routing. Detailed backend standards live under `java-spring-backend/references` and are loaded on demand.

## Core Rules

1. Before making changes, inspect the relevant code, similar implementations, tests, and available recent Git history. If evidence is unavailable, state that honestly.
2. Make only the minimum changes necessary for the current task. Do not opportunistically refactor, rename, reformat, or upgrade unrelated dependencies.
3. Reuse existing implementations and project conventions before introducing new files, abstractions, dependencies, or design patterns. Determine responsibility before Package and file placement.
4. Do not invent business rules, states, codes, defaults, compatibility behavior, API contracts, permissions, data-scope semantics, transaction semantics, or database constraints.
5. Preserve architectural boundaries and dependency direction. Do not bypass established module / Service boundaries or make lower layers depend back on upper layers.
6. Do not introduce speculative concurrency, asynchronous execution, caching, abstractions, or performance optimizations.
7. Never bypass authentication, authorization, data-permission controls, tenant isolation, or other existing security mechanisms. Do not hard-code or log credentials, Tokens, Secrets, or private keys.
8. After changes, inspect the complete diff and run the project's relevant existing tests, static checks, and architecture checks. Do not disable checks, delete failing tests, weaken assertions, or hide failures merely to make validation pass.
9. Report actual validation results and unverified items honestly. A review request is read-only unless fixes are explicitly requested.
10. Do not rewrite Git history unless the user explicitly requests history restructuring.

## Skill Entry Points

- For Java / Spring Boot / MyBatis / MyBatis-Plus / Rabbit-SQL / PostgreSQL development, fixes, and refactoring, use [java-spring-backend](java-spring-backend/SKILL.md).
- For backend implementation review, use [backend-code-review](backend-code-review/SKILL.md).
- For test quality and executable verification, use [review-and-test](review-and-test/SKILL.md).
- For cleaning up or splitting a completed branch into semantic commits, use [rework-commits](rework-commits/SKILL.md) only when explicitly requested.

Preferred completion flow for non-trivial backend changes:

```text
implementation
→ backend-code-review
→ fix confirmed issues
→ review-and-test
→ rework-commits only when explicitly requested
→ PR / merge
```

## Source and Priority of Rules

Apply rules in this order: explicit requirements of the current task → correctness, security, and data integrity → project architecture rules → applicable Skill references → reasonable consistency with the current module → general language and framework conventions.

[java-spring-backend/references](java-spring-backend/references/) is the single detailed backend standards source for this Skill Pack. Do not duplicate those domain rules in this file. For concrete security implementation, also follow the target project's existing security standards and contracts.
