---
name: rework-commits
description: Rewrite a completed local branch into small, semantic, reviewable Git commits while preserving the final repository tree exactly. Use only when the user explicitly asks to clean up, split, reorder, squash, or reorganize commit history. Never push or force-push unless separately and explicitly requested.
---

# Rework Commits

Reorganize a completed branch into a clean sequence of semantic commits without changing the final repository content.

The invariant is strict:

```text
original final tree == reworked final tree
```

Only commit boundaries, ordering, messages, and metadata may change.

This Skill rewrites local Git history. It is intentionally **explicit-use only**.

---

## 1. Safety Rules

Before changing history:

- read the applicable `AGENTS.md`;
- require explicit user intent to rewrite / clean up commits;
- ensure the working tree and index are clean;
- ensure no merge, rebase, cherry-pick, or conflict resolution is active;
- determine the correct base branch / merge base;
- record the original HEAD and original tree SHA;
- create a local recovery ref before destructive history changes;
- never discard uncommitted changes;
- never push or force-push unless separately requested.

Normal restructuring must not use:

```bash
git reset --hard
```

Do not combine commit cleanup with unrelated rebasing, code changes, formatting, or dependency updates.

---

## 2. Inspect Repository State

Start with commands equivalent to:

```bash
git status --short
git branch --show-current
git log --oneline --decorate -n 30
```

Stop history rewriting if there are:

```text
unstaged changes
staged changes
untracked task files that must be preserved
unresolved conflicts
active merge
active rebase
active cherry-pick
```

Commit cleanup begins from a known clean state.

---

## 3. Determine the Correct Base

Do not assume `main`.

Use repository context to identify the branch against which these feature changes are meant to be reviewed. Typical candidates include:

```text
main
master
develop
release branch
PR target branch
```

Determine the merge base and inspect the commit range.

Useful commands may include:

```bash
git merge-base HEAD <base>
git log --oneline <base>..HEAD
git diff --stat <base>...HEAD
```

If selecting the wrong base could rewrite unrelated history and the correct base cannot be established safely, do not guess.

Rebasing onto a newer base is a separate operation. Do not silently rebase as part of commit cleanup.

---

## 4. Record Recovery Information

Before rewriting, record:

```bash
ORIGINAL_HEAD=$(git rev-parse HEAD)
ORIGINAL_TREE=$(git rev-parse HEAD^{tree})
```

Create a local recovery ref pointing at `ORIGINAL_HEAD`, for example a clearly named temporary backup branch or tag.

Report the recovery ref and `ORIGINAL_HEAD` before resetting history.

Keep the recovery ref until final validation succeeds.

---

## 5. Understand the Complete Change

Inspect the full feature diff and commit history before deciding new boundaries.

Understand dependencies among:

```text
schema / migration
shared contracts / models
backend implementation
API
frontend
configuration
tests
documentation
```

Do not split by file count or directory mechanically. Split by logical responsibility and dependency.

---

## 6. Plan Semantic Commits First

Create a plan before resetting any commit.

Each planned commit should:

- represent one logical change;
- have a nameable purpose;
- contain implementation and directly coupled tests when appropriate;
- avoid unrelated cleanup;
- leave the repository in a coherent state when practical;
- appear after the commits it depends on.

A common dependency order is:

```text
schema / migration
→ shared model / contract
→ persistence / domain capability
→ service / API
→ UI / integration
→ supporting documentation
```

This is guidance, not a fixed template.

Bad commit plans include:

```text
misc changes
fix stuff
cleanup
updates
WIP
```

Prefer intent-revealing entries such as:

```text
Add discount-code persistence model
Implement discount validation rules
Expose discount-code API
Add checkout discount integration
Add regression coverage for expired codes
```

---

## 7. Remove Old Commit Boundaries Without Losing Content

After recovery information and the semantic plan are safe, move the branch back to the selected base while preserving all final file changes.

A typical non-destructive approach is:

```bash
git reset --mixed <base-or-merge-base>
```

The goal is:

```text
old commit boundaries removed
+
all final working-tree content preserved
```

Never use `git reset --hard` for this normal workflow.

---

## 8. Rebuild Commits One Logical Unit at a Time

For each planned commit:

1. select only the files or hunks that belong to the logical change;
2. stage them;
3. inspect `git diff --cached` completely;
4. ensure unrelated changes are excluded;
5. commit with an intent-revealing message;
6. continue to the next logical unit.

If one file contains changes for multiple logical commits, use safe partial staging (`git add -p` when suitable, or an equivalent patch-based staging method). Do not group unrelated hunks merely because they share a file.

After each commit, inspect the remaining working-tree diff so omissions surface early.

---

## 9. Commit Message Quality

Prefer messages that describe the logical outcome:

```text
Add discount code validation rules
Preserve tenant scope when querying places
Separate case registration from case management
```

Avoid vague messages:

```text
Update service
Fix code
Cleanup
Changes
```

Use a body only when it adds useful rationale, constraints, or non-obvious tradeoffs.

---

## 10. Preserve the Final Tree Exactly

After all new commits are created, compute:

```bash
NEW_TREE=$(git rev-parse HEAD^{tree})
```

The strongest success condition is:

```text
NEW_TREE == ORIGINAL_TREE
```

Also verify:

```bash
git diff --exit-code "$ORIGINAL_HEAD" HEAD --
git status --short
git log --oneline <base>..HEAD
```

Expected result:

```text
same final tree
same tracked file contents
clean working tree
new semantic commit history
```

If the tree SHA differs or `git diff "$ORIGINAL_HEAD" HEAD` shows any content difference, validation has failed.

Commit cleanup must never silently change code, configuration, generated artifacts, migrations, documentation, or binary files.

---

## 11. Failure and Recovery

If final-tree validation fails:

- stop immediately;
- do not push;
- keep the recovery ref;
- identify added, removed, or altered content;
- report `ORIGINAL_HEAD` and the recovery ref;
- restore from the recovery ref when recovery is required and permitted.

Do not hide a mismatch by editing the new branch until it merely "looks equivalent." The original final tree is the source of truth.

---

## 12. Cleanup

Only after exact tree equality and a clean working tree are confirmed:

- keep or remove the temporary recovery ref according to the user's preference / repository practice;
- remove temporary planning files that were not part of the original tree;
- do not push automatically.

If the rewritten branch had already been published, explain that updating the remote would require a history-rewriting push and treat that as a separate explicit action.

---

## 13. Final Report

Report:

```text
Commit History Reworked

Base:
Original HEAD:
New HEAD:
Original tree:
New tree:
Recovery ref:

Commits:
1. <commit>
2. <commit>
3. <commit>

Final tree preserved exactly: YES / NO
Working tree clean: YES / NO
Pushed: NO
```

When validation succeeds, state clearly:

> The commit history changed, but the final repository tree is exactly unchanged.

---

## Prohibited Without Separate Explicit Request

Do not:

```text
push
force-push
modify remote branches
delete remote branches
rebase onto a different/newer base
change implementation while restructuring commits
discard user changes
use git reset --hard for the normal workflow
```

Final principle:

> Optimize commit history for reviewability while treating exact preservation of the completed code as a hard invariant.
