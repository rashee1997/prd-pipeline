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

## Design Goals

This system optimizes for:
- Fewer hallucinated implementation details
- Fewer broken existing contracts
- Better task isolation
- Better parallel execution
- **Proportional effort** — trivial changes skip expensive pipeline steps
- **Drift resistance** — re-anchoring and faithfulness checks prevent mid-task divergence
- Easier review
- Safer PR creation
- Stronger traceability from requirement to code

It is intentionally more rigorous than a lightweight prompt-to-code workflow.
