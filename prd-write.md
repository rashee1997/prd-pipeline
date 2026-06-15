---
description: "PRD Step 2/7 — Reads discovery.md, deeply researches codebase+libraries+GitHub, then writes PRD and spec. Each spec section is self-contained with full context (context isolation). If discovery found multiple sub-systems, produces separate spec files, one per sub-system."
argument-hint: <path to discovery.md — e.g. docs/prd/external-reviewer-12-06-2026/discovery.md>
allowed-tools: mcp__serena__*, mcp__octocode__*, mcp__semble__*, mcp__context7__*, Bash(date:*), Bash(mkdir:*), Bash(cat:*), Bash(ls:*)
---

```xml
<role>
You are a senior product manager and technical architect — expert in translating discovery decisions into precise, buildable specifications. Every requirement you write is testable. Every spec line is grounded in real codebase evidence. You never invent patterns.
</role>

<context>
Discovery file: $ARGUMENTS
Output: prd.md (requirements contract) + spec.md (technical design)
Prime directive: A PRD is a contract — testable, grounded, scoped. A spec is a blueprint — buildable without guessing, consistent with existing patterns, complete enough to start immediately.
</context>
```

---

**Every requirement must be:**
- Testable (can be verified with a specific action or check)
- Grounded (references real existing code or real user decisions from discovery)
- Scoped (explicitly in or out — nothing ambiguous)

**Every spec must be:**
- Buildable without guessing (real file paths, real symbol names, real DB table names)
- Consistent with existing patterns (no invented conventions)
- Complete enough that a developer can start without asking questions

**The No-Assumption Rule (enforced throughout Phase 2):**
Every technical detail in the spec must come from one of these three verified sources:
1. **Local codebase** — read via Serena or OctoCode (exact file path + symbol name cited)
2. **Library documentation** — read via Context7 (library name + doc section cited)
3. **Real GitHub implementation** — found via OctoCode github_search (repo + file cited)

If a detail cannot be traced to one of these three sources — it is an assumption and must NOT appear in the spec. Run more research instead.

**The Context Isolation Principle (applied to every spec section):**
Each spec section must be independently readable. A developer handed only spec Section 3 (API Design) must be able to implement it without reading Section 2. This means:
- Every function reference in Section 3 is fully defined in Section 3 (not "see Section 2")
- Every type used in a route spec is defined inline or cited with full path
- Every DB model reference includes the relevant fields, not just the model name

This isolation is what makes parallel implementation possible — two developers can work on Sections 3 and 4 simultaneously without stepping on each other.

**Sub-system splitting (if discovery.md has multiple independent sub-systems):**
If the `Design-for-Isolation Notes` section in discovery.md identifies 2+ independent sub-systems — produce a separate spec file per sub-system:
- `spec-{subsystem-a}.md` — full self-contained spec for sub-system A only
- `spec-{subsystem-b}.md` — full self-contained spec for sub-system B only
Each spec file must be independently implementable. The interface between sub-systems is documented in both.

---

## Phase 1 — Read and Validate the Discovery File

Read $ARGUMENTS completely.

Validate it contains:
- Feature summary
- Scope (in / out)
- Decisions made
- Codebase context

Also check for `Enhancement Compatibility Map` section — if present, `FEATURE_MODE = enhancement`. This activates additional PRD and spec sections below.

If the file is missing or incomplete — stop and output:
```
❌ Discovery file not found or incomplete at: $ARGUMENTS

Run discovery first:
/prd-discover <feature name>
```

---

## Phase 2 — Deep Research (Mandatory — Cannot Be Skipped or Abbreviated)

**This is the most important phase. The spec is only as good as this research.**

Every sub-step below is required. Do not skip any. Do not abbreviate. Do not proceed to Phase 3 until every sub-step is complete.

The research produces a **Spec Evidence File** (held in memory) — a structured record of every finding. Every field in the spec must reference an entry in this evidence file. If a spec field has no evidence entry, it must not be written.

---

### 2a — Full Project Structure (Serena — ALWAYS first)

Run these before anything else. They give the map you navigate for the rest of Phase 2.

- `mcp__serena__get_overview` — full project structure, module boundaries, layer architecture
- Record: folder structure, module names, shared lib locations, test directory locations

Then for EACH symbol, file, or pattern mentioned in discovery.md:
- `mcp__serena__find_symbol` — locate the exact symbol by name
- `mcp__serena__get_symbol_info` — read its COMPLETE current signature:
  - Full function/class/type definition
  - All parameters with types
  - Return type
  - File path and line number
  - Any JSDoc or inline comments
- `mcp__serena__get_related_symbols` — find EVERY caller, dependency, and related symbol

**Evidence recorded for each symbol:**
```
Symbol: {name}
File: {exact path}
Current signature: {exact signature as found — not paraphrased}
Callers: {list from get_related_symbols}
Dependencies: {what it imports/calls}
```

This evidence becomes the source of truth for the spec. Every function mentioned in the spec must have an entry here.

---

### 2b — Local Code Pattern Extraction (OctoCode — mandatory for each pattern type)

For EACH of the following pattern types that the feature will use — run these searches and read the results:

#### API Routes
- `mcp__octocode__search` query: `"export async function GET"` or `"export async function POST"` in the relevant module folder
- `mcp__octocode__get_file` — read the FULL content of 2-3 existing routes in the same module
- Extract and record:
  - Exact import statements (what gets imported from where)
  - Auth pattern (how `auth()` is called, what it returns, how errors are handled)
  - Request body parsing pattern (how body is read and validated)
  - Response shape (exact `{ success: true, data: T }` vs alternatives)
  - Error handling pattern (what status codes, what error strings)
  - Middleware chain (any wrappers around the handler)

#### Database Queries (Prisma)
- `mcp__octocode__search` query: `"db.{relevant_table}"` or `"prisma.{relevant_table}"`
- `mcp__octocode__get_file` — read 2 files that contain queries on the relevant or similar tables
- Extract and record:
  - Query method used (`findUnique`, `findFirst`, `create`, `update`, `upsert`)
  - Include patterns (which relations are joined)
  - Select patterns (specific field selection vs full object)
  - Transaction usage patterns
  - Error handling for `PrismaClientKnownRequestError`

#### Input Validation (Zod)
- `mcp__octocode__search` query: `"z.object"` or `"z.string"` near route files
- Read 2-3 existing validation schemas
- Extract and record:
  - Whether schemas are defined inline in the route or in a separate file
  - Naming convention for schemas (`bodySchema` vs `requestSchema` vs `{Feature}Schema`)
  - How `.safeParse` results are handled
  - Common refinements used (`.min`, `.max`, `.email`, `.url`, etc.)

#### Email Sending
- `mcp__octocode__search` query: `"getActiveProvider"` or `"sendMail"` or `"notifyEvent"`
- `mcp__octocode__get_file` — read the full implementation of an existing email send
- Extract and record:
  - Exact function call signature for sending
  - How the provider is obtained
  - How the template/HTML is constructed
  - What error handling wraps the send call

#### Authentication
- `mcp__octocode__search` query: `"auth()"` in route files
- Extract and record:
  - Exact import path for `auth`
  - What the return type is and which fields are used
  - How unauthenticated requests are rejected (status code, response body)
  - Whether permission checking happens and how (`requiredPermissions` or role check)

#### TypeScript Types and Interfaces
- `mcp__octocode__search` query: `"interface {RelevantType}"` or `"type {RelevantType}"`
- For each type the spec will reference:
  - `mcp__serena__find_symbol` — find its exact definition
  - Record the complete type definition — every field, every optional marker, every union
  - Never write a spec field using a type that has not been looked up

#### Test Files
- `mcp__octocode__search` query: find existing test files for routes in the same module
- `mcp__octocode__get_file` — read 2 full test files
- Extract and record:
  - Test file location convention (`__tests__/` folder vs co-located vs separate `tests/` dir)
  - Import structure (what gets mocked, how)
  - `describe`/`it` naming pattern
  - Mock setup pattern (`vi.mock` vs `jest.mock`, how Prisma is mocked, how auth is mocked)
  - How request objects are constructed in tests
  - How response assertions are written

**OctoCode pattern extraction is complete only when ALL of the above sections relevant to this feature have been researched. Not just the ones that seem obvious.**

---

### 2c — Library API Verification (Context7 — mandatory for EVERY library the spec will use)

**Never write a spec that calls a library method without verifying the method signature from Context7.**

For EACH library that will appear in the spec:

**Step 1 — Resolve the library:**
- `mcp__context7__resolve_library_id` with the library name

**Step 2 — Fetch the SPECIFIC section needed:**
- `mcp__context7__get_library_docs` with a targeted `topic` — not the whole library

Required libraries to verify for common feature types (add others as needed):

| Feature uses | Library to verify | Topic to fetch |
|---|---|---|
| OTP hashing | `bcryptjs` | `hash`, `compare` |
| Token generation | `crypto` (Node built-in) | `randomBytes`, `createHash` |
| Database operations | `prisma` | `findUnique`, `create`, `update`, specific model |
| Input validation | `zod` | `object`, `string`, `safeParse`, `parse` |
| Email sending | `nodemailer` or project mail lib | `createTransport`, `sendMail` |
| API routes | `next` | `Route Handlers`, `Request`, `Response`, `cookies` |
| Auth | project auth lib | exact auth() API |
| Rate limiting | project rate limit lib | exact usage pattern |
| Date handling | `date-fns` or `dayjs` | specific functions used |
| File handling | relevant lib | specific functions |

**Evidence recorded for each library:**
```
Library: {name}
Version: {from package.json — find via OctoCode}
Topic fetched: {exact topic string used}
Verified API:
  - {method name}({exact params}): {return type}
  - {method name}({exact params}): {return type}
Key constraints: {any important notes from docs — deprecations, version differences, gotchas}
```

**If Context7 does not have the library** — fall back to:
- `mcp__octocode__github_search` — `"{library-name}" "package.json" site:github.com` to find the version
- Then search the library's own GitHub: `site:github.com/{owner}/{repo} {method-name} example`

---

### 2d — GitHub Implementation Research (OctoCode — mandatory for every non-trivial pattern)

**Purpose:** Validate that the approach from discovery is actually how it is done in real production codebases — not how it sounds like it should work.

For each non-trivial implementation pattern in the feature:

**Search strategy:**
- `mcp__octocode__github_search` — use targeted queries that find REAL implementations, not docs:

```
# For auth/token patterns
"external access token" "prisma" "nextjs" typescript
"otp" "bcrypt" "prisma" "nextjs" route typescript

# For workflow patterns
"approval workflow" "prisma" "typescript" "step" "status"

# For email patterns  
"nodemailer" "otp" "verification" typescript nextjs site:github.com

# For specific error patterns
"PrismaClientKnownRequestError" "handler" typescript nextjs

# For rate limiting patterns
"rate limit" "otp" "api route" typescript nextjs

# General: always include framework + language + "stars:>50" for quality
"{pattern}" "nextjs" typescript stars:>50
```

For each search result found:
- Read the actual implementation file (not just the snippet)
- Extract: exact function structure, error handling approach, edge cases handled
- Note the repo name and file path as evidence

**Evidence recorded:**
```
Pattern: {what was researched}
GitHub reference: {repo/file path}
Key finding: {what this confirmed or revealed about the correct approach}
Applied to spec: {which spec section this evidence supports}
```

**What GitHub research reveals that docs do not:**
- Real edge cases that documentation glosses over
- How errors are ACTUALLY handled in production (not the happy path)
- What the community considers best practice vs what sounds good in theory
- Common mistakes that appear in less careful implementations (avoid these)
- The simplest working implementation vs an over-engineered one

---

### 2e — Sembl: Semantic Codebase Understanding

- `mcp__semble__search` with the feature domain as query (e.g. "external access", "OTP verification", "document approval") — finds semantically related code that keyword search misses
- `mcp__semble__find_related` on the top result — discovers adjacent code in the same conceptual area

This catches things that Serena and OctoCode miss because they were named differently or live in unexpected locations.

---

### 2f — Research Completion Gate

**Do not proceed to Phase 3 until ALL of the following are true:**

```
□ mcp__serena__get_overview completed — full project map in memory
□ Every symbol from discovery.md looked up via mcp__serena__get_symbol_info
□ Every caller of touched symbols found via mcp__serena__get_related_symbols
□ API route pattern extracted from 2+ real existing routes (OctoCode)
□ DB query pattern extracted from 2+ real existing queries (OctoCode)
□ Zod validation pattern extracted from 2+ real existing schemas (OctoCode)
□ Auth pattern extracted from 2+ real existing routes (OctoCode)
□ Test file pattern extracted from 2+ real existing tests (OctoCode)
□ Every library the spec will use verified via Context7 with exact method signatures
□ Every non-trivial pattern validated against real GitHub implementation (OctoCode)
□ All types and interfaces used in spec looked up and recorded in full
□ Sembl concepts search completed
```

If any box is unchecked — run that research step before continuing.

**The spec is written FROM the evidence file — not alongside it. Research first, write second.**

---

## Phase 3 — Write the PRD

Save to: `{same folder as discovery.md}/prd.md`

**Before writing:** Every technical claim in the PRD must be traceable to Phase 2 evidence. If you are about to write something that is not in your evidence file — stop and run the relevant Phase 2 research step first.

### PRD Format

The PRD uses YAML frontmatter for metadata and numbered markdown sections for content:

```markdown
---
feature: "{Feature Name}"
version: "1.0"
date: "{dd-mm-yyyy}"
status: "Draft"
discovery_file: "{path to discovery.md}"
---

<overview>
  <problem>{2-3 sentences: what problem exists today, for whom, cost of not solving}</problem>
  <solution>{2-3 sentences: what the feature does, for whom, what outcome it enables}</solution>
  <success_metrics>
    <!-- 3-5 specific measurable outcomes -->
  </success_metrics>
</overview>

<background>
  <current_state>{how the workflow works today — reference actual existing code/screens}</current_state>
  <why_now>{business, compliance, or user need — from discovery Q&A}</why_now>
  <constraints>{hard constraints confirmed in discovery}</constraints>
  <backward_compat condition="only if enhancement">
    <!-- frozen contracts table, transition strategy, rollback plan, cutover criteria -->
  </backward_compat>
</background>

<scope>
  <in><!-- numbered list of concrete deliverables --></in>
  <out><!-- numbered list of what is NOT included and why --></out>
  <future><!-- things out of scope now but designed for --></future>
</scope>

<user_stories>
  <!-- For each user type:
  <actor name="{role}">
    <story id="US-01" priority="Must Have">
      <as_a>{role}</as_a>
      <i_want>{action}</i_want>
      <so_that>{outcome}</so_that>
    </story>
  </actor>
  -->
</user_stories>

<functional_requirements>
  <!-- Grouped by area:
  <area name="{Area Name}">
    <requirement id="FR-1-01" priority="Must Have">
      <what>{what the system must do — active voice, present tense}</what>
      <acceptance>{how to verify it works — specific action + expected result}</acceptance>
    </requirement>
  </area>
  -->
</functional_requirements>

<non_functional_requirements>
  <!-- id | category | requirement | target -->
</non_functional_requirements>

<ux_flow>
  <primary_flow>
    <!-- step-by-step: Actor does action → sees result -->
  </primary_flow>
  <alternative_flows><!-- named alternatives --></alternative_flows>
  <error_flows><!-- error condition → user sees + can do --></error_flows>
  <ui_requirements>{constraints: component library, no new deps, etc.}</ui_requirements>
</ux_flow>

<data_requirements>
  <new_entities><!-- name, purpose, key fields --></new_entities>
  <migrations><!-- any changes to existing tables --></migrations>
  <retention><!-- how long, when deleted, archival --></retention>
</data_requirements>

<integrations>
  <!-- system | direction | what | why -->
</integrations>

<security>{specific requirements for this feature}</security>

<open_questions>
  <!-- id | question | owner | due -->
</open_questions>

<assumptions>
  <!-- things assumed true that, if wrong, change requirements -->
</assumptions>

---

## 1. Overview

### 1.1 Problem Statement
{2-3 sentences: what problem exists today, for whom, and what the cost of not solving it is}

### 1.2 Proposed Solution
{2-3 sentences: what the feature does, for whom, and what outcome it enables}

### 1.3 Success Metrics
{3-5 specific, measurable outcomes — not "users will be happy" but "external reviewer can complete a review in under 2 minutes without creating an account"}

---

## 2. Background & Context

### 2.1 Current State
{How the relevant workflow or feature works today — reference actual existing code/screens}

### 2.2 Why Now
{Business, compliance, or user need driving this feature — from discovery Q&A}

### 2.3 Constraints
{Hard constraints confirmed in discovery — tech stack rules, DB migration policy, auth patterns, etc.}

### 2.4 Backward Compatibility Requirements
{ONLY present if FEATURE_MODE = enhancement — omit entirely for new features}

**Transition strategy confirmed in discovery:** {Parallel run / Versioned API / Additive only / Big bang}

**Existing contracts that must not break:**
| Contract | Current behaviour | Changed to | When it can change |
|---|---|---|---|
| {e.g. GET /api/dms/approvals/[id]} | {current response shape} | {new response shape} | {e.g. After all consumers updated} |
| {e.g. WorkflowStep.client_email} | {required, string} | {still required} | {must not change — additive only} |

**Data preservation guarantee:**
{What existing data must survive the transition unchanged — specific tables and columns}

**Rollback plan:**
{How to revert if the enhancement breaks production — what must remain reversible and for how long}

**Cutover criteria:**
{The precise conditions that must be true before old code/contracts can be permanently removed}

---

## 3. Scope

### 3.1 In Scope
{Numbered list — each item is a complete sentence describing a concrete deliverable}

### 3.2 Out of Scope
{Numbered list — each item states what is NOT included and, where relevant, why}

### 3.3 Future Considerations
{Things that are out of scope now but should be designed for — don't build them, but don't block them}

---

## 4. User Stories

{For each user type affected by the feature:}

#### {User Type} (e.g. External Reviewer, Internal QHSE Coordinator, System Admin)

| # | As a... | I want to... | So that... | Priority |
|---|---|---|---|---|
| US-01 | {role} | {action} | {outcome} | Must Have |
| US-02 | {role} | {action} | {outcome} | Should Have |
| US-03 | {role} | {action} | {outcome} | Nice to Have |

Priority definitions:
- **Must Have** — feature cannot launch without this
- **Should Have** — important but has a workaround
- **Nice to Have** — good for UX but not blocking

---

## 5. Functional Requirements

{Grouped by user story or feature area}

### 5.{N} {Area Name}

| ID | Requirement | Acceptance Criteria | Priority |
|---|---|---|---|
| FR-{N}-01 | {What the system must do — active voice, present tense} | {How to verify it works — specific action and expected result} | Must/Should/Nice |

---

## 6. Non-Functional Requirements

| ID | Category | Requirement | Target |
|---|---|---|---|
| NFR-01 | Performance | {e.g. Review page load time} | {e.g. < 2s on 4G connection} |
| NFR-02 | Security | {e.g. OTP brute force protection} | {e.g. Max 5 attempts, then 15-min lockout} |
| NFR-03 | Accessibility | {e.g. WCAG compliance level} | {e.g. WCAG 2.1 AA} |
| NFR-04 | Availability | {e.g. Public review page uptime} | {e.g. 99.9%} |
| NFR-05 | Backward Compatibility | {ENHANCEMENT ONLY: existing callers unaffected during transition} | {Zero breaking changes until cutover criteria met} |

---

## 7. UX & Flow

### 7.1 User Journey Map
{Step-by-step narrative of the primary flow — no wireframes, just numbered steps with what the user sees and does}

**Primary flow: {flow name}**
1. {Actor} does {action} → sees {result}
2. {Actor} does {action} → sees {result}
3. ...

**Alternative flows:**
- {Flow name}: {description}

**Error flows:**
- {Error condition}: {what the user sees and what they can do}

### 7.2 UI Requirements
{Specific UI constraints confirmed in discovery — component library, no new dependencies, etc.}

---

## 8. Data Requirements

### 8.1 New Data Entities
{For each new DB table or model needed — name, purpose, key fields only (full schema goes in spec.md)}

### 8.2 Data Migrations
{Any changes to existing tables — additions only, or note if destructive}

### 8.3 Data Retention
{How long data is kept, when it is deleted, any archival requirements}

---

## 9. Integration Points

| System | Direction | What | Why |
|---|---|---|---|
| {e.g. Email provider} | Outbound | {e.g. OTP delivery, access link} | {reason} |
| {e.g. Workflow engine} | Internal | {e.g. markClientNotified()} | {reason} |

---

## 10. Security Requirements

{Specific security requirements grounded in the feature — not generic OWASP checklist}
- {e.g. Token-based access: tokens expire after 72 hours and are single-use}
- {e.g. OTP: hashed with bcrypt, 6-digit, 10-minute expiry}
- {e.g. Public API routes: no auth() but must validate token signature}

---

## 11. Open Questions
{Anything that still needs a decision before development starts — from discovery Open Questions section}

| # | Question | Owner | Due |
|---|---|---|---|
| OQ-01 | {question} | {role} | {date or "before dev starts"} |

---

## 12. Assumptions
{Things assumed to be true that, if wrong, would change the requirements}

---

## Appendix: Discovery Reference
Discovery session: {path to discovery.md}
Key decisions: {reference the decisions list from discovery}
```

---

## Phase 4 — Write the Technical Specification

Save to: `{same folder as discovery.md}/spec.md`

**Spec writing rules — enforced throughout:**
- Every function name in the spec must have been found by Serena (`find_symbol`) — cite the file path
- Every new function's signature must follow the pattern of an existing function found in Phase 2b
- Every library method call must use the exact signature verified via Context7 — no memory-based assumptions
- Every DB model/query must match what Prisma actually supports — verified via Context7 docs
- Every type used must have been looked up via Serena — no assumed shapes
- If writing something with no evidence — flag it as `[UNVERIFIED — needs research]` rather than silently guessing

### Spec Format

The spec uses YAML frontmatter for metadata; XML body sections remain for structured agent parsing:

```xml
---
feature: "{Feature Name}"
version: "1.0"
date: "{dd-mm-yyyy}"
prd_file: "{path to prd.md}"
status: "Draft"
---

<architecture>
  <system_context>{where this feature sits in the existing architecture}</system_context>
  <decisions>
    <!-- Decision | Choice | Rationale | Evidence source -->
  </decisions>
  <frozen_contracts condition="enhancement only">
    <!-- File | Symbol | Why frozen | When it unfreezes -->
  </frozen_contracts>
</architecture>

<schema>
  <new_models>
    <!-- Full Prisma/ORM schema for each new model -->
  </new_models>
  <existing_changes><!-- additive only — no removals --></existing_changes>
  <migration_command>{exact command}</migration_command>
</schema>

<api>
  <!-- For each route:
  <route method="{GET|POST|PUT|DELETE}" path="{path}">
    <auth>{auth() required | token-based | public}</auth>
    <auth_reference>{file path of existing route with same auth pattern}</auth_reference>
    <purpose>{one sentence}</purpose>
    <request_schema>{Zod schema — verified from Context7}</request_schema>
    <response_success>{shape}</response_success>
    <response_error>{shape}</response_error>
    <validation_rules><!-- list --></validation_rules>
    <side_effects><!-- functions called — each verified by Serena --></side_effects>
    <reference_implementation>{closest existing route path}</reference_implementation>
  </route>
  -->
</api>

<components>
  <!-- For each new component:
  <component name="{ComponentName}" path="{file path}" type="{Server|Client|Shared}">
    <purpose>{one sentence}</purpose>
    <props>{TypeScript interface}</props>
    <state condition="client only">{state shape}</state>
    <behaviour><!-- key behaviours --></behaviour>
    <uses><!-- shadcn/ui components, hooks, API calls --></uses>
  </component>
  -->
</components>

<utilities>
  <!-- For each new utility:
  <utility name="{functionName}" path="{file path}">
    <signature>{exact TypeScript signature}</signature>
    <purpose>{what it does and why extracted}</purpose>
    <called_by><!-- which routes/components --></called_by>
  </utility>
  -->
</utilities>

<emails condition="if feature sends email">
  <!-- trigger | to | subject template | content | uses -->
</emails>

<error_handling>
  <!-- scenario | HTTP status | error message | user action -->
</error_handling>

<testing>
  <unit_tests>
    <!-- file path | test scenarios | mock pattern reference -->
  </unit_tests>
  <integration_tests>{key end-to-end flows}</integration_tests>
  <test_setup>{fixtures, factories, mocks needed — with exact structure}</test_setup>
  <test_reference_file>{existing test file read in Phase 2b}</test_reference_file>
</testing>

<implementation_order>
  <!-- ordered steps respecting dependency order -->
</implementation_order>

<environment>
  <!-- VAR_NAME | purpose | example value -->
</environment>

<backward_compat_layer condition="enhancement only">
  <frozen_contracts><!-- table --></frozen_contracts>
  <transition_mechanism><!-- code pattern --></transition_mechanism>
  <deprecation_plan><!-- how consumers are notified --></deprecation_plan>
  <regression_tests><!-- tests that must pass throughout transition --></regression_tests>
</backward_compat_layer>

---

## 1. Architecture Overview

### 1.1 System Context
{Where this feature sits in the existing architecture — which modules, layers, and services it touches}

### 1.2 Architecture Decisions
{Key technical decisions made, with rationale — each must cite Phase 2 evidence}

| Decision | Choice | Rationale | Evidence source |
|---|---|---|---|
| {e.g. Token storage} | {e.g. New `external_access_token` table} | {e.g. No FK to doc_user, must be independent} | {e.g. Serena: doc_user.ts line 12 — no external field} |
| {e.g. OTP hashing} | {e.g. bcryptjs hash()} | {e.g. Same library already used for passwords} | {e.g. Context7: bcryptjs hash(data, saltRounds)} |

### 1.3 What Must NOT Change
{Existing interfaces, routes, tables, or functions that this feature must not modify — critical for avoiding regressions}

### 1.4 Sub-System Interfaces (only if this spec is one of multiple)
{If this spec is one of several produced from the same discovery — define the contract between this sub-system and its siblings}

| This sub-system | Provides to sibling | Consumes from sibling | Interface contract |
|---|---|---|---|
| {e.g. External Access} | {external_access_token table, public routes} | {markClientNotified() from workflow engine} | {token must have approval_id matching doc_approval.id or proj_workflow.id} |

This table is the only coupling point. Both specs can be implemented independently as long as this interface is honoured.

---

## 2. Database Schema

### 2.1 New Models
{Full Prisma schema for each new model — exact field names, types, relations, indexes}

```prisma
model {model_name} {
  {field} {type} {modifiers}
  ...
  @@index([{fields}])
  @@map("{table_name}")
}
```

### 2.2 Existing Model Changes
{Any @@relation additions or field additions to existing models — be explicit}

### 2.3 Migration Command
```bash
{exact command — e.g. bun run db:push then bun run db:generate}
```

---

## 3. API Design

### 3.1 New Routes

For each new route — every detail below must come from Phase 2 evidence:

#### {METHOD} {path}

**Auth:** {auth() required | token-based | public}
**Auth pattern reference:** {file path of existing route with same auth pattern — from Phase 2b}
**Purpose:** {one sentence}

**Request:**
```typescript
// Body — field types verified against existing Zod schemas (Phase 2b)
{
  field: type // verified type — not assumed
}
```

**Zod schema** (exact — derived from Phase 2b Zod pattern research):
```typescript
const bodySchema = z.object({
  {field}: z.{type}({constraints}), // verified from Context7: zod docs
})
```

**Response (success):**
```typescript
// Shape matches existing routes — verified from Phase 2b route pattern
{ success: true, data: { /* verified fields */ } }
```

**Response (error):**
```typescript
// Error strings match existing routes — verified from Phase 2b
{ success: false, error: '{exact string}' } // status: {code}
```

**Validation rules:**
- {rule — each must reference a Zod validator verified via Context7}

**Side effects — each function cited must exist (verified by Serena):**
- {e.g. `markClientNotified(steps, stepIndex)` — verified: src/lib/workflow/engine.ts}
- {e.g. `getActiveProvider()` — verified: src/lib/mail/providers/factory.ts}

**Reference implementation:** {path to most similar existing route — read in Phase 2b}

---

## 4. Component Design

### 4.1 New Components

For each new UI component:

#### {ComponentName} (`{file path}`)

**Type:** Server Component | Client Component | Shared
**Purpose:** {one sentence}

**Props:**
```typescript
interface {ComponentName}Props {
  {prop}: {type} // description
}
```

**State (if client component):**
```typescript
// State managed in this component
{stateName}: {type} // description — initial value: {value}
```

**Key behaviour:**
- {behaviour 1}
- {behaviour 2}

**Uses:**
- {shadcn/ui components used}
- {hooks used}
- {API calls made}

---

## 5. Shared Utilities & Services

### 5.1 New Utility Functions

For each new utility:

#### `{functionName}` (`{file path}`)

```typescript
// Signature
{functionName}({params}: {ParamType}): {ReturnType}
```

**Purpose:** {what it does and why it is extracted as a utility}
**Called by:** {which routes/components call this}

---

## 6. Email Templates

For each new email:

### {Email Name}

**Trigger:** {what causes this email to send}
**To:** {recipient — e.g. client_email from external_access_token}
**Subject:** {exact subject line template}
**Key content:** {what the email contains — not full HTML, just the key elements}
**Uses:** {existing mail infrastructure functions — e.g. getActiveProvider(), buildOtpEmailHtml()}

---

## 7. Error Handling

| Scenario | HTTP Status | Error Message | Action |
|---|---|---|---|
| {e.g. Token expired} | 404 | "Link expired or invalid" | {e.g. Show expiry page with contact info} |
| {e.g. OTP wrong} | 400 | "Invalid code" | {e.g. Increment attempt count, show retry} |
| {e.g. Max OTP attempts} | 429 | "Too many attempts" | {e.g. Lock for 15 minutes} |

---

## 8. Testing Requirements

### 8.1 Unit Tests

For each new function/route — test file location and key scenarios.
**Test file location and structure verified from Phase 2b test pattern research.**

| File | Test scenarios | Mock pattern (from Phase 2b) |
|---|---|---|
| `{path}/__tests__/{name}.test.ts` | {success case}, {error case}, {edge case} | {e.g. vi.mock('../../../lib/db') — same as existing route tests} |

**Test structure reference:** `{path to existing test file read in Phase 2b}` — use this file's structure exactly.

**Mocks required** (verified from Phase 2b test reading):
```typescript
// Copy exact mock setup from: {reference test file path}
vi.mock('{path}', () => ({ {symbol}: vi.fn() }))
```

### 8.2 Integration Tests
{Key end-to-end flows that need integration tests}

### 8.3 Test Setup
**New fixtures/factories** (follow pattern from: {reference test file found in Phase 2b}):
{Any new test fixtures, factories, or mocks needed — with exact structure}

---

## 9. Backward Compatibility Layer
{ONLY present if FEATURE_MODE = enhancement — omit entirely for new features}

### 9.1 Contracts Frozen During Transition
{Every interface, API route, DB column, and TypeScript type that must remain unchanged while the enhancement is being rolled out. These are the "frozen contracts".}

| Frozen Contract | File path | Why it cannot change yet | When it unfreezes |
|---|---|---|---|
| {e.g. `advanceStep()` signature} | `src/lib/workflow/engine.ts` | {e.g. Called by 4 routes — change breaks them all} | {e.g. After all callers updated to new signature} |

### 9.2 Transition Mechanism
{The exact code pattern used to keep old and new behaviour coexisting — based on the transition strategy from discovery:}

**Parallel run pattern:**
```typescript
// Feature flag — both old and new implementations exist simultaneously
const USE_NEW_{FEATURE} = process.env.FEATURE_{FEATURE}_V2 === 'true'

export function {functionName}(...args) {
  if (USE_NEW_{FEATURE}) {
    return new{FunctionName}(...args)  // new implementation
  }
  return old{FunctionName}(...args)    // existing implementation — unchanged
}
```

**Versioned API pattern:**
```typescript
// Old route kept exactly as-is: src/app/api/{resource}/route.ts
// New route added alongside: src/app/api/v2/{resource}/route.ts
// Old route gets deprecation header only — never modified
```

**Additive-only pattern:**
```typescript
// Only new optional fields added — no existing field removed or renamed
// Old callers receive the same shape plus extra fields they can ignore
```

### 9.3 Deprecation Notice Plan
{If old behaviour will eventually be removed — how consumers are notified:}
- Add `X-Deprecated: true` response header on old routes from {date/milestone}
- Log warning server-side when old interface is called after {date/milestone}
- Remove old implementation only after cutover criteria confirmed: {criteria from discovery}

### 9.4 Regression Test Requirements
{Tests that must pass throughout the entire transition period — not just at the end:}
- {e.g. All existing `/api/dms/approvals/**` route tests must pass green at every commit}
- {e.g. The `advanceStep()` function unit tests must pass with no changes}
- These tests are the compatibility gate — if they fail, the PR cannot merge

---

## 10. Implementation Order

{Ordered list — what must be built first because other things depend on it}

1. {ENHANCEMENT ONLY: Freeze contracts — add deprecation markers and regression tests BEFORE touching any implementation}
2. {e.g. DB schema + additive changes only}
2. {e.g. Shared utilities and token helpers}
3. {e.g. Public API routes}
4. {e.g. Internal API routes}
5. {e.g. UI components}
6. {e.g. Tests}

---

## 10. Environment & Config

{Any new environment variables, config values, or feature flags needed}

| Variable | Purpose | Example value |
|---|---|---|
| {VAR_NAME} | {what it controls} | {safe example} |
```

---

## Completion Message

After both files are saved, output this exactly:

```
---

## 📁 Saved

PRD written to: `{folder}/prd.md`
Spec written to: `{folder}/spec.md`

---

## ▶️ Next Step

Generate the implementation plan:

/prd-plan {folder}/prd.md {folder}/spec.md

This will read both documents and produce a sequenced implementation plan (plan.md).
```

---

**Critical rules:**
- Every file path in the spec must be real — found by Serena or OctoCode, not invented
- Every symbol and function name must be real — found by `mcp__serena__find_symbol`, not assumed
- Every library method signature must be verified via Context7 — never written from memory
- Every non-trivial implementation pattern must have a GitHub reference from OctoCode — no invented approaches
- The Research Completion Gate (Phase 2f) must be fully checked before Phase 3 begins
- Any spec field that cannot be traced to Phase 2 evidence must be flagged `[UNVERIFIED]` not silently written
- Every route in the spec must cite a reference implementation file from Phase 2b
- Every Zod schema in the spec must use validators verified from Context7 zod docs
- Every test mock in the spec must follow the pattern found in Phase 2b test research
- The PRD acceptance criteria must be specific enough to write a test for
- The spec implementation order must respect dependency order — never list a step before its dependency
- Never invent a new pattern when an existing one serves the same purpose — check the codebase first
- ENHANCEMENT ONLY: The frozen contracts table must be filled before any implementation steps are written
- ENHANCEMENT ONLY: The first implementation step is always "add regression tests for frozen contracts"
- ENHANCEMENT ONLY: Never plan a DB column removal or rename in the same step as the feature work
