# Principles

## Evidence Before Assumptions

Every technical detail should come from one of:
- Existing local codebase
- Verified library documentation
- Real implementation examples
- Explicit user decisions

If evidence is missing, the agent must research more or mark the item as unverified.

## Blast Radius Before Specification

Before writing requirements or implementation plans, the system asks:
- What API routes may change?
- What DB tables or models are affected?
- What auth/security boundaries are involved?
- What UI surfaces are affected?
- What workflows, jobs, events, or integrations may break?
- What tests protect existing behavior?
- What makes rollback safe or unsafe?

## Compatibility First for Enhancements

Enhancements are treated as risky by default. The system identifies:
- Frozen contracts
- Current callers
- Existing persisted data
- Current user-facing behavior
- Rollback constraints
- Cutover criteria

Layer 0 compatibility tasks must pass before implementation begins.

## One Task, One Commit

Each task file represents:
- One implementation unit
- One acceptance check
- One commit
- One clear rollback boundary

No task should require reading the PRD, spec, plan, or another task before starting.

## Complexity-Proportional Effort

Research depth and planning granularity scale with actual task complexity, not uniform maximum depth:
- **Trivial** (≤3 files, no schema/auth changes): minimal research, skip external patterns, max 3 questions
- **Simple** (clear scope, minor API change): standard research, skip external patterns, max 7 questions
- **Complex** (cross-cutting, new models/auth): full pipeline with external research

This prevents wasting 60-80% of token budget on trivial changes while ensuring complex work gets the depth it needs.

## Content-Hashed Spec Anchors

Spec sections carry SHA-256 content hashes for drift detection. When spec content changes after implementation, the hash mismatch flags the code as stale — enabling CI gates to catch spec/code divergence before it reaches production.

## Context Budget as Hard Constraint

Every task has an estimated token budget and a hard cap. If context would exceed the cap, the task splits rather than silently truncating — truncated instructions produce hallucinated code.

## Ponytail Discipline

The pipeline follows a "lazy senior engineer" mindset:

1. Does this need to exist?
2. Can existing code do it?
3. Can the standard library do it?
4. Can the platform do it?
5. Can an installed dependency do it?
6. Can it be one line?
7. Only then: write the minimum safe code.

The system is never lazy about:
- Security
- Input validation
- Auth
- Data loss
- Accessibility
- Compatibility contracts
- Explicit PRD requirements
- Tests for non-trivial behavior

## Semble-First Research

`mcp__semble` is mandatory for all code discovery — it is 100x more token-efficient than regex/grep. All research phases must call `mcp__semble__search` before any other MCP tool. This applies to every command in the pipeline.

## When To Use This

**Use for:**
- Brownfield features
- Complex product work
- Multi-file changes
- Security-sensitive features
- Workflow/state-machine changes
- DB/schema work
- Compatibility-sensitive enhancements
- Multi-agent implementation

**Avoid for:**
- Tiny one-line fixes
- Throwaway scripts
- Purely cosmetic copy changes
- Experiments where process cost exceeds risk

## Re-Anchoring Against Drift

At ~50% of task implementation steps, the agent re-reads the original task goal and acceptance criteria. This prevents the six identified mechanisms of agent drift (goal drift, context drift, role drift, tool-use drift, hallucination cascade, plan decay) that cause long-running agents to lose the plot.

## Faithfulness Over Test Coverage

A diff that passes all tests but behaves differently from the stated acceptance criteria is drift, not done. The spec-compliance reviewer checks faithfulness — does the code actually do what the criteria say, not just pass whatever test was written for it?

## Fail-Fast Input Gates

Every command validates its required inputs before doing any work. If `discovery.md` doesn't exist, `prd-write` stops immediately with a clear message: what is missing and which command to run first. Silent failures waste tokens and produce garbage output.

## Chain-of-Verification (CoVe)

Before `prd-write` writes its final files, it extracts every file path, symbol name, API route, and schema field cited in the draft and re-verifies each one against the live codebase via MCP. Items not found become `[UNVERIFIED: name]` — never silently substituted with a plausible guess. Research shows this reduces hallucinated symbol references by ~77% compared to single-pass generation.

## Greenfield Mode

When `--greenfield` is passed to `/prd-discover` and `/prd-validate`, the pipeline switches from live-code-verification mode to spec-conformance mode:

- Discovery asks tech-stack, architectural boundary, and ADR questions instead of blast-radius questions
- Research targets external patterns and similar projects instead of the local codebase
- Validation checks spec-conformance score (≥ 80% of tasks must link to a PRD requirement) instead of file/symbol existence

The rest of the pipeline (write → plan → tasks → implement → review → pr) runs identically.

## Delta Spec Mode

When `--delta` is passed to `/prd-write` for a small brownfield enhancement, `spec.md` outputs only three tables: ADDED, MODIFIED, REMOVED. Unchanged sections are omitted. `prd.md` remains full-format. For 1-3 file changes this reduces spec size by 60-80% without losing traceability.

## Shared Rules (`_shared.md`)

Base execution settings, the semble-first research rule, the UNVERIFIED tagging protocol, and the MCP fallback policy live in `_shared.md`. Every command references it. When a rule needs updating, one file changes — not nine.

## Task Output Log

When a task completes, `prd-implement` appends a brief `> **Output:**` note (max 60 words) directly below the task row in `tasks/index.md`. The next sequential task reads this note before researching — catching renames, unexpected side effects, or deviations from the plan before they propagate as stale assumptions.

## Design Goals

This system optimizes for:
- Fewer hallucinated implementation details
- Fewer broken existing contracts
- Better task isolation
- Better parallel execution
- **Proportional effort** — trivial changes skip expensive pipeline steps
- **Drift resistance** — re-anchoring and faithfulness checks prevent mid-task divergence
- **Greenfield support** — `--greenfield` flag covers new projects without a separate pipeline
- **Fail-fast** — every command gates on valid inputs before consuming tokens
- Easier review
- Safer PR creation
- Stronger traceability from requirement to code

It is intentionally more rigorous than a lightweight prompt-to-code workflow.
