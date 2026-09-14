# AGENTS.md

This file contains only global guardrails, Skill routing, and the preferred completion pipeline. Detailed backend standards live under `java-spring-backend/references` and are loaded on demand.

## Core Rules

1. Before making changes, inspect the relevant code, similar implementations, tests, and available recent Git history. If evidence is unavailable, state that honestly.
2. Make only the minimum changes necessary for the current task. Do not opportunistically refactor, rename, reformat, or upgrade unrelated dependencies.
3. Reuse existing implementations and project conventions before introducing new files, abstractions, dependencies, or design patterns. Determine responsibility before Package and file placement.
4. Do not invent business rules, states, codes, defaults, compatibility behavior, API contracts, permissions, data-scope semantics, transaction semantics, or database constraints.
5. Preserve architectural boundaries and dependency direction. Do not bypass established module / Service boundaries or make lower layers depend back on upper layers.
6. Do not introduce speculative concurrency, asynchronous execution, caching, abstractions, or performance optimizations.
7. Never bypass authentication, authorization, data-permission controls, tenant isolation, or other existing security mechanisms. Do not hard-code or log credentials, Tokens, Secrets, or private keys.
8. After implementation, inspect the complete diff. Formal implementation review belongs to `backend-code-review`; executable tests, static checks, architecture checks, and builds belong to `review-and-test`. Do not disable checks, delete failing tests, weaken assertions, or hide failures merely to make validation pass.
9. Report actual review and verification results honestly. A pure review request is read-only unless fixes are explicitly requested.
10. Do not rewrite Git history unless the user explicitly requests history restructuring.

## Skill Entry Points

- For Java / Spring Boot / MyBatis / MyBatis-Plus / Rabbit-SQL / PostgreSQL implementation, fixes, and refactoring, use [java-spring-backend](java-spring-backend/SKILL.md).
- For backend implementation review, use [backend-code-review](backend-code-review/SKILL.md).
- For executable verification, test-quality assessment, and repository checks, use [review-and-test](review-and-test/SKILL.md).
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

Each Skill owns only its stage. Do not duplicate another Skill's workflow inside the current Skill.

## Source and Priority of Rules

Apply rules in this order:

```text
explicit requirements of the current task
→ correctness, security, and data integrity
→ target-project architecture and contracts
→ applicable Skill references
→ reasonable consistency with the current module
→ general language and framework conventions
```

[java-spring-backend/references/routing.md](java-spring-backend/references/routing.md) is the single standards-routing index for this Skill Pack. The referenced domain documents are the single detailed source for their own subjects; do not duplicate those rules in `AGENTS.md` or sibling Skills.

For authentication, authorization, tenant isolation, data scope, and other security behavior, also follow the target project's existing security standards and contracts.
