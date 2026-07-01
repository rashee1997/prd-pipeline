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

**Greenfield:** `/prd-discover "your app idea" --greenfield`

**Output:** `docs/prd/{feature-slug}-{date}/discovery.md`

**What it does:**
- **Input gate** — stops immediately with a recovery message if no feature description is provided
- Assesses whether the feature should be decomposed
- **Complexity routing** — classifies task as trivial/simple/complex; skips deep research for ≤3-file changes (60-80% token savings)
- Researches the existing codebase before asking questions
- Detects new feature vs enhancement
- Maps blast radius across API, DB, auth, UI, workflow, integrations, tests, and rollback
- Asks one precise grounded question at a time
- Question count adapts to complexity: 3 max (trivial), 7 max (simple), 12 max (complex)
- Writes `discovery.md` with complexity routing evidence

**`--greenfield` mode:**
- Skips live codebase scan (nothing exists yet)
- Researches external patterns and similar projects via GitHub/web instead
- Asks tech-stack, architecture, and ADR (Architecture Decision Record) questions instead of blast-radius questions
- Outputs `discovery.md` with `mode: greenfield` marker and `adr_stubs` section

## 2. `/prd-write`

Turns discovery into a PRD and technical specification.

**Input:** `/prd-write docs/prd/{feature}/discovery.md`

**Delta mode:** `/prd-write docs/prd/{feature}/discovery.md --delta`

**Outputs:** `prd.md`, `spec.md`

**What it does:**
- **Input gate** — stops if `discovery.md` is missing or empty
- Reads `discovery.md`
- Performs deeper codebase research
- Verifies library APIs and implementation patterns
- **Conditional blast-radius** — only researches dimensions with evidence of impact (~40% smaller specs)
- **CoVe (Chain-of-Verification) checkpoint** — before writing final files, extracts every cited file path/symbol/route from the draft and re-verifies each via MCP; unconfirmed names become `[UNVERIFIED]` rather than silently passing through (~77% fewer hallucinated names)
- **EARS acceptance criteria** — uses machine-parseable WHEN/WHILE/IF/WHERE + SHALL notation
- **Content-hashed spec blocks** — SHA-256 anchors in drift_anchors section for CI drift detection
- Writes a testable PRD
- Writes a buildable technical spec
- Captures compatibility requirements for enhancements

**`--delta` mode** (brownfield only):
- `spec.md` outputs three tables only: ADDED / MODIFIED / REMOVED with `file:symbol:change_type` columns
- Unchanged sections are omitted — 60-80% smaller specs for small enhancements
- `prd.md` remains full-format regardless

## 3. `/prd-plan`

Turns the PRD and spec into a slim dependency graph.

**Input:** `/prd-plan docs/prd/{feature}/prd.md docs/prd/{feature}/spec.md`

**Optional:** `--tdd` for TDD mode (RED → IMPL → GREEN → REFACTOR)

**Output:** `plan.md` (high-level — prd-tasks fills in details)

**What it does:**
- Validates PRD/spec assumptions against the live codebase
- Maps files to create and modify
- **Decomposition guidance** — DGI* ≈ 0.85√S heuristic prevents over-decomposition (avoids 71% coordination overhead)
- Builds the shortest safe dependency graph
- Identifies critical path and parallel tracks
- Adds compatibility Layer 0 for enhancements
- **Orchestrator handoff** — ≤200 token summary for prd-tasks (70% less context carry-over)
- Supports TDD planning
- Keeps plan.md lean — no per-step build bullets, acceptance checks, or blast-radius coverage (those belong in tasks)

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
- **Context budget per task** — estimated token cap + complexity tag (minimal/normal/complex); tasks fail loudly instead of silently truncating if budget exceeded
- **File size enforcement** — splits tasks that would push any file past ~350 lines
- **Re-derives detail** — plan.md is high-level; prd-tasks fills per-step build bullets, blast-radius coverage, and acceptance checks from spec + live MCP research

## 4a. `/prd-validate`

Validates generated tasks before implementation.

**Input:** `/prd-validate docs/prd/{feature}/tasks/index.md`

**Greenfield:** `/prd-validate docs/prd/{feature}/tasks/index.md --greenfield`

**Output:** `tasks/validation.md`

**What it does:**
- **Input gate** — stops if tasks path does not exist
- Checks task context isolation
- Validates dependency graph correctness
- Checks file path accuracy
- Checks acceptance commands
- **EARS format check** — validates acceptance criteria use WHEN/WHILE/IF/WHERE + SHALL notation
- Checks compatibility gates
- Checks parallel safety
- Checks live codebase references
- Blocks implementation if generated tasks are not ready

**`--greenfield` mode:**
- Replaces live-code existence checks with spec-conformance checks (no codebase exists yet)
- Checks path conventions instead of path existence
- Checks cross-task interface consistency instead of blast-radius
- Reports `spec_conformance_score` (0-100%); pass threshold ≥ 80%

## 5. `/prd-implement`

Implements validated task files.

**Input:** `/prd-implement docs/prd/{feature}/tasks/TASK-0-01.md`

**Parallel layer mode:** `/prd-implement --parallel-layer 0`

**What it does:**
- **Input gate** — stops if no task specified or validation has not passed
- Reads one task file
- Confirms dependencies are complete
- **Reads output notes from completed dependencies** — before researching, reads the `> **Output:**` note below each dependency row in `index.md` to catch deviations, renames, or side effects from prior tasks
- Confirms `/prd-validate` passed
- Finds the nearest existing codebase pattern via MCP
- **Re-anchoring checkpoint** — re-reads task goal at 50% of steps to prevent agent drift
- **File size guard** — splits files past 400 lines into separate responsibilities
- Writes the minimum code needed
- Runs acceptance checks
- Commits implementation
- Updates `tasks/index.md` — marks task `[x]` and appends a brief `> **Output:**` note (max 60 words) below the task row recording what was implemented and any deviations relevant to downstream tasks

## 6. `/prd-review`

Runs parallel specialist review.

**Input:** `/prd-review docs/prd/{feature}`

**Review dimensions:**
- Spec compliance (includes **faithfulness check** — verifies diff matches acceptance criteria, not just "passes tests")
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
├── _shared.md          ← shared base rules for all commands (not a command itself)
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
