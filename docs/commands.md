# Commands

## Pipeline

```text
/prd-discover
  ↓
/prd-write
  ↓
/prd-plan
  ↓
/prd-tasks
  ↓
/prd-validate   ← Step 4a — quality gate before implementation
  ↓
/prd-implement
  ↓
/prd-review
  ↓
/prd-pr
```

`/prd-release` is an optional Step 8 for release management.

## 1. `/prd-discover`

Research-first Socratic discovery. Assesses scope, researches codebase + external patterns, performs blast-radius-aware Q&A, and saves `discovery.md`.

**Input:** `/prd-discover "external reviewer approval flow"`

**Output:** `docs/prd/{feature-slug}-{date}/discovery.md`

**What it does:**
- Assesses whether the feature should be decomposed
- Researches the existing codebase before asking questions
- Detects new feature vs enhancement
- Maps blast radius across API, DB, auth, UI, workflow, integrations, tests, and rollback
- Asks one precise grounded question at a time
- Writes `discovery.md`

## 2. `/prd-write`

Turns discovery into a PRD and technical specification.

**Input:** `/prd-write docs/prd/{feature}/discovery.md`

**Outputs:** `prd.md`, `spec.md`

**What it does:**
- Reads `discovery.md`
- Performs deeper codebase research
- Verifies library APIs and implementation patterns
- Performs blast-radius analysis
- Writes a testable PRD
- Writes a buildable technical spec
- Captures compatibility requirements for enhancements

## 3. `/prd-plan`

Turns the PRD and spec into a dependency graph.

**Input:** `/prd-plan docs/prd/{feature}/prd.md docs/prd/{feature}/spec.md`

**Optional:** `--tdd` for TDD mode (RED → IMPL → GREEN → REFACTOR)

**Output:** `plan.md`

**What it does:**
- Validates PRD/spec assumptions against the live codebase
- Maps files to create and modify
- Builds the shortest safe dependency graph
- Identifies critical path and parallel tracks
- Adds compatibility Layer 0 for enhancements
- Supports TDD planning

## 4. `/prd-tasks`

Turns the plan into one self-contained task file per implementation unit.

**Input:** `/prd-tasks docs/prd/{feature}/plan.md`

**Outputs:** `tasks/index.md`, `tasks/TASK-*.md`

**What it does:**
- Creates a lightweight TODO index
- Creates one task file per task
- Embeds task-specific code context
- Defines exact files to create or modify
- Defines exact acceptance checks
- Defines commit instructions
- Ensures every task can be executed by a fresh subagent

## 4a. `/prd-validate`

Validates generated tasks before implementation.

**Input:** `/prd-validate docs/prd/{feature}/tasks/index.md`

**Output:** `tasks/validation.md`

**What it does:**
- Checks task context isolation
- Validates dependency graph correctness
- Checks file path accuracy
- Checks acceptance commands
- Checks compatibility gates
- Checks parallel safety
- Checks live codebase references
- Blocks implementation if generated tasks are not ready

## 5. `/prd-implement`

Implements validated task files.

**Input:** `/prd-implement docs/prd/{feature}/tasks/TASK-0-01.md`

**Parallel layer mode:** `/prd-implement --parallel-layer 0`

**What it does:**
- Reads one task file
- Confirms dependencies are complete
- Confirms `/prd-validate` passed
- Finds the nearest existing codebase pattern via MCP
- Writes the minimum code needed
- Runs acceptance checks
- Commits implementation
- Updates `tasks/index.md`

## 6. `/prd-review`

Runs parallel specialist review.

**Input:** `/prd-review docs/prd/{feature}`

**Review dimensions:**
- Spec compliance
- Security
- Performance
- TypeScript strictness
- Code quality and architecture
- Test coverage

**Output format:**
```
SEVERITY · CATEGORY · path/to/file.ts:LINE
  ✗ problem
  ✓ fix
```

## 7. `/prd-pr`

Creates a pull request after all gates pass.

**Input:** `/prd-pr` or `/prd-pr develop`

**What it does:**
- Confirms current branch is a feature branch
- Confirms all tasks are complete
- Confirms task validation passed
- Confirms review is clean
- Runs final TypeScript and test checks
- Builds a self-contained PR description
- Pushes the feature branch
- Opens the PR with GitHub CLI

## 8. `/prd-release` (optional)

Release manager. Builds changelog/release notes, bumps version, commits, tags, and creates GitHub Release.

**Input:** `/prd-release docs/prd/{feature} --patch`

## Recommended Workflow

```bash
/prd-discover "external reviewer approval flow"

/prd-write docs/prd/external-reviewer-28-06-2026/discovery.md

/prd-plan docs/prd/external-reviewer-28-06-2026/prd.md docs/prd/external-reviewer-28-06-2026/spec.md

/prd-tasks docs/prd/external-reviewer-28-06-2026/plan.md

/prd-validate docs/prd/external-reviewer-28-06-2026/tasks/index.md

/prd-implement docs/prd/external-reviewer-28-06-2026/tasks/TASK-0-01.md

/prd-review docs/prd/external-reviewer-28-06-2026

/prd-pr
```

## Repository Layout

```text
.
├── README.md
├── LICENSE
├── assets/
│   └── logo.png
├── prd-discover.md
├── prd-write.md
├── prd-plan.md
├── prd-tasks.md
├── prd-validate.md
├── prd-implement.md
├── prd-review.md
├── prd-pr.md
├── prd-release.md
└── docs/
    ├── installation.md
    ├── commands.md
    ├── principles.md
    └── comparison.md
```

## Output Structure

A typical generated feature folder:

```text
docs/prd/external-reviewer-28-06-2026/
├── discovery.md
├── prd.md
├── spec.md
├── plan.md
└── tasks/
    ├── index.md
    ├── validation.md
    ├── TASK-0-01.md
    ├── TASK-0-02.md
    ├── TASK-1-01.md
    └── TASK-CUTOVER-01.md
```
