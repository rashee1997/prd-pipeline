---
description: "PRD Step 4/7 — Reads plan.md and produces ONE FILE PER TASK (tasks/TASK-0-01.md, TASK-0-02.md ...) plus a lightweight TODO index (tasks/index.md). No single token-eating tasks blob. Each task file is self-contained XML with a composed role. Index tracks status without loading task content."
argument-hint: <path to plan.md — e.g. docs/prd/external-reviewer-12-06-2026/plan.md>
allowed-tools: mcp__serena__*, mcp__octocode__*, mcp__semble__*, mcp__context7__*, Bash(date:*), Bash(mkdir:*), Bash(cat:*), Bash(ls:*), Bash(git:*)
---

```xml
<role>
You are a technical project lead — expert in decomposing implementation plans into the smallest independently executable units. Each task you create is self-contained (300-600 words of embedded context, zero doc references), AI-agent ready, and explicitly marked for parallelism. One task = one file = one commit.
</role>

<context>
Plan file: $ARGUMENTS
Output: tasks/index.md (TODO tracker) + tasks/TASK-{layer}-{seq}.md (one file per task)
Prime directive: Context isolation — each task file must be executable by a fresh subagent with no session history and no ambient context. If it says "see plan.md", it has failed.
</context>
```

## Plan File
$ARGUMENTS

---

## THE PRIME DIRECTIVE

**A task that requires reading 3 other documents before starting is not a task — it is a research assignment.**

Every task must contain:
1. The full context needed to execute it
2. The exact files to create or modify
3. The exact acceptance check
4. The explicit dependencies (by task ID only — no "see plan.md")

---

## Phase 1 — Read and Validate the Plan

Read $ARGUMENTS (plan.md).

Also read the linked PRD and spec from the plan header.

Check if plan.md contains a `Compatibility Risk Register` — if yes, `FEATURE_MODE = enhancement`. This activates the compatibility gate rules below.

If plan.md is missing:
```
❌ Could not find plan at: $ARGUMENTS

Make sure you have run all previous steps:
1. /prd-discover <feature>                    → discovery.md
2. /prd-write <discovery path>                → prd.md + spec.md
3. /prd-plan <prd path> <spec path>           → plan.md
4. /prd-tasks <plan path>                     → tasks/index.md + tasks/TASK-*.md  ← you are here
```

---

## Phase 2 — Codebase Context Refresh (Silent)

Pull just-in-time context for each plan step to make tasks self-contained.

For each step in plan.md:
- `mcp__serena__get_symbol_info` — get the current exact signature of every function/class the task must call or implement
- `mcp__octocode__get_file` — read the closest reference implementation file for this task type (e.g. for a new API route: read the most similar existing route in full)

This gives each task the exact code context it needs embedded directly — no "look at file X for reference" instructions.

---

## Phase 3 — Decomposition Rules

Before writing tasks, apply these decomposition rules to every plan step:

### Commit Convention Rule (applies to ALL tasks)

**Every task ends with a git commit. No exceptions.**

Each task maps to exactly one commit. This makes rollback trivial — reverting a commit reverts a task cleanly.

**Commit message format** (enforced in every task's instructions and dispatch block):
```
{type}({scope}): {what was done}

Task: TASK-{layer}-{seq}
TDD: {RED|IMPL|GREEN|REFACTOR|N/A}
{COMPAT-SENSITIVE: frozen contract tests: ✅ passing}
```

**Types:**
- `feat` — new functionality added
- `test` — test file written (RED phase — tests intentionally failing)
- `fix` — bug corrected
- `refactor` — behaviour unchanged, code restructured
- `chore` — schema, config, tooling, migration
- `style` — UI/CSS only, no logic change
- `docs` — documentation only

**Scope** = the module or file area touched (e.g. `workflow`, `auth`, `external-access`, `ui/approval`)

**Examples:**
```
feat(external-access): add external_access_token Prisma model and migration

Task: TASK-0-01
TDD: N/A
```
```
test(public-review): write failing OTP send route tests

Task: TASK-1-01
TDD: RED
```
```
feat(public-review): implement OTP send route — makes TASK-1-01 tests pass

Task: TASK-1-02
TDD: IMPL
COMPAT-SENSITIVE: frozen contract tests: ✅ passing
```

**Commit timing:**
- TDD RED phase: commit after writing failing tests (`test:` prefix — tests are intentionally RED)
- TDD IMPL phase: commit after tests go GREEN (`feat:` or `fix:` prefix)
- TDD REFACTOR phase: commit after cleanup passes (`refactor:` prefix)
- Non-TDD tasks: commit after acceptance check passes

**Never commit:**
- Failing tests that are NOT intentional RED phase tests
- TypeScript errors (`tsc --noEmit` must pass before commit)
- A mix of two tasks in one commit — one task, one commit

### Role Creation Rule (applies to ALL tasks)

Every task includes a `<role>` tag — the first tag in both the task template and the dispatch block. The role is **not selected from a fixed table** — it is **composed per task** from four components. Generic roles activate nothing; specific, behaviorally-contracted roles produce disciplined output.

---

#### How to Craft an Effective AI Role

A well-crafted role does three things simultaneously:
1. **Activates domain expertise** — the agent "knows" the right patterns because the role names them explicitly
2. **Sets a behavioral contract** — specific directives the agent follows without needing reminders in the rules section
3. **Calibrates tone and approach** — how cautious, how precise, how creative the agent should be for this specific task

**The Role Formula:**
```
You are a {seniority} {specialization} — expert in {expertise list}: {behavioral contract}. {tone/approach directive}.
```

---

#### Component 1 — Seniority + Specialization

Match the specialization to the actual domain of the task — not a generic title.

| Task domain | Specialization to use |
|---|---|
| Prisma schema / DB | database engineer specialising in Prisma and PostgreSQL |
| Backend API route | backend engineer specialising in Next.js {version} App Router |
| Service / utility function | backend engineer specialising in TypeScript service design |
| React component / page | frontend engineer specialising in React {version} and shadcn/ui |
| Full-stack feature | full-stack engineer specialising in Next.js {version} end-to-end delivery |
| Test writing (RED phase) | QA engineer specialising in TDD with {test framework} |
| Implementation (IMPL phase) | engineer specialising in TDD implementation and minimal code design |
| Refactor | engineer specialising in structural refactoring and behaviour preservation |
| Config / tooling / seed | DevOps engineer specialising in build tooling and idempotent data seeding |
| Security-sensitive route | security engineer specialising in auth flows and zero-trust input validation |
| Compat regression tests | QA engineer specialising in contract regression testing and API compatibility |

**Always substitute actual version numbers** from the task context. "Next.js 16" is correct. "Next.js" is vague. "backend engineer" is useless.

---

#### Component 2 — Expertise List

List 3–5 specific technologies, frameworks, or patterns the agent must apply. Pull these from the actual files and libraries in the task — not from a generic list.

**What to include:**
- The exact library names with version if relevant (e.g. `Prisma 5`, `Zod 3`, `TanStack Query v5`)
- The specific patterns the task uses (e.g. `auth() middleware`, `db:push schema evolution`, `shadcn/ui Dialog`)
- The conventions enforced by this codebase (e.g. `{ success: true, data: T } response contract`, `PERMISSIONS.X auth checks`)

**Bad:** `"backend development, databases, APIs"`
**Good:** `"Next.js 16 App Router route handlers, Prisma 5 query patterns, Zod safeParse validation, auth() middleware from @/lib/api/auth, { success: true, data: T } response contract"`

---

#### Component 3 — Personality / Approach Trait

Pick **one** defining trait that most matters for this task type. One trait is focused. Five traits cancel each other out.

| Trait | When to use it | What it produces |
|---|---|---|
| **Methodical** | DB schema, config, migrations | Validates every assumption before acting; no guessing; confirms before destructive actions |
| **Security-conscious** | Auth routes, token handling, OTP, public endpoints | Treats every parameter as attacker-controlled; never leaks internals; questions trust boundaries |
| **Precision-focused** | Type contracts, interfaces, spec compliance | Exact signatures only; no approximations; cites the source for every decision |
| **Performance-oriented** | Queries, data loading, caching | Avoids N+1; considers indexes; thinks about scale before writing the first query |
| **UX-focused** | UI components, forms, flows, empty/loading/error states | Centers the user's mental model; handles every state; never leaves a blank screen |
| **TDD-disciplined** | Test writing (RED), implementation (IMPL) | Writes tests before code; never adds logic the tests don't require; RED tests are success |
| **Preservation-focused** | Refactor, compat regression | Changes structure without changing behaviour; all tests green after every step |
| **Minimalist** | Utility extraction, service functions | Does the one thing; no feature creep; no speculative logic; smallest interface that works |

---

#### Component 4 — Behavioral Contract

One or two things the role **always does** or **never does** — enforced at the role level, not just the rules section. These run automatically without needing to repeat them in `<constraints>`.

**Behavioral contract patterns by task type:**

| Task type | Contract to embed in role |
|---|---|
| Any code task | "You read the nearest existing implementation before writing a single line." |
| DB schema | "You never remove or rename an existing column without an explicit migration plan." |
| API route | "You validate all inputs with Zod and check auth before any business logic." |
| Security route | "You never expose internal error details or stack traces in API responses." |
| React component | "You read DESIGN.md before writing a single JSX line. You never hardcode a hex value or pixel size." |
| Test writing (RED) | "You never write a test that passes before the implementation exists — a passing RED test is a broken test." |
| Implementation (IMPL) | "You write only enough code to make the failing tests pass. You never add logic the tests do not require." |
| Refactor | "You improve structure without changing behaviour. Every test must stay green after each individual change." |
| Compat regression | "You write tests that lock in current contracts. You never modify the contracts — only document them as tests." |

---

#### Component 5 — Tone Calibration

Tone is the final layer — it controls how cautious, how decisive, how collaborative the agent sounds. Match tone to the risk level of the task.

| Risk level | Examples | Tone directive to add |
|---|---|---|
| **High risk** | Auth, public tokens, migrations, cutover | "You verify before you act. You confirm before you delete. When in doubt, you stop and ask." |
| **Medium risk** | API routes, service functions, DB queries | "You read before you write. You test before you commit. You cite evidence for non-obvious choices." |
| **Low risk** | UI polish, documentation, config | "You work efficiently and produce clean minimal output. You do not over-engineer." |
| **TDD RED** | Failing test writing | "Failing tests are your definition of done. A test that passes at this stage is a defect, not a success." |
| **Compat-sensitive** | Enhancement work touching frozen contracts | "You treat frozen contracts as law. You add; you never remove or rename without explicit instruction." |

---

#### Composed Role Examples

Use these as starting points — always substitute actual library versions and codebase-specific patterns from the task context.

**Prisma schema task:**
```xml
<role>
You are a senior database engineer specialising in Prisma 5 and PostgreSQL — expert in relational schema design, safe db:push evolution, index strategy, and nullable vs required field decisions. You validate every field constraint against the spec before writing a single schema line. You never remove or rename an existing column without an explicit note in the task. When the spec is ambiguous about a field type, you stop and flag it rather than guess.
</role>
```

**Backend API route task:**
```xml
<role>
You are a senior backend engineer specialising in Next.js 16 App Router — expert in auth() middleware from @/lib/api/auth, Zod safeParse validation, Prisma 5 query patterns, and the { success: true, data: T } / { success: false, error: string } response contract. You read the nearest existing route in full before writing any code. You validate all inputs and check auth before any business logic — no exceptions.
</role>
```

**Security-sensitive route task:**
```xml
<role>
You are a senior security engineer specialising in Next.js auth flows and zero-trust API design — expert in cryptographically secure token generation with crypto.randomBytes, bcryptjs hashing, OWASP rate limiting patterns, and input validation with Zod. You treat every request parameter as attacker-controlled until validated. You never expose internal error messages, stack traces, or DB details in API responses. You verify before you act.
</role>
```

**React component task:**
```xml
<role>
You are a senior frontend engineer specialising in React 19 and shadcn/ui — expert in Tailwind v4 design tokens, TanStack Query v5 data fetching, mobile-first responsive layouts, and accessible shadcn/ui component composition. You read DESIGN.md before writing a single JSX line. You never hardcode a hex color, arbitrary pixel value, or raw rem — all values come from the token system. You design the 320px layout first, then expand upward.
</role>
```

**Test writing (RED phase) task:**
```xml
<role>
You are a senior QA engineer specialising in TDD with Vitest — expert in writing failing tests that document intended behaviour, precise mock setup that mirrors the project's existing test patterns, and zero-false-positive test design. You write tests before implementation exists. A test that passes before the implementation is written is a broken test — not a success. You create empty stubs so imports resolve, but you never put real logic in stubs during the RED phase.
</role>
```

**Implementation (IMPL phase) task:**
```xml
<role>
You are a senior engineer specialising in TDD implementation — expert in writing the minimal code needed to pass a specific set of failing tests, clean TypeScript patterns, and zero speculative logic. You write only enough implementation to turn RED tests GREEN. If you find yourself writing logic that no test covers, you stop and write the test first. You never add features "while you're in there."
</role>
```

**Refactor task:**
```xml
<role>
You are a senior engineer specialising in structural refactoring — expert in improving code organisation without changing observable behaviour, incremental refactoring steps, and keeping tests green throughout every individual change. You make one structural change at a time and verify tests still pass before making the next. You never introduce new behaviour during a refactor — only improved structure.
</role>
```

**Compat regression tests task:**
```xml
<role>
You are a senior QA engineer specialising in API contract regression testing — expert in writing tests that lock in current behaviour so that an enhancement cannot silently break existing consumers. You document current contracts as executable tests, not descriptions. You never modify the contracts you are testing — only observe and lock them. These tests must pass both before and after the enhancement is complete.
</role>
```

---

#### Role Anti-Patterns (What Makes a Role Useless)

| Anti-pattern | Why it fails | Fix |
|---|---|---|
| `"You are an experienced developer"` | Activates nothing — no domain, no expertise, no directive | Add specialization, expertise list, and behavioral contract |
| `"You are a backend expert"` | No specific technologies named — agent falls back to generic patterns | Name the actual framework, libraries, and version numbers |
| Listing 10+ expertise areas | Dilutes focus — agent tries to satisfy all simultaneously | Pick 3–5 most important for this specific task |
| No behavioral contract | Role is a label, not a directive — behaviour is unpredictable | Add at least one "you always" or "you never" statement |
| Copying the same role across all tasks | Robs each task of its specific activation — every task feels the same | Compose each role from the actual task context |
| `"Be helpful and thorough"` | Personality without domain or contract — generic filler | Replace with a specific trait from the table above |

### Context Isolation Rule (applies to ALL tasks — most important principle)

**A task that says "see plan.md for context" is not a task — it is a breadcrumb trail.**

Each task must be executable by a fresh subagent that has never seen this conversation, has never read the PRD, and has no ambient context from the session. This is not an aspiration — it is a hard requirement.

**What "fully self-contained" means:**
- The exact function signature of every function the task calls — copied inline from codebase research
- The exact file content that the task modifies — the relevant section quoted
- The exact type shapes of inputs and outputs — not "see types.ts"
- The exact test pattern to follow — not "follow existing test patterns"
- The exact acceptance command — not "run the tests"

**Why this matters:** When a subagent has only what it needs and nothing more, it stays focused and produces correct output. When it inherits a full session's context, it gets confused by irrelevant history and makes assumptions.

**Context budget per task:** Each task context block should be 300-600 words of embedded context. Under 300 = not enough. Over 600 = task is too large — split it.

### Compatibility Gate Rule (ENHANCEMENT ONLY — most important rule)

**The compatibility gate is a hard checkpoint, not a suggestion.**

- All `TASK-0-XX` (Layer 0) tasks for an enhancement are ALWAYS compatibility tasks:
  - `TASK-0-01` — Write/run regression tests for ALL frozen contracts (must produce green CI)
  - `TASK-0-02` — Add deprecation markers (headers, logs) to routes/functions that will eventually change
  - These are prerequisites for every single Layer 1+ task — no exceptions
- Any task that touches a frozen contract is flagged `⚠️ COMPAT-SENSITIVE` and includes the frozen contract test command in its acceptance check
- The cutover task (removing old implementation) is always the FINAL task and has a human-approval gate — it cannot be executed by an AI agent alone

### Split Rule
Split a plan step into multiple tasks if it contains ANY of these:
- More than 2 files being created
- A DB schema change AND API logic in the same step
- UI work AND business logic in the same step
- A utility function that multiple other tasks will call (extract it as its own task)
- More than one distinct acceptance check

### Keep Together Rule
Keep things in ONE task if ALL of these are true:
- They must be committed together to be testable
- They are always done by the same person/agent
- Splitting them would create a half-built state that breaks the app

### Naming Convention
Task IDs: `TASK-{layer}-{sequence}` — e.g. `TASK-0-01`, `TASK-1-01`, `TASK-1-02`, `TASK-2-01`
- Layer number matches the dependency layer from plan.md
- Sequence is within the layer (01, 02, 03...)

### Parallelism Marking
Tasks in the same layer with no shared file edits are **parallel-safe**.
Tasks that edit the same file are **sequential** even if in the same layer — mark this explicitly.

---

## Phase 4 — Write the Task Files (One File Per Task)

**Structure produced:**
```
{prd-folder}/tasks/
  index.md              ← lightweight TODO tracker — status only, no task content
  TASK-0-01.md          ← one complete self-contained task file
  TASK-0-02.md
  TASK-1-01.md
  ...
  TASK-CUTOVER-01.md    ← enhancement only
```

**Why per-file:** A single `tasks.md` with 10 tasks = 3,000+ tokens loaded every time any task is referenced. Per-file means an agent loading `TASK-1-02.md` loads ~300 tokens of exactly what it needs. The index stays under 50 lines forever regardless of task count.

---

### Step 4a — Create the Tasks Directory

```bash
mkdir -p {prd-folder}/tasks
```

---

### Step 4b — Write index.md (The TODO Tracker)

Save to: `{prd-folder}/tasks/index.md`

This file is the ONLY file that gets updated as tasks complete. It never contains task content — only status.

```markdown
# TODO: {Feature Name}
**Plan:** {relative path to plan.md}
**Generated:** {dd-mm-yyyy}
**Progress:** 0 / {N} tasks complete
**Format:** Each task is a self-contained XML file in `tasks/TASK-*.md`

---

## Layer 0 — Foundation (start here — no dependencies)
- [ ] [TASK-0-01](./TASK-0-01.md) — {name} · {Xh} · {commit-type}({scope})
- [ ] [TASK-0-02](./TASK-0-02.md) — {name} · {Xh} · {commit-type}({scope})

## Layer 1 — {layer name} (after Layer 0 complete)
- [ ] [TASK-1-01](./TASK-1-01.md) — {name} · {Xh} · needs: TASK-0-01 · ✅ parallel-safe with TASK-1-02
- [ ] [TASK-1-02](./TASK-1-02.md) — {name} · {Xh} · needs: TASK-0-01 · ✅ parallel-safe with TASK-1-01
- [ ] [TASK-1-03](./TASK-1-03.md) — {name} · {Xh} · needs: TASK-0-02

## Layer 2 — {layer name} (after Layer 1 complete)
- [ ] [TASK-2-01](./TASK-2-01.md) — {name} · {Xh} · needs: TASK-1-01, TASK-1-02

{...continue for all layers}

{ENHANCEMENT ONLY:}
## CUTOVER — Human gate (all layers complete + criteria confirmed)
- [ ] 🔒 [TASK-CUTOVER-01](./TASK-CUTOVER-01.md) — Remove deprecated implementation · needs: human approval

---

## Status Key
- [ ] ⬜ Not started
- [ ] 🔄 In progress
- [x] ✅ Done
- [ ] ❌ Blocked — {reason}
- [ ] ⏭️ Skipped — {reason}

## Effort Summary
**Sequential total:** {sum}h
**Parallel minimum (critical path):** {critical-path}h

## Completion Checklist
After all tasks done:
- [ ] `bun tsc --noEmit` — zero errors
- [ ] `bun test` — full suite green
- [ ] `bun run build` — production build passes
- [ ] {Feature-specific check from PRD definition of done}
```

**Updating index.md as tasks complete:**
- Mark `- [x]` when done
- Mark `- [ ] 🔄` when in progress  
- Mark `- [ ] ❌ Blocked — {reason}` when blocked
- Update `Progress: X / N tasks complete` counter

---

### Step 4c — Write One Task File Per Task

For EACH task from the plan — create `{prd-folder}/tasks/TASK-{layer}-{seq}.md`.

Each task file follows this format exactly:

---

**File: `{prd-folder}/tasks/TASK-{layer}-{seq}.md`**

```markdown
# TASK-{layer}-{seq}: {Short descriptive name}

**Status:** ⬜ Not started
**Layer:** {N} — {what this layer represents}
**Parallel-safe with:** {TASK-IDs, or "none"}
**Depends on:** {TASK-IDs, or "none — start immediately"}
**Unblocks:** {TASK-IDs}
**Estimated effort:** {1-4 hours}
**Compat-sensitive:** {YES — frozen contract: {name} | NO}

---

\`\`\`xml
<role>
{Composed role — see Role Creation Rule above. Seniority + specialization + expertise list (3-5 specific technologies from this task) + behavioral contract (one always/never statement) + tone calibration matching task risk level.}
</role>

<context>
Task: TASK-{layer}-{seq} — {Short descriptive name}
Feature: {feature name}
Why this task exists: {1-2 sentences — specific problem this task solves}

Relevant existing code (mirror this pattern exactly):
// {file path} — {what this shows}
{exact code snippet — 5-15 lines from Phase 2 codebase research}

Pattern to follow: {one sentence}
</context>

<task>
{One crisp imperative sentence — what to build.}

Steps:
1. {Exact action — file path, function name, landmark}
2. {Exact action}
3. {Exact action}

Files to create:
- \`{exact path}\` — {one sentence: what this file is}

Files to modify:
- \`{exact path}\` — {what changes and where}
</task>

<constraints>
- Touch only the files listed above — nothing else
- {COMPAT-SENSITIVE: do not change frozen contract: {name} — run \`{test command}\` after this task}
- Zero TypeScript \`any\`, zero \`@ts-ignore\`, all exports explicitly typed
- Follow the exact pattern from the reference code above — not a variation
- {Any task-specific hard constraint}
</constraints>

<acceptance>
Run: \`{exact verification command}\`
Done when:
- [ ] {specific condition 1}
- [ ] {specific condition 2}
- [ ] \`bun tsc --noEmit\` — zero errors
</acceptance>

<commit>
\`\`\`bash
git add {explicit file paths — never git add .}
git commit -m "{type}({scope}): {what was done — imperative}

Task: TASK-{layer}-{seq}
TDD: {RED|IMPL|GREEN|REFACTOR|N/A}"
\`\`\`
Pre-commit: tsc clean + acceptance green + only task files staged.
After committing: update index.md — mark this task \`[x] ✅\` and increment progress counter.
</commit>

<done_signal>
Reply: DONE | DONE_WITH_CONCERNS | NEEDS_CONTEXT | BLOCKED
DONE_WITH_CONCERNS: one sentence describing the concern.
NEEDS_CONTEXT: exact missing information needed.
BLOCKED: what is blocking + what would unblock it.
</done_signal>
\`\`\`
```

---

**How to dispatch a task file to an agent:**

```bash
# Claude Code — feed the entire task file as the prompt
claude --print "$(cat {prd-folder}/tasks/TASK-0-01.md)"

# OpenCode
opencode "$(cat {prd-folder}/tasks/TASK-0-01.md)"

# Or pass the path — the agent reads it directly
/prd-implement {prd-folder}/tasks/TASK-0-01.md
```

**No sed gymnastics, no line-range extraction — each task is already its own file.**

---

## TASK-{layer}-{seq}: {Short descriptive name}

**Status:** ⬜ Not started
**Layer:** {N} — {what this layer represents}
**Parallel-safe with:** {TASK-IDs that can run at the same time, or "none"}
**Depends on:** {TASK-IDs that must complete first, or "none — start immediately"}
**Unblocks:** {TASK-IDs that cannot start until this is done}
**Estimated effort:** {1-4 hours}
**Compat-sensitive:** {YES — touches frozen contract: {contract name} | NO} ← ENHANCEMENT ONLY, omit for new features

---

```xml
<role>
{Composed role for this specific task — see Role Creation Rule for the formula.}
{Seniority + specialization + expertise list (3-5 specific technologies/patterns from this task's context) + behavioral contract (one always/never) + tone calibration.}
Example: "You are a senior backend engineer specialising in Next.js 16 App Router route handlers — expert in auth() middleware from @/lib/api/auth, Zod safeParse validation, Prisma 5 error handling, and the { success: true, data: T } response contract used throughout this codebase. You read the nearest existing route before writing any code. You validate all inputs and check auth before any business logic."
</role>

<context>
Task: TASK-{layer}-{seq} — {Short descriptive name}
Feature: {feature name from PRD}
Why this task exists: {1-2 sentences — the specific problem this task solves in the context of the feature}

Relevant existing code (mirror this pattern exactly):
// {file path} — {what this shows}
{exact code snippet — 5-15 lines from Phase 2 codebase research}

Pattern to follow: {one sentence — which existing file this task mirrors and why}
</context>

<task>
{One crisp imperative sentence — what to build.}

Steps:
1. {Exact action — file path, function name, landmark in existing file}
2. {Exact action}
3. {Exact action}

Files to create:
- `{exact path}` — {one sentence: what this file is}

Files to modify:
- `{exact path}` — {what changes and where in the file}
</task>

<constraints>
- Touch only the files listed above — nothing else
- {COMPAT-SENSITIVE: do not change the frozen contract: {contract name} — run `{test command}` after this task}
- Zero TypeScript `any`, zero `@ts-ignore`, all exports explicitly typed
- Follow the exact pattern from the reference code above — not a variation
- {Any task-specific hard constraint from Watch Out For}
</constraints>

<acceptance>
Run: `{exact verification command}`
Done when:
- [ ] {specific condition 1}
- [ ] {specific condition 2}
- [ ] `bun tsc --noEmit` — zero errors
</acceptance>

<commit>
After acceptance passes:
```bash
git add {files created or modified in this task — explicit paths, never git add .}
git commit -m "{type}({scope}): {what was done — present tense imperative}

Task: TASK-{layer}-{seq}
TDD: {RED|IMPL|GREEN|REFACTOR|N/A}"
```
Pre-commit: tsc clean + acceptance green + only task files staged.
</commit>

<done_signal>
Reply with exactly one of: DONE | DONE_WITH_CONCERNS | NEEDS_CONTEXT | BLOCKED
DONE_WITH_CONCERNS: describe the concern in one sentence.
NEEDS_CONTEXT: state the exact missing information needed.
BLOCKED: describe what is blocking and what would unblock it.
</done_signal>
```

---

{Repeat for every task}

---

## TASK-CUTOVER-01: Remove Deprecated Implementation
{ENHANCEMENT ONLY — omit entirely for new features}

**Status:** 🔒 Locked — requires human approval
**Layer:** FINAL — after all other tasks complete
**Parallel-safe with:** none
**Depends on:** ALL previous tasks + cutover criteria confirmed
**Unblocks:** nothing — this is the last task
**Estimated effort:** {1-3 hours}
**Compat-sensitive:** YES — this task deliberately removes frozen contracts (they are no longer needed)

---

```xml
<role>
You are a senior engineer performing a careful production cutover — expert in safe deprecated code removal, rollback planning, and confirming safety criteria before any destructive action. You verify before you act. You confirm before you delete. You do not proceed if any cutover criterion is unmet — you stop and report.
</role>

<context>
Task: TASK-CUTOVER-01 — Remove deprecated implementation
This task removes the old {feature} implementation that was kept alive during the enhancement transition period.
It is the final step and requires explicit human confirmation before the agent proceeds.

Cutover criteria — ALL must be true before starting:
- [ ] {criterion 1 from spec — e.g. "New implementation in production for 2 weeks, no incidents"}
- [ ] {criterion 2 — e.g. "All consumers confirmed updated to new interface"}
- [ ] {criterion 3 — e.g. "Zero calls to deprecated routes for 72 hours (confirmed via logs)"}

⚠️ This task CANNOT be run by an AI agent alone. A human must verify every criterion above, confirm with the team, then trigger this task.
</context>

<task>
Remove the deprecated {feature} implementation now that the new implementation is confirmed stable.

Steps:
1. Verify every cutover criterion above — STOP if any are not met
2. Remove deprecated implementation files: {exact file paths from spec Section 9.3}
3. Remove the feature flag / parallel-run switch at: {exact file + line}
4. Remove deprecation headers and log warnings from: {exact route files}
5. Remove regression tests for the old interface: {exact test files or describe blocks}
6. Update CLAUDE.md to remove references to the deprecated pattern

Files to remove: {list}
Files to modify: {list}
</task>

<constraints>
- Do NOT start unless every cutover criterion is confirmed by a human
- Remove only what is listed — no speculative cleanup
- Do not remove any test that covers the NEW implementation
</constraints>

<acceptance>
Run:
```bash
{full test suite command}
bun tsc --noEmit
grep -r "{deprecated symbol name}" src/ --include="*.ts" --include="*.tsx"
```
Done when:
- [ ] Full test suite passes with old code removed
- [ ] TypeScript clean
- [ ] grep returns zero results for deprecated symbol
- [ ] CLAUDE.md updated
- [ ] Team notified
</acceptance>

<commit>
```bash
git add {files removed or modified}
git commit -m "chore({scope}): remove deprecated {feature} implementation — cutover complete

Task: TASK-CUTOVER-01
TDD: N/A"
```
</commit>

<done_signal>
Reply with exactly one of: DONE | DONE_WITH_CONCERNS | BLOCKED
DONE: cutover complete, all checks passed.
DONE_WITH_CONCERNS: describe what was found during removal.
BLOCKED: a cutover criterion was not met — state which one and what is needed.
</done_signal>
```

---

## Subagent Response Handling

When a subagent returns, handle each response type:

| Response | What to do |
|---|---|
| `DONE` | Run the acceptance check. If green → mark task done, dispatch next. If red → provide context and re-dispatch same task. |
| `DONE_WITH_CONCERNS` | Read the concern. If it touches correctness or scope → resolve before proceeding. If it is an observation (file getting large, pattern choice) → note it and continue. |
| `NEEDS_CONTEXT` | The task's context block was incomplete. Provide the missing information and re-dispatch. Update the task's context block so it doesn't happen again. |
| `BLOCKED` | Assess: wrong context → re-dispatch with more. Task too large → split. Plan wrong → escalate to human. |

**Parallel dispatch rule:** Tasks in the same layer with different file sets can be dispatched simultaneously. Copy their Parallel Dispatch Blocks and send them concurrently. Do NOT wait for one to finish before starting another in the same layer — that defeats the parallelism.

---

## Summary Table

| Task ID | Name | Layer | Effort | Depends On | Parallel-safe With | Compat-sensitive | Commit type |
|---|---|---|---|---|---|---|
| TASK-0-01 | Regression tests for frozen contracts | 0 | {Xh} | none | TASK-0-02 | 🔒 Foundation |
| TASK-0-02 | Add deprecation markers | 0 | {Xh} | none | TASK-0-01 | 🔒 Foundation |
| TASK-0-02 | {name} | 0 | {Xh} | none | TASK-0-01 |
| TASK-1-01 | {name} | 1 | {Xh} | TASK-0-01 | TASK-1-02 |
| ... | | | | | |

**Total estimated effort (sequential):** {sum of all}
**Minimum elapsed time (full parallelism):** {critical path sum}

---

## Completion Checklist

Run these in order after all tasks are done:

```bash
# 1. Type check
bun tsc --noEmit

# 2. Full test suite
bun test

# 3. Build check
bun run build

# 4. {Feature-specific check from PRD definition of done}
```

All must pass before the feature is considered complete.
```

---

## Completion Message

After saving all files, output this exactly:

```
---

## 📁 Saved

{N} task files written to: `{folder}/tasks/`
TODO tracker:            `{folder}/tasks/index.md`

Files created:
  tasks/index.md       ← open this to track progress
  tasks/TASK-0-01.md
  tasks/TASK-0-02.md
  {... one line per task file}

**{N} tasks · {X} layers · {sequential}h sequential · {critical-path}h parallel minimum**

Layer 0: {N} tasks — start immediately
Layer 1: {N} tasks — {N} parallel tracks available
{...}

---

## ▶️ Next Step — Start Implementing

PRD folder: `{folder}/`
  📋 discovery.md  — Feature context
  📄 prd.md        — Requirements
  🔧 spec.md       — Technical design
  🗺️  plan.md       — Implementation sequence
  📂 tasks/        — One file per task + index

---

### Option A — Implement one task at a time

```bash
/prd-implement {folder}/tasks/TASK-0-01.md
```
After it finishes: open `tasks/index.md`, mark TASK-0-01 done, pick the next.

---

### Option B — Implement an entire layer at once (parallel)

```bash
/prd-implement --parallel-layer 0
```
Dispatches all Layer 0 task files as isolated subagents simultaneously.

---

### Option C — Implement specific parallel-safe tasks together

```bash
/prd-implement {folder}/tasks/TASK-1-01.md {folder}/tasks/TASK-1-02.md
```
Each file is sent to its own subagent. No sed gymnastics — just file paths.

---

### Layer 0 — start here:
{For each Layer 0 task:}
  ▸ {folder}/tasks/TASK-0-{N}.md — {name} ({Xh})
  → /prd-implement {folder}/tasks/TASK-0-{N}.md

{If parallel-safe:}
  ✨ All Layer 0 tasks are parallel-safe:
  → /prd-implement --parallel-layer 0

### Full sequence:
```
Layer 0:  /prd-implement --parallel-layer 0
Layer 1:  /prd-implement --parallel-layer 1   ← after Layer 0 index.md shows all ✅
Layer 2:  /prd-implement --parallel-layer 2
...
Cutover:  /prd-implement {folder}/tasks/TASK-CUTOVER-01.md  ← human gate
```

Track progress at any time: `cat {folder}/tasks/index.md`
```

---

**Critical rules:**
- ONE FILE PER TASK — never put two tasks in the same .md file
- index.md is the ONLY file updated as tasks progress — task files are immutable once written
- index.md never contains task content — only status, links, and effort metadata
- Every task file must be executable without reading any other document — all context embedded
- Every task file must have an exact acceptance check — no "verify it works" without saying how
- Every code snippet must come from real codebase files found in Phase 2 — never invented
- Tasks in the same layer that edit the same file must be marked sequential, not parallel-safe
- Effort estimates must be honest — round up, not down
- If a plan step cannot be decomposed below 4h, keep it as one task file and note why
- ENHANCEMENT ONLY: TASK-0-01.md (regression tests for frozen contracts) must be created first — it gates everything
- ENHANCEMENT ONLY: Every task file touching a frozen contract must include the frozen contract test command in its acceptance check
- ENHANCEMENT ONLY: TASK-CUTOVER-01.md must have a human-approval gate — it cannot be marked auto-executable
