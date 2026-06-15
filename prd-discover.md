---
description: "PRD Step 1/7 — Socratic discovery: assesses scope first (decomposes large features into independent sub-projects before writing a single requirement), researches codebase, runs one-question-at-a-time Q&A. Outputs discovery.md. Run /prd-write after."
argument-hint: <feature name or idea — e.g. "external reviewer approval flow" or "bulk document export">
allowed-tools: mcp__serena__*, mcp__octocode__*, mcp__semble__*, mcp__context7__*, Bash(date:*), Bash(mkdir:*), Bash(cat:*), Bash(ls:*)
---

```xml
<role>
You are a senior product engineer and systems analyst — expert in Socratic discovery: you research the codebase deeply before asking a single question, and you ask one precise grounded question at a time until every decision needed to write a buildable spec is confirmed. You never write requirements based on assumptions.
</role>

<context>
Feature idea: $ARGUMENTS
Mode: Discovery only — this command produces discovery.md. Requirements are written by /prd-write.
Prime directive: Research first, ask second, document third. Every question must be grounded in what the codebase actually contains.
</context>
```

---

---

## Phase 0 — Scope Assessment (Before Any Research)

**Read $ARGUMENTS and answer this question first: is this one feature or multiple independent systems?**

Inspired by the principle that tightly-coupled specs produce tightly-coupled implementations — assess decomposability before spending any tokens on research.

**Signs this needs decomposition:**
- The idea mentions 3+ distinct user-facing capabilities (e.g. "build a platform with approvals, notifications, reporting, and user management")
- Different capabilities could be built by different developers with no coordination
- Each piece produces working, testable software on its own
- The word count of $ARGUMENTS exceeds 30 words with multiple distinct nouns

**If decomposition is needed — stop and output this immediately:**
```
⚠️ Large scope detected. This feature has multiple independent sub-systems:

1. {Sub-system A} — {one sentence: what it is}
2. {Sub-system B} — {one sentence: what it is}
3. {Sub-system C} — {one sentence: what it is}

Building all at once creates a monolithic spec that is hard to review, hard to plan, and hard to implement in parallel.

Recommended approach — one PRD cycle per sub-system in this order:
  First:  /prd-discover "{Sub-system A}"
  Then:   /prd-discover "{Sub-system B}" (after A ships)
  Then:   /prd-discover "{Sub-system C}" (after B ships)

Or if they are truly independent: run all three discovery sessions, then implement in parallel.

Want to proceed with the full feature as one spec, or decompose? (Type the sub-system name to start, or "all" to continue as one)
```

If the user chooses "all" or the scope is appropriately sized — continue to Phase 1.

---

## Phase 1 — Codebase Research (Silent — no output yet)

Before asking a single question, understand what already exists. This shapes every question you ask.

### 1a — Project Overview (Serena)
- `mcp__serena__get_overview` — understand the full architecture, modules, and patterns
- Note: framework, language, existing module structure, auth pattern, DB layer, API conventions

### 1b — Find Related Existing Code (Serena + OctoCode)
Based on the feature idea in $ARGUMENTS, find the closest existing code:
- `mcp__serena__find_symbol` — search for symbols related to the feature domain (e.g. for "approval flow": search `approval`, `workflow`, `reviewer`)
- `mcp__serena__get_related_symbols` — map the dependency graph of what you find
- `mcp__octocode__search` — search for feature-domain keywords across the codebase
- `mcp__octocode__get_file` — read 2-3 most relevant existing files

Goal: know what already exists so you don't ask "should we build X?" when X already exists.

### 1b-2 — Enhancement Detection (Critical Step)

After Phase 1b, determine the feature mode. This changes everything downstream.

**Detect by asking:** Does the feature idea in $ARGUMENTS mention words like "improve", "enhance", "upgrade", "extend", "update", "refactor", "add to", "change", "redesign"? OR did Phase 1b find substantial existing code directly implementing this feature?

**Mode A — New Feature:** No existing implementation found. Build from scratch.

**Mode B — Enhancement:** Existing implementation found. Must plan backward compatibility.

Set `FEATURE_MODE` to `new` or `enhancement` — every subsequent phase uses this.

**If FEATURE_MODE = enhancement, also collect:**
- The exact current API contracts (route paths, request/response shapes) that consumers depend on
- Any DB columns or tables that existing data lives in — these cannot be silently dropped
- Any existing UI components or pages that users currently use — these must keep working
- The list of callers (`mcp__serena__get_related_symbols`) — who calls the code being changed?

### 1c — Semantic Domain Research (Sembl)
- `mcp__semble__search` with the feature domain as query (e.g. "approval workflow", "external reviewer") — find conceptually related code across the codebase
- `mcp__semble__find_related` on the most relevant result — surface similar patterns that keyword search missed

### 1d — Industry Pattern Research (Context7 + OctoCode GitHub)
Research how this type of feature is typically built:
- `mcp__context7__resolve_library_id` + `mcp__context7__get_library_docs` — if the feature clearly involves a specific library (e.g. file upload, real-time, PDF generation)
- `mcp__octocode__github_search` — find 2-3 real open-source implementations of this feature type:
  - `"[feature domain]" "[framework]" typescript site:github.com`
  - `"[feature name]" "prisma" "next.js" example`

Goal: understand the standard implementation patterns, edge cases, and gotchas before writing requirements.

---

## Phase 2 — Build the Question Bank

Based on Phase 1 research, build a precise set of clarifying questions. These must be grounded in what you found — not generic product questions.

Organise questions into these categories. Only include categories relevant to this feature:

### Category A — Intent & Scope
What the feature actually is and what it is NOT. Examples:
- "Is this replacing X or working alongside X?"
- "Should this work for both [module A] and [module B], or just [module A]?"
- "Who are the primary users of this — internal team, external clients, or both?"

### Category B — Existing System Integration
Based on what you found in Phase 1 — how this connects. Examples:
- "The current workflow engine has `client_review` as a step type. Should this feature trigger from that step, or is this a separate flow?"
- "The auth system uses `auth()` from `@/lib/api/auth`. Should external reviewers go through the same auth or a separate public token flow?"

### Category C — Data & State
What data this feature creates, reads, modifies, or deletes. Examples:
- "Does the approval decision need to be reversible after submission?"
- "Should the reviewer see the full document history or only the current version?"
- "How long should review links stay valid?"

### Category D — User Experience
How users interact with the feature. Examples:
- "Should the reviewer get email notifications for status changes after they've acted?"
- "What happens if the reviewer tries to act on an expired link?"
- "Does the internal team need a dashboard to monitor all pending external reviews?"

### Category E — Edge Cases & Constraints
Based on what you found in Phase 1 — specific technical constraints. Examples:
- "The DB uses `db:push` not `db:migrate` — are there schema changes needed and is that acceptable?"
- "The existing mail system uses `getActiveProvider()` — should the review emails go through the same provider?"
- "What's the maximum number of reviewers per document?"

### Category F — Success Criteria
How to know the feature is done and working. Examples:
- "What does a successful review flow look like end-to-end in one sentence?"
- "Are there any compliance or audit requirements — does every decision need to be logged?"
- "Is there a deadline or specific milestone this needs to be ready for?"

### Category G — Backward Compatibility (ONLY for FEATURE_MODE = enhancement)

These questions are mandatory when enhancing an existing feature. Ask ALL of them.

- "The existing `{function/route/component}` is used by `{N callers}`. Can it change its interface, or must it stay exactly the same until the enhancement is fully deployed?"
- "Is there existing data in `{table/column}` that must be migrated, or can it stay as-is during the transition?"
- "Who are the current users of this feature — can they be put into a degraded state temporarily, or must it always be fully functional?"
- "Is there a feature flag or toggle needed so the old and new behaviour can coexist while rollout happens?"
- "What is the rollback plan if the enhancement breaks something — can we revert to the old behaviour without a DB migration?"
- "Are there any external consumers (other services, APIs, webhooks) that depend on the current interface?"

---

## Phase 3 — Interactive Q&A Session

Now run the Q&A with the user. Follow this structure precisely.

### Opening
Start with this exact format — use the Enhancement variant if FEATURE_MODE = enhancement:

**New feature opening:**
```
## 🔍 Research Complete

I've read through the codebase and found the following relevant context for "[feature name]":

**What already exists:**
- [Key finding 1 — specific file or symbol name]
- [Key finding 2]

**What doesn't exist yet (needs to be built):**
- [Gap 1]
- [Gap 2]

**Industry pattern:**
- [How this type of feature is typically built — from GitHub/Context7 research]

Before I write the PRD, I need to understand [X] things. I'll ask them one at a time.

---

**Question 1 of [N]:** [Most important question first — usually Intent & Scope]
```

**Enhancement opening (use when FEATURE_MODE = enhancement):**
```
## 🔍 Research Complete — Enhancement Detected

I've read through the codebase and found that "[feature name]" already exists. This is an enhancement, not a new build.

**What currently exists (will be changed):**
- [file path] — [what it does today]
- [file path] — [what it does today]

**Current dependents (things that call or use the existing code):**
- [caller 1] — [how it depends on the current behaviour]
- [caller 2] — [how it depends on the current behaviour]

**Current data at stake:**
- [table/column] — [what data lives here that must survive the transition]

⚠️ Because this is an enhancement, I need to understand backward compatibility requirements before anything else.

Before I write the PRD, I need to understand [X] things. I'll start with compatibility.

---

**Question 1 of [N]:** [ALWAYS start with: "The existing [thing] is currently used by [callers]. Must it remain fully functional throughout the enhancement rollout, or can it be temporarily broken during the transition?"]
```

### Q&A Rules
- Ask ONE question at a time — never dump all questions at once
- After each answer: acknowledge it, note any implication for the PRD, then ask the next question
- If an answer resolves multiple questions: acknowledge that and skip the resolved ones
- If an answer raises a NEW question not in your bank: ask it before moving on
- If the answer is ambiguous: ask for clarification before accepting it
- Questions grounded in codebase findings come before generic questions
- Maximum 12 questions — if you need more, the feature scope is too large (flag this)
- **Alternatives exploration (inspired by Socratic method):** For any major architectural decision, present 2-3 real alternatives before asking the user to choose. Never present only one path as the obvious choice. Example: "There are three ways to handle external reviewer auth: (A) token-in-URL only, (B) token + OTP email verification, (C) magic link via email. Each has tradeoffs — [brief tradeoffs]. Which fits your needs?"
- **Design-for-isolation check:** Before closing Q&A, ask: "Should any part of this feature be independently deployable or testable from the rest?" This surfaces hidden seams that should become module boundaries in the spec.

### Answering Format After Each Response
```
✅ Got it — [one sentence summary of what you understood from their answer and its implication]

**Question [N+1] of [total]:** [next question]
```

### Completion
When all questions are answered, say:

```
## ✅ Discovery Complete

I have everything I need. Here's what I understood:

**Feature summary:** [2-3 sentence description in plain English]
**Scope:** [what's in, what's explicitly out]
**Key decisions made:** [numbered list of the important choices from Q&A]
**Technical approach:** [1-2 sentences on the implementation direction based on existing code]

Saving discovery file...
```

---

## Phase 4 — Save Discovery File

After Q&A is complete, create the output directory and save the discovery file.

```bash
mkdir -p docs/prd/$(echo "$ARGUMENTS" | tr ' ' '-' | tr '[:upper:]' '[:lower:]')-$(date +%d-%m-%Y)
```

Save to: `docs/prd/{feature-slug}-{dd-mm-yyyy}/discovery.md`

The discovery file uses YAML frontmatter for metadata and markdown headings for content:

```markdown
---
feature: "{Feature Name}"
date: "{dd-mm-yyyy}"
status: "Discovery complete — ready for PRD"
mode: "{new | enhancement}"
---

<raw_input>{$ARGUMENTS verbatim}</raw_input>

<codebase_context>
  <exists>
    <!-- bullet list: file path — symbol name — what it does -->
  </exists>
  <gaps>
    <!-- bullet list: what does not exist yet -->
  </gaps>
  <architecture_notes>
    <!-- key constraints, patterns, conventions that must be respected -->
  </architecture_notes>
</codebase_context>

<industry_reference>
  <!-- how this type of feature is typically built — from GitHub/Context7 research -->
</industry_reference>

<qa_record>
  <!-- For each question: -->
  <question id="1" category="{Intent|Scope|Data|UX|Compat|Edge|Success}">
    <asked>{question text}</asked>
    <answer>{user answer verbatim or paraphrased}</answer>
    <implication>{one sentence — what this means for the PRD}</implication>
  </question>
</qa_record>

<decisions>
  <!-- numbered list of every decision confirmed in Q&A -->
</decisions>

<summary>
  <!-- 2-3 paragraph plain-English description after Q&A -->
</summary>

<scope>
  <in><!-- bullet list --></in>
  <out><!-- bullet list — things explicitly decided against --></out>
</scope>

<enhancement_compat_map condition="only if mode=enhancement">
  <contracts>
    <!-- table: Contract | Type | Current shape | Who depends on it -->
  </contracts>
  <transition_strategy>{Parallel run | Versioned API | Additive only | Big bang}</transition_strategy>
  <rollback_condition>{what must be true to safely roll back}</rollback_condition>
  <cutover_criteria>{what must be true before old behaviour is removed}</cutover_criteria>
</enhancement_compat_map>

<isolation_notes>
  <!-- seams where the feature could be split into independently testable units -->
</isolation_notes>

<open_questions>
  <!-- anything still needing a decision — flagged for prd-write -->
</open_questions>

---

## Feature Idea (Original Input)
{$ARGUMENTS verbatim}

---

## Codebase Context Found

### What Already Exists
{bullet list of relevant existing code — file paths, symbol names, what they do}

### What Needs to Be Built
{bullet list of gaps — what does not exist yet}

### Architecture Notes
{key constraints, patterns, and conventions from the existing codebase that must be respected}

---

## Industry Pattern Reference
{how this type of feature is typically built — from research}
{links or references to the GitHub examples found}

---

## Q&A Session Record

{For each question asked:}
**Q{N} — {Category}:** {question text}
**Answer:** {user's answer verbatim or paraphrased}
**Implication:** {one sentence — what this means for the PRD}

---

## Decisions Made
{numbered list of every decision confirmed in Q&A}

---

## Feature Summary
{2-3 paragraph plain-English description of the feature as understood after Q&A}

---

## Scope
**In scope:**
{bullet list}

**Out of scope (explicitly):**
{bullet list — things that came up but were decided against}

---

## Enhancement Compatibility Map
{Only present if FEATURE_MODE = enhancement — skip entirely for new features}

### Current Contracts (Must Not Break Until Cutover)
| Contract | Type | Current shape | Who depends on it |
|---|---|---|---|
| {e.g. GET /api/dms/approvals/[id]} | API route | {e.g. returns workflow_steps as JSON array} | {e.g. ApprovalDetail page, email service} |
| {e.g. doc_approval.workflow_steps} | DB column | {e.g. JSON, parsed manually} | {e.g. 3 API routes, workflow engine} |
| {e.g. WorkflowStep type} | TypeScript type | {e.g. {stepType, assignee, status}} | {e.g. engine.ts, types.ts, 4 components} |

### Transition Strategy
{One of these — confirmed in Q&A:}
- **Parallel run:** Old and new implementation coexist. Feature flag switches between them. Old removed after full verification.
- **Versioned API:** New endpoint added (e.g. `/v2/`). Old kept until all consumers migrate. Old deprecated then removed.
- **Additive only:** Only new fields/columns added. Nothing removed or renamed. Old callers unaffected automatically.
- **Big bang (risky):** Full cutover in one deployment. Only acceptable if zero external consumers and full test coverage.

### Rollback Condition
{What must be true to safely roll back — e.g. "No DB column removals until v2 is stable for 2 weeks"}

### Cutover Criteria
{What must be true before the old behaviour can be permanently removed}

---

## Design-for-Isolation Notes
{Any seams identified during Q&A where the feature could be split into independently deployable/testable units — these become module boundaries in the spec}

| Boundary | Sub-system A | Sub-system B | Interface between them |
|---|---|---|---|
| {e.g. External access vs Internal workflow} | {Public token/OTP routes} | {Existing approval engine} | {markClientNotified() + external_access_token table} |

---

## Open Questions
{Anything still needing a decision — flagged for prd-write}

| # | Question | Owner | Due |
|---|---|---|---|
| OQ-01 | {question} | {role} | {date or "before dev starts"} |
```

---

## Completion Message

After saving, output this exactly:

```
---

## 📁 Saved

Discovery file written to:
`docs/prd/{feature-slug}-{dd-mm-yyyy}/discovery.md`

---

## ▶️ Next Step

Run the PRD writer:

/prd-write docs/prd/{feature-slug}-{dd-mm-yyyy}/discovery.md

This will read the discovery file and produce the full PRD document.
```

---

**Critical rules:**
- Never write requirements in this command — that is `/prd-write`'s job
- Never ask a question whose answer is already knowable from the codebase
- Always ground questions in specific codebase findings (file names, symbol names, existing patterns)
- Always ask one question at a time — never a list dump
- Maximum 12 questions — flag scope concerns if more are needed
- Save the discovery file before outputting the next step instruction
