---
description: "PRD Step 7/7 — Creates a pull request from the feature branch to main (or a specified target). Reads prd.md, spec.md, plan.md, and tasks/index.md to build a detailed PR description. Runs final tsc + test verification before opening the PR."
argument-hint: <target branch (optional) — defaults to main or master if omitted>
allowed-tools: Bash(git:*), Bash(gh:*), Bash(bun:*), Bash(cat:*), Bash(ls:*)
---

```xml
<role>
You are a senior engineer preparing a pull request for a completed feature. You write PR descriptions that give reviewers full context without forcing them to read every implementation file. You link decisions to requirements, surface breaking changes prominently, and make the test plan specific enough to follow.
</role>

<context>
Target branch: $ARGUMENTS (defaults to main or master if not provided)
Prime directive: The PR description must be self-contained — a reviewer who has not seen this feature should understand what was built, why, and how to verify it.
</context>
```

## Feature Branch / PRD Folder
$ARGUMENTS

---

## Phase 1 — Pre-flight Checks

### 1a — Confirm on the Feature Branch

```bash
git branch --show-current
```

The current branch must be `feat/{feature-slug}-impl` (created by `/prd-implement`).

If you are on `main`, `master`, or any non-feature branch — stop:

```
⛔ Not on a feature branch.
   Switch to the feature branch first:
   git checkout feat/{feature-slug}-impl

   Then re-run /prd-pr.
```

### 1b — Confirm All Tasks Are Complete

Find the PRD folder from the current branch name or `$ARGUMENTS`, then:

```bash
cat {prd-folder}/tasks/index.md
```

Scan for any unchecked task lines (containing `[ ]` without `[x]`):

```bash
grep "\- \[ \]" {prd-folder}/tasks/index.md
```

If any unchecked tasks remain — stop:

```
⛔ Feature is not complete. The following tasks are still pending:
   {list of unchecked task lines}

   Complete all tasks with /prd-implement before opening a PR.
```

If all tasks show `[x] ✅` — proceed.

### 1c — Final Verification

```bash
# Type check — must be zero errors
bun tsc --noEmit 2>&1 | head -30

# Test suite — must be green
bun test 2>&1 | tail -20
```

If either fails — fix before continuing. Do not open a PR with broken TypeScript or failing tests.

---

## Phase 2 — Read PRD Documents

Locate the PRD folder. From the feature branch name (`feat/{slug}-impl` → slug) or `$ARGUMENTS`:

```bash
ls docs/prd/ | grep {slug}
# or find the folder the tasks live in
find . -name "index.md" -path "*/tasks/*" | head -5
```

Read each document in order and extract the sections listed below. Do not read more than what is specified — extract only what the PR description needs.

### 2a — From `prd.md`

Extract:
- `feature` from YAML frontmatter → PR title base
- `<overview><problem>` → "Problem" section of PR
- `<overview><solution>` → "Solution" section of PR
- `<overview><success_metrics>` → "Success Metrics" section
- `<scope><in>` → "What's Included"
- `<scope><out>` → "What's NOT Included"
- `<background><backward_compat>` (if present) → "Breaking Changes" section
- `<user_stories>` → condensed list for "Who This Affects"
- `<non_functional_requirements>` → any NFRs worth calling out in the PR

### 2b — From `spec.md`

Extract:
- `<architecture><decisions>` → "Key Decisions" section
- `<schema><new_models>` + `<schema><existing_changes>` → "Schema Changes" (if any)
- `<api>` routes → list of new/modified endpoints (method + path + one-line purpose)
- `<components>` → list of new/modified UI components (if any)
- `<backward_compat_layer>` or `<architecture><frozen_contracts>` → feeds "Breaking Changes"

### 2c — From `plan.md`

Extract:
- YAML `feature`, `prd_file`, `spec_file` → links in PR description
- `<summary><method>` → TDD or standard
- `<summary><total_steps>` → total tasks completed
- `<definition_of_done>` → checklist items for the PR

### 2d — From `tasks/index.md`

Extract:
- Progress line (e.g. `Progress: 8 / 8 tasks complete`)
- Full list of `[x] ✅` task lines → "Tasks Implemented" section
- The `Plan:` link at the top for the PR doc links

---

## Phase 3 — Determine Target Branch

```bash
# If $ARGUMENTS is provided — use it as the target
TARGET="${ARGUMENTS:-}"

# If not provided — detect the repo default branch
if [ -z "$TARGET" ]; then
  TARGET=$(git remote show origin 2>/dev/null | grep "HEAD branch" | awk '{print $NF}')
  # Fallback if no remote
  if [ -z "$TARGET" ]; then
    TARGET="main"
  fi
fi

echo "Target branch: $TARGET"
```

Confirm the target branch exists:

```bash
git branch -r | grep "origin/$TARGET"
```

If it does not exist — ask the user which branch to target before continuing.

---

## Phase 4 — Push the Feature Branch

```bash
# Check if the remote branch already exists
git ls-remote --heads origin feat/{feature-slug}-impl
```

If not yet pushed:

```bash
git push -u origin feat/{feature-slug}-impl
```

If already pushed — ensure it is up to date:

```bash
git push origin feat/{feature-slug}-impl
```

---

## Phase 5 — Build PR Title and Description

### PR Title

Format: `feat({feature-slug}): {short description from prd.md solution sentence}`

Rules:
- ≤ 72 characters
- Imperative mood ("add", "implement", "introduce" — not "added" or "adds")
- Scope matches the feature slug

### PR Description

Compose the full body using the content extracted in Phase 2. Use this exact structure:

```markdown
## Summary

{2-3 sentences: what this PR implements, for whom, and what outcome it enables.
Derived from prd.md `<overview><solution>`.}

---

## Problem

{From prd.md `<overview><problem>` — what existed before, who was affected, cost of not solving it.}

---

## Solution

{From prd.md `<overview><solution>` — what was built and how it solves the problem.}

---

## What's Included

{Numbered list from prd.md `<scope><in>` — concrete deliverables only.}

## What's NOT Included

{Numbered list from prd.md `<scope><out>` — explicit exclusions and why.}

---

## Key Implementation Decisions

{From spec.md `<architecture><decisions>` — each decision with rationale.}

| Decision | Choice | Rationale |
|---|---|---|
| {e.g. Session storage} | {e.g. DB-backed, not JWT} | {e.g. Required for server-side revocation} |

---

## Schema Changes

{From spec.md `<schema>` — only if schema changed. Omit section entirely if no DB changes.}

**New models:** {list}
**Modified models:** {list — additive only changes only}
**Migration command:** `{exact command}`

---

## API Changes

{From spec.md `<api>` — only if routes were added or modified. Omit section if no API changes.}

| Method | Path | Purpose | Auth |
|---|---|---|---|
| {GET} | {/api/...} | {one sentence} | {required / public} |

---

## UI Changes

{From spec.md `<components>` — only if UI was changed. Omit section if no UI changes.}

{List of new/modified components with their purpose.}

---

## Breaking Changes

{From spec.md `<architecture><frozen_contracts>` or prd.md `<background><backward_compat>`.
If no breaking changes — write: "None — this is a fully additive change."}

---

## Tasks Implemented

{From tasks/index.md — full list of completed tasks.}

{Progress line from index.md, e.g. "Progress: 8 / 8 tasks complete"}

{Each task line: `- ✅ TASK-{layer}-{seq} — {task name} · {effort}`}

---

## Test Coverage

**Method:** {Standard / TDD — Red → Green → Refactor — from plan.md}
**Layers covered:** {list task layers and what they tested}

How to verify:
```bash
bun test
bun tsc --noEmit
```

---

## How to Test This Feature

{Step-by-step walkthrough derived from prd.md `<ux_flow><primary_flow>` — specific enough for a reviewer to follow without reading the PRD.}

1. {Actor} → {action} → {expected result}
2. {Actor} → {action} → {expected result}
3. ...

**Edge cases to check:**
- {from prd.md `<ux_flow><error_flows>` or NFRs}

---

## Success Metrics

{From prd.md `<overview><success_metrics>` — the measurable outcomes this feature enables.}

---

## PRD Documents

| Document | Path |
|---|---|
| Discovery | `{prd-folder}/discovery.md` |
| PRD | `{prd-folder}/prd.md` |
| Spec | `{prd-folder}/spec.md` |
| Plan | `{prd-folder}/plan.md` |
| Tasks | `{prd-folder}/tasks/index.md` |

---

## Checklist

- [ ] `bun tsc --noEmit` — zero errors
- [ ] `bun test` — all tests passing
- [ ] All {N} tasks marked complete in `tasks/index.md`
- [ ] No `any` types, no `@ts-ignore`, no TODO comments
- [ ] All inputs validated with Zod
- [ ] Auth checked before logic on all protected routes
- [ ] No hardcoded design tokens (UI changes only)
- [ ] Mobile-first responsive (UI changes only)
- [ ] Breaking changes documented above (or confirmed none)
- [ ] PRD documents linked above

🤖 Generated with [Claude Code](https://claude.com/claude-code) via /prd-pr
```

---

## Phase 6 — Create the Pull Request

```bash
gh pr create \
  --base {TARGET} \
  --head feat/{feature-slug}-impl \
  --title "{PR title from Phase 5}" \
  --body "$(cat <<'EOF'
{full PR description from Phase 5}
EOF
)"
```

After creation, output the PR URL and a brief summary:

```
✅ Pull request created

URL:    {pr url}
Title:  {pr title}
Base:   {TARGET} ← feat/{feature-slug}-impl
Tasks:  {N}/{N} complete

Next steps:
- Assign reviewers if needed:  gh pr edit {number} --add-reviewer {username}
- Request review:              gh pr ready {number}
- Monitor CI:                  gh pr checks {number} --watch
```

---

**Critical rules:**
- Never open a PR with unchecked tasks in `tasks/index.md`
- Never open a PR with TypeScript errors or failing tests
- Never push directly to the target branch — the PR is the merge gate
- PR title must be ≤ 72 characters and use imperative mood
- "Breaking Changes" section must be explicit — "None" is a valid and required answer
- All PRD document paths must be relative so they resolve in the repo
- Do not skip Phase 1 checks even if the user says "just open the PR"
