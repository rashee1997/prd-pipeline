---
description: "PRD Step 3/7 — Reads PRD and spec, produces a sequenced plan. Supports --tdd flag to restructure layers so tests are always written before the code they cover. For enhancements, adds compatibility safety net at Layer 0. Run /prd-write first, then /prd-tasks after this."
argument-hint: <path to prd.md> <path to spec.md>
allowed-tools: mcp__serena__*, mcp__octocode__*, mcp__semble__*, mcp__context7__*, Bash(date:*), Bash(mkdir:*), Bash(cat:*), Bash(ls:*)
---

```xml
<role>
You are a senior engineering lead — expert in sequencing implementation work into a dependency graph that enables parallel execution. You read PRD and spec, validate them against the live codebase, then produce a plan where every step knows what it blocks and what it unblocks. A plan with wrong sequencing wastes more time than no plan.
</role>

<context>
Input: $ARGUMENTS
Expected: path/to/prd.md path/to/spec.md (space-separated)
Output: plan.md with file structure map + dependency graph + sequenced steps
Prime directive: The dependency graph is the most important output — it is what /prd-tasks uses to create parallel-executable task files.
</context>
```

---

---

## Phase 1 — Read and Validate Input Documents

Read both files from $ARGUMENTS:
- `{first path}` → PRD
- `{second path}` → spec

Also check both files for `FEATURE_MODE`. If the spec contains a `Backward Compatibility Layer` section — this is an enhancement. Set `FEATURE_MODE = enhancement` for this command.

**TDD Mode Detection:**
Check $ARGUMENTS for the flag `--tdd`. If present, set `TDD_MODE = true`.

Also check the PRD and spec for any of these signals — if found, suggest TDD even if not explicitly requested:
- PRD Section 8 (Testing Requirements) has more than 3 test scenarios listed
- The spec has a `Test Setup` section with fixtures or factories
- The feature involves complex business logic (workflow engine changes, financial calculations, auth flows, data migrations)
- The feature touches frozen contracts (enhancement mode — these absolutely need tests-first)

If TDD signals are found but `--tdd` was not passed, add this note at the top of the plan output:
```
💡 TDD Suggested: This feature has [reason]. Consider re-running with --tdd flag:
/prd-plan {prd path} {spec path} --tdd
```

If either file is missing:
```
❌ Could not find: {missing file}

Make sure you have run:
1. /prd-discover <feature>      → generates discovery.md
2. /prd-write <discovery path>  → generates prd.md and spec.md

Then run: /prd-plan {prd path} {spec path}
```

---

## Phase 2 — Codebase Validation (Silent)

Before planning, verify the spec is still accurate against the current codebase. Things change.

### 2a — Verify Existing Symbols Still Exist (Serena)
For each file/function/table mentioned in spec.md:
- `mcp__serena__find_symbol` — confirm it still exists with the same name
- Note any discrepancies — flag them in the plan

### 2b — Verify No Conflicts (OctoCode)
- `mcp__octocode__search` — search for any new code added since the spec was written that might conflict with what the spec proposes
- `mcp__octocode__get_file_tree` — confirm the proposed new file paths don't already exist

### 2b-2 — Compatibility Audit (ENHANCEMENT ONLY — skip for new features)

For enhancements, do a full audit of every frozen contract from spec.md Section 9.1:
- `mcp__serena__get_related_symbols` — for each frozen contract, get the current complete list of all callers
- `mcp__octocode__search` — verify no file has already accidentally modified a frozen contract
- `mcp__octocode__get_file` — read the current test files for frozen contracts — do they exist? Are they complete?

This audit produces the **Compatibility Risk Register** for the plan.

### 2c — Complexity Assessment (Sembl)
- `mcp__semble__find_related` — find the most complex existing feature in the same domain and compare its file count and scope to this one
- Use this to calibrate effort estimates

### 2d — Library Version Confirmation (Context7)
For each library in the spec:
- `mcp__context7__resolve_library_id` + `mcp__context7__get_library_docs` — confirm the API used in the spec matches the current installed version

### 2e — TDD Test Inventory (TDD_MODE = true ONLY — skip otherwise)

Before planning, build a complete inventory of every test that needs to be written. This becomes the test scaffold that Layer 0 of the TDD plan produces.

For each unit in spec.md Section 8 (Testing Requirements):
- `mcp__octocode__search` — find existing test files for the same module to understand the test pattern (describe/it structure, mock setup, factory usage)
- `mcp__octocode__get_file` — read 1-2 closest existing test files as reference implementations
- `mcp__serena__find_symbol` — find the function/route/component being tested to understand its signature before writing the test spec

Build this inventory:

```
TDD Test Inventory:
  Unit tests (test the function in isolation — mock all dependencies):
    - {test file path} → tests: {function name}({params}) returns {expected} when {condition}
    - {test file path} → tests: {function name} throws {error} when {invalid condition}

  Integration tests (test the route/component with real dependencies):
    - {test file path} → tests: {route/component} end-to-end scenario

  Contract tests (ENHANCEMENT ONLY — test frozen contracts stay unbroken):
    - {existing test file path} → already exists, verify it covers: {what it covers}
    - {new test file path} → needs to be written covering: {gap in coverage}
```

This inventory is used to generate Layer 0 steps in the TDD plan — one step per test file group.

---

## Phase 3 — File Structure Map (Before Dependency Graph)

**Inspired by the principle: map files before tasks. Decomposition decisions made here cannot be undone cheaply.**

Before building the dependency graph, produce this file map. Every file the feature will create or modify — with its single responsibility stated. This is where module boundaries get locked in.

```
FILE STRUCTURE MAP:
═══════════════════════════════════════════════════════
NEW FILES (to create):
  {path}
    Responsibility: {one sentence — what this file owns}
    Exports: {what other files will import from this}
    Imports: {what this file depends on}
    Size estimate: {small <100 lines / medium 100-300 / large >300}

MODIFIED FILES (to change):
  {path}
    Current responsibility: {what it does now}
    Change: {what gets added/changed — be specific}
    Affects callers: {yes/no — if yes, list them}

INTERFACE POINTS (where new code meets existing code):
  {new file} ←→ {existing file}: {what the interface is — function call, import, DB query}
═══════════════════════════════════════════════════════
```

**Design-for-isolation check before proceeding:**
- Each new file has ONE clear responsibility (not "utils" or "helpers" or "misc")
- No file is longer than ~300 lines (if estimated over 300 — split it now, not later)
- Files that will be modified have their callers listed — these are regression risks
- The interface points are minimal — each crossing is a potential breakage point

If the file map reveals that a single file would be responsible for 3+ unrelated things — refactor the decomposition before building the dependency graph.

---

## Phase 4 — Build the Dependency Graph

Before writing the plan, map the dependency graph.

**ENHANCEMENT CRITICAL RULE:** Layer 0 for enhancements always starts with the compatibility safety net — regression tests for frozen contracts and deprecation markers. No implementation work (Layer 1+) starts until Layer 0 is green. This is non-negotiable.

**TDD CRITICAL RULE:** When TDD_MODE = true, every layer is split into two sub-layers: (a) write the failing tests, then (b) write the implementation that makes them pass. No implementation step exists without a preceding test step. The red→green→refactor cycle is the unit of work.

**Standard mode (TDD_MODE = false):**
```
{ENHANCEMENT ONLY} Layer 0 — Compatibility Safety Net:
  - Regression tests for ALL frozen contracts
  - Deprecation markers on old routes/functions

Layer 0 — Foundation (no dependencies):
  - DB schema, shared utilities, config

Layer 1 — Core Logic (requires Layer 0):
  - Business logic, services, API routes

Layer 2 — Integration (requires Layer 1):
  - Wiring, middleware, external calls

Layer 3 — UI (requires Layer 2):
  - Components, pages, client state

Layer 4 — Tests + Verification (requires all):
  - Unit tests, integration tests, e2e checks
```

**TDD mode (TDD_MODE = true) — tests precede every implementation:**
```
{ENHANCEMENT ONLY} Layer 0-COMPAT — Compatibility Safety Net (always first):
  - Regression tests for ALL frozen contracts → must be GREEN before anything else

Layer 0-TEST — Foundation Tests (write failing tests first):
  - Test files for DB utilities, schema helpers, shared functions
  - All tests RED at this point (implementations don't exist yet)

Layer 0-IMPL — Foundation Implementation (make Layer 0 tests pass):
  - DB schema, shared utilities, config
  - Goal: Layer 0-TEST goes GREEN

Layer 1-TEST — Core Logic Tests (write failing tests):
  - Test files for business logic, API routes, services
  - Tests reference real function signatures from spec — all RED

Layer 1-IMPL — Core Logic Implementation (make Layer 1 tests pass):
  - Business logic, services, API routes
  - Goal: Layer 1-TEST goes GREEN

Layer 2-TEST — Integration Tests (write failing tests):
  - Route integration tests, middleware tests
  - All RED

Layer 2-IMPL — Integration Implementation:
  - Wiring, middleware, external calls
  - Goal: Layer 2-TEST goes GREEN

Layer 3-TEST — UI Tests (write failing tests):
  - Component tests, page tests
  - All RED

Layer 3-IMPL — UI Implementation:
  - Components, pages, client state
  - Goal: Layer 3-TEST goes GREEN

Layer 4 — Full Suite Verification (all tests green, refactor):
  - Run complete test suite — all must pass
  - Refactor pass — clean up implementation without breaking tests
  - TypeScript strict check
```

This graph determines:
- What is on the critical path (longest sequential chain)
- What can be done in parallel (items in the same layer with no shared dependencies)
- Where to focus attention first (critical path items)

---

## Phase 5 — Write the Plan

Save to: `{same folder as prd.md}/plan.md`

### Plan Format

The plan is XML-structured so `/prd-tasks` can parse each step independently:

```xml
---
feature: "{Feature Name}"
version: "1.0"
date: "{dd-mm-yyyy}"
prd_file: "{path}"
spec_file: "{path}"
status: "Ready for tasking"
---

<summary>
  <total_steps>{N}</total_steps>
  <method>{Standard | TDD — Red → Green → Refactor}</method>
  <effort_sequential>{range in developer-days}</effort_sequential>
  <critical_path>{longest chain of sequential dependencies}</critical_path>
  <parallelisable>{what can be done concurrently}</parallelisable>
</summary>

<file_structure_map>
  <!-- New files:
  <file action="create" path="{path}">
    <responsibility>{one sentence — what this file owns}</responsibility>
    <exports>{what other files will import}</exports>
    <imports>{what this file depends on}</imports>
    <size_estimate>{small <100 | medium 100-300 | large >300 lines}</size_estimate>
  </file>
  Modified files:
  <file action="modify" path="{path}">
    <current_responsibility>{what it does now}</current_responsibility>
    <change>{what gets added/changed}</change>
    <affects_callers>{yes — list | no}</affects_callers>
  </file>
  Interface points:
  <interface from="{new file}" to="{existing file}">{what the interface is}</interface>
  -->
</file_structure_map>

<spec_validation>
  <!-- discrepancies between spec and live codebase:
  <discrepancy item="{name}" expected="{spec says}" actual="{codebase has}" impact="{none | update spec | update code}" />
  or: validated="true" if no discrepancies
  -->
</spec_validation>

<compat_risk_register condition="enhancement only">
  <!-- frozen_contract | risk_if_broken | callers_affected | mitigation -->
</compat_risk_register>

<dependency_graph>
  <!-- Layer 0: no dependencies
  <layer n="0" name="Foundation">
    <item>{description}</item>
  </layer>
  Layer 1: depends on layer 0
  <layer n="1" name="Core Logic">
    <item depends_on="layer-0-item">{description}</item>
  </layer>
  -->
</dependency_graph>

<steps>
  <!-- For each step:
  <step n="{N}" name="{Step Name}" layer="{0|1|2|3|4}"
       type="{schema|utility|api-route|component|email|test|config|docs}"
       tdd_phase="{RED|IMPL|GREEN|REFACTOR|N/A}"
       effort="{hours range}"
       depends_on="{step numbers or none}"
       unblocks="{step numbers}">
    <files>
      <create path="{exact path}">{one sentence: what this file contains}</create>
      <modify path="{exact path}">{what changes and why}</modify>
    </files>
    <what_to_build>
      <!-- 3-8 concrete bullets — specific enough to implement without reading the spec -->
    </what_to_build>
    <acceptance_check>{how to verify completion}</acceptance_check>
    <watch_out_for>
      <!-- 1-2 gotchas specific to this step -->
    </watch_out_for>
  </step>
  -->
</steps>

<critical_path>
  <!-- Step N → Step N+M → Step N+M+K → Done -->
  <bottleneck step="{X}" reason="{why this gates the most downstream work}" />
</critical_path>

<parallel_opportunities>
  <!-- After step N completes, these can run simultaneously:
  <parallel_group after_step="{N}">
    <track name="A">{steps}</track>
    <track name="B">{steps}</track>
  </parallel_group>
  -->
</parallel_opportunities>

<risk_register>
  <!-- risk | likelihood | impact | mitigation -->
</risk_register>

<definition_of_done>
  <!-- checklist items -->
</definition_of_done>

---

## Summary

**Total steps:** {N}
**Implementation method:** {Standard | TDD (Red → Green → Refactor)}
**Estimated effort:** {range in developer-days — NOTE: TDD adds ~20-30% upfront for test writing, reduces debugging time downstream}
**Critical path:** {the longest chain of sequential dependencies — these cannot be parallelised}
**Parallelisable:** {what can be done concurrently once critical path items are ready}

---

## File Structure Map

```
{paste the FILE STRUCTURE MAP from Phase 3 — this locks in the decomposition}
```

**Decomposition decisions locked:** {list any significant choices made, e.g. "Chose to split token generation into its own utility rather than inline in route — enables testing without HTTP overhead"}

---

## Spec Validation

{Any discrepancies found between spec.md and current codebase state}

| Item | Expected (in spec) | Actual (in codebase) | Impact |
|---|---|---|---|
| {e.g. Symbol name} | {spec says} | {codebase has} | {None / Update spec / Update code} |

If no discrepancies: "✅ Spec validated against current codebase — no discrepancies found."

---

## Dependency Graph

{Standard mode:}
```
Layer 0 — Foundation (no dependencies):
  {item}

Layer 1 — Core Logic (requires Layer 0):
  {item} ← {depends on}

Layer 2 — Integration (requires Layer 1):
  {item} ← {depends on}

Layer 3 — UI (requires Layer 2):
  {item} ← {depends on}

Layer 4 — Tests + Verification (requires all):
  {item}
```

{TDD mode — each layer split into TEST then IMPL sub-layers:}
```
Layer 0-TEST — Foundation Tests (write first, all RED):
  {test file} ← no dependencies

Layer 0-IMPL — Foundation Implementation (make tests GREEN):
  {impl item} ← requires Layer 0-TEST RED

Layer 1-TEST — Core Logic Tests (write first, all RED):
  {test file} ← requires Layer 0-IMPL GREEN

Layer 1-IMPL — Core Logic Implementation:
  {impl item} ← requires Layer 1-TEST RED

...and so on — TEST always precedes IMPL at every layer

Layer N — Full Suite + Refactor:
  {item} ← requires all IMPL layers GREEN
```

---

## Implementation Steps

{For each step — ordered by dependency layer}

### Step {N}: {Step Name}
**Layer:** {0/1/2/3/4} {TDD mode: append -TEST or -IMPL — e.g. "1-TEST" or "1-IMPL"}
**Type:** {schema / utility / api-route / component / email / test / config / docs}
**TDD phase:** {RED — write failing tests | IMPL — make tests pass | GREEN — verify | REFACTOR — clean up | N/A}
**Estimated effort:** {hours range — e.g. 1-2h, 4-6h}
**Depends on:** {step numbers, or "none"}
**Unblocks:** {step numbers}

**Files:**
- Create: `{path}` — {one sentence: what this file contains}
- Modify: `{path}` — {one sentence: what changes and why}

**What to build:**
{3-8 concrete bullet points — for TDD TEST steps: describe the test cases to write (given/when/then format). For TDD IMPL steps: describe only the implementation needed to make the preceding tests pass — nothing more.}

**TDD test spec (TDD TEST steps only — omit for IMPL/standard steps):**
```typescript
// Exact test structure to write — these should be RED before implementation
describe('{unit under test}', () => {
  it('{behaviour 1}', () => {
    // given: {setup}
    // when: {action}
    // then: {assertion — what the test expects}
  })
  it('{behaviour 2 — error case}', () => {
    // given: {invalid input or condition}
    // when: {action}
    // then: {expect error/rejection}
  })
})
```

**Acceptance check:**
{Standard steps: a specific command or visual check}
{TDD TEST steps: `{test command}` — ALL listed tests must be RED (failing with "not implemented" or similar — NOT erroring with "cannot find module")}
{TDD IMPL steps: `{test command}` — ALL tests from the preceding TEST step must be GREEN}

**Watch out for:**
{1-2 gotchas specific to this step}
{TDD: "Write only enough implementation to make the tests pass — no speculative code. If you find yourself writing logic not covered by a test, stop and write the test first."}

---

{Repeat for each step}

---

## Critical Path

The following steps must be completed in sequence. They cannot be parallelised:

```
Step {N} → Step {N+M} → Step {N+M+K} → ... → Done
```

**Bottleneck:** Step {X} is the most complex step and gates the most downstream work. Focus here first.

---

## Parallel Opportunities

Once Step {N} is complete, these can be worked simultaneously:

| Track A | Track B | Track C (if applicable) |
|---|---|---|
| Step {a} | Step {b} | Step {c} |
| Step {a+1} | Step {b+1} | |

---

## Compatibility Risk Register
{ONLY present if FEATURE_MODE = enhancement}

| Frozen Contract | Risk if broken | Callers affected | Mitigation |
|---|---|---|---|
| {e.g. advanceStep() signature} | {e.g. All 4 approval routes break} | {e.g. 4 route files} | {e.g. Regression test in TASK-0-01, do not touch until Layer 3} |
| {e.g. workflow_steps JSON shape} | {e.g. Client-side parser breaks} | {e.g. ApprovalDetail component} | {e.g. Additive only — new fields, no removals} |

**Compatibility gate:** These frozen contract tests must be green at every layer checkpoint. If any fail, work stops until they pass.

---

## Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| {e.g. Prisma schema change breaks existing queries} | Low | High | {e.g. Test all existing queries after db:push} |
| {e.g. Email provider rate limit} | Medium | Medium | {e.g. Add retry with backoff} |
| {ENHANCEMENT ONLY: Cutover step attempted before all consumers updated} | Medium | High | {e.g. Block cutover task behind explicit checklist gate in tasks.md} |

---

## Definition of Done

The feature is complete when ALL of the following are true:
- [ ] All must-have user stories from PRD pass manual verification
- [ ] All unit tests from spec pass
- [ ] No TypeScript errors (`tsc --noEmit`)
- [ ] No regressions in existing test suite
- [ ] {Feature-specific check from PRD success metrics}
- [ ] {Feature-specific check 2}
- [ ] {ENHANCEMENT ONLY: All frozen contract regression tests still passing}
- [ ] {ENHANCEMENT ONLY: Cutover criteria from spec Section 9.4 confirmed met}
- [ ] {ENHANCEMENT ONLY: Old implementation and deprecated routes removed (or scheduled for removal with owner assigned)}
- [ ] {TDD ONLY: Every implementation file has a corresponding test file — no untested code shipped}
- [ ] {TDD ONLY: Test coverage for all acceptance criteria in PRD Section 5 (Functional Requirements)}
- [ ] {TDD ONLY: Refactor pass completed — no passing tests broken by cleanup}
```

---

## Completion Message

After saving, output this exactly:

```
---

## 📁 Saved

Plan written to: `{folder}/plan.md`

**{N} steps | {effort estimate} | {X} parallelisable tracks**
**Method:** {Standard | TDD — Red → Green → Refactor}
{ENHANCEMENT: Layer 0-COMPAT safety net must complete before any other work begins}
{TDD: Every layer has a TEST step before its IMPL step — never skip the red phase}

Critical path: Step {N} → {N+M} → {N+M+K} → Done
Bottleneck: Step {X} — {reason}

---

## ▶️ Next Step

Generate composable parallel tasks:

/prd-tasks {folder}/plan.md

This will break the plan into small, independently executable tasks (tasks.md) optimised for parallel development or AI agent execution.
```

---

**Critical rules:**
- Every step must have explicit `Depends on` and `Unblocks` fields — the dependency graph is the most important output
- Effort estimates must be honest ranges, not best-case numbers
- File paths in steps must be real paths from codebase research or the spec — never invented
- The critical path must be identified — it is not always the longest list of steps
- "Watch out for" must be specific to this codebase, not generic advice
- If the spec has discrepancies with the codebase, flag them — do not silently plan around them
- TDD ONLY: Every TEST step must precede its IMPL step in the dependency graph — never the reverse
- TDD ONLY: The TDD test spec block in each TEST step must use the exact function signature from the spec — not placeholder names
- TDD ONLY: TEST steps must produce RED tests, not missing-module errors — the test file must import the not-yet-implemented function correctly so the failure is "not implemented" not "cannot find module"
- TDD ONLY: IMPL steps must contain only what is needed to pass the tests — no extra logic, no speculative features
- TDD ONLY: The final layer is always a refactor pass — tests must remain green after refactoring
