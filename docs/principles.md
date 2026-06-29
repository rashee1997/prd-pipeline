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

## Design Goals

This system optimizes for:
- Fewer hallucinated implementation details
- Fewer broken existing contracts
- Better task isolation
- Better parallel execution
- Easier review
- Safer PR creation
- Stronger traceability from requirement to code

It is intentionally more rigorous than a lightweight prompt-to-code workflow.
