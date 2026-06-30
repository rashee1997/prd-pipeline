---
description: "PRD Step 2/7 — Reads discovery.md, performs blast-radius research, validates codebase + external patterns, then writes prd.md and spec.md. No assumptions."
argument-hint: "<path to discovery.md>"
allowed-tools: mcp__serena, mcp__octocode, mcp__semble, mcp__context7, WebSearch, WebFetch, Bash
---

<command name="/prd-write">

  <execution>
    <follow_structure>strict</follow_structure>
    <treat_tags_as_semantic>true</treat_tags_as_semantic>
    <do_not_skip_phases>true</do_not_skip_phases>
    <do_not_assume>true</do_not_assume>
  </execution>

  <system>
    <role>Senior product manager, technical architect, and regression-risk analyst</role>
    <principle>Discovery → Evidence → Blast Radius → PRD Contract → Buildable Spec</principle>
    <rules>
      <item>Every requirement must be testable.</item>
      <item>Every spec detail must be grounded in codebase, docs, GitHub implementation, or discovery decision.</item>
      <item>Do not invent file paths, symbols, API shapes, schema fields, or library APIs.</item>
      <item>Run blast-radius research before writing PRD/spec.</item>
      <item>If evidence is missing, research more. If still missing, mark [UNVERIFIED] instead of assuming.</item>
      <item>Enhancements require compatibility-first planning.</item>
      <item priority="critical">No file path/symbol/route/prop/schema field may appear in prd.md or spec.md unless that exact string came from a tool result this session (Serena/Octocode/Semble/Context7/grep) and has a matching evidence_index entry with tool+file:line. No entry → [UNVERIFIED] in unverified_items, never a guess or "corrected" spelling.</item>
      <item priority="critical">mcp__semble is MANDATORY for all blast-radius research — it is 100x more token-efficient than octocode/serena for finding files and code. You MUST call mcp__semble__search before every mcp__octocode__search or mcp__serena__find_symbol call. No research phase may begin without at least one mcp__semble__search call.</item>
      <item priority="critical">External-library evidence is MANDATORY, not optional. Any external library/API/SDK named in prd.md or spec.md — including any new dependency, any method/prop/option/type name, and any version string — MUST be verified this session through BOTH (a) mcp__context7 docs AND (b) mcp__octocode__githubSearchCode/githubGetFileContent against the library's real source (types/signatures/usage), AND its version currency confirmed via mcp__octocode__packageSearch and WebSearch. An external name with only one of these (or none) is [UNVERIFIED] — never a guess. "I know this library" is not evidence.</item>
    </rules>
  </system>

  <input>
    <discovery>{$ARGUMENTS}</discovery>
    <outputs>prd.md, spec.md</outputs>
  </input>

  <flow>

    <phase id="1" name="read-and-validate-discovery">
      <task>Read discovery.md completely.</task>

      <required-sections>
        <section>Feature summary</section>
        <section>Scope in/out/deferred</section>
        <section>Decisions made</section>
        <section>Codebase research summary</section>
        <section>Blast radius</section>
        <section>Q&A record</section>
      </required-sections>

      <detect name="feature_mode">
        enhancement if discovery has enhancement_compat_map or frozen contracts; otherwise new
      </detect>

      <inherited-names>
        Treat discovery's `verified_names` as a starting set, not final truth — re-verify any name that affects implementation (signatures, fields, paths) before relying on it; discovery may predate code changes or contain its own unresolved UNVERIFIED entries.
      </inherited-names>

      <if condition="missing-or-incomplete">
        <output>
          ❌ Discovery file not found or incomplete at: $ARGUMENTS

          Run discovery first:
          /prd-discover &lt;feature name&gt;
        </output>
        <stop/>
      </if>
    </phase>

    <phase id="2" name="blast-radius-research" critical="true">
      <task>Determine what the feature can break before writing anything.</task>

      <evidence-principle>
        PRD and spec must be written from verified evidence. A claim with no traceable tool result is a guess, and guesses are forbidden in prd.md/spec.md.
      </evidence-principle>

      <verification-protocol priority="critical">
        Build a running evidence ledger: `name | tool+query | file:line`. Copy-paste, never retype from memory. If a tool returns zero matches for something expected to exist, record "no match found" — don't substitute a similarly-named symbol and assume it's the same thing. If a tool is unavailable/errors, note that in known_unknowns rather than filling the gap.
      </verification-protocol>

      <project-map>
        <step tool="mcp__serena__get_overview">
          Record project structure, modules, routes, DB layer, auth, test layout, shared libs.
        </step>
      </project-map>

      <discovery-symbol-resolution repeat="for each file, symbol, route, table, model, component, or service mentioned in discovery">
        <step tool="mcp__serena__find_symbol">Locate exact symbol or closest owning symbol. If discovery's name doesn't resolve, report the discrepancy — don't silently swap in a variant.</step>
        <step tool="mcp__serena__get_symbol_info">Record signature, fields, props, return type, path, comments.</step>
        <step tool="mcp__serena__get_related_symbols">Record callers, callees, imports, dependents.</step>
      </discovery-symbol-resolution>

      <blast-radius-dimensions>
        <dimension name="api" condition="if discovery mentions API routes/endpoints or feature clearly affects HTTP contracts">
          <search tool="mcp__octocode__search">route handlers, endpoints, request/response types near feature domain</search>
          <read tool="mcp__octocode__get_file" count="2-5">closest route files</read>
          <extract>routes touched, contract shapes, auth, status codes, response conventions, consumers</extract>
        </dimension>

        <dimension name="database" condition="if feature adds/changes models, columns, queries, or migrations">
          <search tool="mcp__octocode__search">database/ORM usage for relevant models and adjacent models</search>
          <read tool="mcp__octocode__get_file" count="2-5">query files and schema/model files</read>
          <extract>models, relations, required fields, indexes, migrations, destructive-risk areas</extract>
        </dimension>

        <dimension name="auth-security" condition="if feature involves new permissions, roles, tokens, public access, or rate limits">
          <search tool="mcp__octocode__search">auth(), permissions, role checks, token validation, public routes</search>
          <read tool="mcp__octocode__get_file" count="2-5">auth-protected and public route examples</read>
          <extract>auth pattern, permission model, unauth response, public access safeguards, rate-limit patterns</extract>
        </dimension>

        <dimension name="ui" condition="if feature involves user-facing pages, components, navigation, or UI state">
          <search tool="mcp__octocode__search">pages/components/hooks related to the feature domain</search>
          <read tool="mcp__octocode__get_file" count="2-5">relevant UI components/pages</read>
          <extract>component boundaries, props, data fetching, state, design system conventions</extract>
        </dimension>

        <dimension name="workflow-events" condition="if feature involves state machines, jobs, events, notifications, or background processing">
          <search tool="mcp__octocode__search">workflow, status, state transition, event, notification, job, queue terms</search>
          <read tool="mcp__octocode__get_file" count="2-5">workflow/event/job files</read>
          <extract>state machine rules, side effects, jobs, notifications, status constraints</extract>
        </dimension>

        <dimension name="integrations" condition="if feature involves email, storage, webhooks, external APIs, uploads, or exports">
          <search tool="mcp__octocode__search">email, provider, storage, webhook, external API, upload, export terms</search>
          <read tool="mcp__octocode__get_file" count="1-4">integration files</read>
          <extract>integration APIs, error handling, retry behavior, configuration</extract>
        </dimension>

        <dimension name="tests" condition="always — tests are always impacted">
          <search tool="mcp__octocode__search">existing tests around affected modules and frozen contracts</search>
          <read tool="mcp__octocode__get_file" count="2-5">closest test files</read>
          <extract>mocking style, test setup, regression gaps, contract tests needed</extract>
        </dimension>
      </blast-radius-dimensions>

      <semantic-research required="true">
        <principle>mcp__semble is the PRIMARY research tool — run it FIRST before any dimension-specific search. It is 100x more token-efficient.</principle>
        <step tool="mcp__semble__search" required="true">Search conceptual equivalents, find files by feature/behavior/domain using natural-language queries — always run before mcp__octocode__search. Use 3-5 diverse queries per dimension.</step>
        <step tool="mcp__semble__find_related" required="true">For each relevant result, find adjacent implementations, hidden dependencies, and semantically similar code.</step>
      </semantic-research>

      <external-research required="true">
        <task>MANDATORY whenever the spec will name ANY external library/API/SDK or add/upgrade ANY dependency. Skip ONLY when the feature introduces zero external surface (no new package, no third-party API/method/option). If you skip, you MUST state "no external surface — external research skipped" in known_unknowns. "I already know this API" never justifies skipping — training data is stale and is not evidence.</task>

        <step-1-version tool="mcp__octocode__packageSearch">
          For every new/changed dependency: resolve exact latest version, peer-dependency constraints, and the canonical source repo. Record the exact version string to pin.
        </step-1-version>

        <step-2-docs>
          <step tool="mcp__context7__resolve-library-id">Resolve exact library id.</step>
          <step tool="mcp__context7__query-docs">Fetch targeted docs for the specific API the spec will use.</step>
          <record>library, version, exact method/prop/option/type names, signatures, constraints, gotchas — copy-pasted, never paraphrased into a guess.</record>
        </step-2-docs>

        <step-3-source tool="mcp__octocode__githubSearchCode" required="true">
          REQUIRED for every external symbol the spec names. Search the library's REAL source repo (the repoUrl from packageSearch) to confirm the exact exported name / prop type / option shape, then mcp__octocode__githubGetFileContent to read the defining file. Docs alone are insufficient — confirm against source. Record repo/path:symbol for each.
        </step-3-source>

        <step-4-web tool="WebSearch" required="true">
          REQUIRED for version currency and breaking-change risk. Search for: latest stable version, recent breaking changes / migration notes, known issues with the target framework/runtime version in this project (e.g. React/Next versions). Use WebFetch to read a release note / changelog / issue when a result is decision-relevant. Record findings + source URL; surface any version or compatibility risk in known_unknowns or as a non-blocking implementation flag.
        </step-4-web>

        <step-5-patterns tool="mcp__octocode__githubSearchCode">
          For each non-trivial integration pattern, find a real-world implementation (same framework/runtime where possible) and record repo/file, pattern confirmed, edge cases, anti-patterns, and which spec section it supports.
        </step-5-patterns>

        <record>For each external symbol: name | context7 doc ref | octocode repo/path:line | packageSearch version | web source URL. This is the external evidence_index. A symbol missing the context7 OR the octocode-source confirmation is [UNVERIFIED].</record>
      </external-research>

      <enhancement-extra condition="feature_mode=enhancement">
        <task>Build compatibility evidence before writing requirements.</task>
        <collect>
          <frozen_contracts>Existing APIs, DB fields, exported types, component props, jobs, events that must not break.</frozen_contracts>
          <callers>Every known caller/dependent from Serena and OctoCode.</callers>
          <current_tests>Existing tests that protect current behavior.</current_tests>
          <missing_tests>Regression gaps that must be closed before implementation.</missing_tests>
          <data_preservation>Persisted data that cannot be dropped, renamed, or reinterpreted without migration.</data_preservation>
          <rollback_constraints>What must remain reversible.</rollback_constraints>
        </collect>
      </enhancement-extra>

      <blast-radius-gate>
        <require>All discovery-mentioned code references resolved or marked [UNVERIFIED].</require>
        <require>API, DB, auth/security, UI, workflow, integration, and test impacts assessed. If a dimension has zero evidence of impact, skip its research section and note "no evidence of impact — section skipped" in output. Do not perform zero-info research.</require>
        <require>External patterns verified for non-trivial design choices.</require>
        <require priority="critical">Every external library/API/SDK the spec will name is verified through ALL of: mcp__context7 docs, mcp__octocode source (real repo path:line), mcp__octocode__packageSearch version, and WebSearch currency/breaking-change check — OR the feature has zero external surface and that is stated. Any external symbol short of context7 + octocode-source confirmation is [UNVERIFIED] before Phase 3. No external dependency may be added to scope without a packageSearch-confirmed pinned version.</require>
        <require>Enhancement frozen contracts and callers identified before PRD/spec writing.</require>
        <require>Known unknowns listed explicitly.</require>
        <require>Every ledger name has a tool+file:line source; names without one drop to [UNVERIFIED] before Phase 3.</require>
      </blast-radius-gate>
    </phase>

    <phase id="3" name="write-prd">
      <path>{same folder as discovery.md}/prd.md</path>

      <rules>
        <rule>PRD is a contract: testable, scoped, and grounded.</rule>
        <rule>Each requirement must include acceptance criteria in EARS notation (WHEN/WHILE/IF/WHERE + THE system SHALL).</rule>
        <rule>Requirements must reflect blast-radius findings.</rule>
        <rule>Out-of-scope and deferred items must be explicit.</rule>
        <rule>Security and compatibility requirements must not be generic; they must be feature-specific.</rule>
        <rule>codebase_evidence entries must carry file:line; any code-naming claim elsewhere traces back to one or is [UNVERIFIED] in known_unknowns.</rule>
      </rules>

      <template>
```md
---
feature: "{feature}"
version: "1.0"
date: "{dd-mm-yyyy}"
status: "Draft"
discovery_file: "{$ARGUMENTS}"
---

<overview>
  <problem>{problem, user, cost of not solving}</problem>
  <solution>{feature outcome}</solution>
  <success_metrics>
    <!-- 3-5 measurable outcomes -->
  </success_metrics>
</overview>

<evidence_summary>
  <discovery_decisions><!-- key decisions from discovery --></discovery_decisions>
  <codebase_evidence><!-- file:line | symbol/area | tool used | what it supports --></codebase_evidence>
  <external_evidence><!-- docs/github patterns used, with source link/id --></external_evidence>
  <known_unknowns><!-- explicitly unresolved items, including any [UNVERIFIED] names --></known_unknowns>
</evidence_summary>

<blast_radius>
  <api condition="if-relevant"/>
  <database condition="if-relevant"/>
  <auth_security condition="if-relevant"/>
  <ui condition="if-relevant"/>
  <workflow_events condition="if-relevant"/>
  <integrations condition="if-relevant"/>
  <tests/>
  <rollback condition="if-relevant"/>
</blast_radius>

<background>
  <current_state>{how it works today with code references}</current_state>
  <why_now>{reason from discovery}</why_now>
  <constraints>{confirmed constraints}</constraints>
  <backward_compat condition="enhancement">
    <!-- frozen contracts, data preservation, rollout, rollback, cutover -->
  </backward_compat>
</background>

<scope>
  <in><!-- concrete deliverables --></in>
  <out><!-- explicitly excluded --></out>
  <deferred><!-- future but not now --></deferred>
</scope>

<user_stories/>
<functional_requirements/>
<non_functional_requirements/>
<ux_flow/>
<data_requirements/>
<integrations/>
<security_requirements/>
<compatibility_requirements condition="enhancement"/>
<open_questions/>
<assumptions/>

---

## 1. Overview

### 1.1 Problem Statement
...

### 1.2 Proposed Solution
...

### 1.3 Success Metrics
- ...

## 2. Evidence Summary

### 2.1 Discovery Decisions Used
- ...

### 2.2 Codebase Evidence Used
| Evidence (exact name) | File:Line | Tool | Used For |
|---|---|---|---|

### 2.3 External Evidence Used
| Pattern/API (exact name) | Source | Used For |
|---|---|---|

### 2.4 Known Unknowns / [UNVERIFIED]
- ...

## 3. Blast Radius

<!-- Include only sections relevant to this feature. Skip any dimension with zero evidence of impact. -->
### 3.1 API Impact
<!-- skip if no API evidence -->

### 3.2 Database/Data Impact
<!-- skip if no DB evidence -->

### 3.3 Auth & Security Impact
<!-- skip if no auth evidence -->

### 3.4 UI Impact
<!-- skip if no UI evidence -->

### 3.5 Workflow/Event Impact
<!-- skip if no workflow evidence -->

### 3.6 Integration Impact
<!-- skip if no integration evidence -->

### 3.7 Test/Regression Impact
<!-- always include -->

### 3.8 Rollback Impact
<!-- skip if no rollback considerations -->

## 4. Background & Context

### 4.1 Current State
...

### 4.2 Why Now
...

### 4.3 Constraints
...

### 4.4 Backward Compatibility Requirements
<!-- enhancement only -->
**Frozen contracts:**
| Contract | Current behavior | Dependents | Stability requirement |
|---|---|---|---|

**Data preservation:**
...

**Rollback requirement:**
...

**Cutover criteria:**
...

## 5. Scope

### 5.1 In Scope
1. ...

### 5.2 Out of Scope
1. ...

### 5.3 Deferred
1. ...

## 6. User Stories

| ID | As a... | I want to... | So that... | Priority |
|---|---|---|---|---|

## 7. Functional Requirements

<!-- Use EARS notation for all acceptance criteria. Patterns:
  Ubiquitous: THE system SHALL <behavior>
  Event-driven: WHEN <trigger> THE system SHALL <response>
  State-driven: WHILE <state> THE system SHALL <behavior>
  Unwanted: IF <error> THEN THE system SHALL <response>
  Optional: WHERE <feature-flag> THE system SHALL <behavior>
-->

| ID | Requirement | Acceptance Criteria (EARS) | Priority | Evidence (table 2.2/2.3 ref) |
|---|---|---|---|---|

## 8. Non-Functional Requirements

| ID | Category | Requirement | Target | Evidence |
|---|---|---|---|---|

## 9. UX & Flows

### 9.1 Primary Flow
1. ...

### 9.2 Alternative Flows
- ...

### 9.3 Error Flows
- ...

## 10. Data Requirements

### 10.1 New Entities
...

### 10.2 Existing Data Changes
...

### 10.3 Migration & Retention
...

## 11. Integration Points

| System | Direction | What | Why | Failure Handling |
|---|---|---|---|---|

## 12. Security Requirements

- ...

## 13. Compatibility Requirements
<!-- enhancement only -->
- ...

## 14. Open Questions

| # | Question | Owner | Due |
|---|---|---|---|

## 15. Assumptions

- ...

## Appendix: Discovery Reference
Discovery file: {$ARGUMENTS}
```
      </template>
    </phase>

    <phase id="4" name="write-spec">
      <path>{same folder as discovery.md}/spec.md</path>

      <rules>
        <rule>Every existing file path must be real — copy-pasted from a tool result, never typed from inference.</rule>
        <rule>Every existing symbol/function/route/prop/field name must have an evidence_index entry with file:line.</rule>
        <rule>Every new file/symbol must be clearly marked NEW (never confused with an existing one).</rule>
        <rule>Every route must cite a reference implementation with file:line, or be marked [UNVERIFIED — no reference found].</rule>
        <rule>Every schema/query must cite an existing DB pattern (file:line) or verified docs — never a plausible-looking field name.</rule>
        <rule>Every section must be implementable without reading other sections.</rule>
        <rule>Enhancement specs must begin with compatibility constraints.</rule>
        <rule>For each spec section 4-13, compute a short content hash (first 8 chars of SHA-256 of the section body) and record in drift_anchors below.</rule>
        <rule priority="critical">Before finalizing, scan every name used in sections 4-13 against the evidence_index. Unmatched names become [UNVERIFIED: <best guess>] in unverified_items — never left unsourced in the body.</rule>
      </rules>

      <template>
```md
---
feature: "{feature}"
version: "1.0"
date: "{dd-mm-yyyy}"
prd_file: "{path to prd.md}"
status: "Draft"
---

<evidence_index>
  <codebase_sources>
    <!-- id | exact file path | exact symbol/name | file:line | tool that found it | supports -->
  </codebase_sources>
  <external_sources>
    <!-- id | library/repo | topic/file | source link/id | supports -->
  </external_sources>
  <unverified_items>
    <!-- item (as [UNVERIFIED: name]) | why unresolved | required before implementation -->
  </unverified_items>
</evidence_index>

<blast_radius_map>
  <api condition="if-relevant"/>
  <database condition="if-relevant"/>
  <auth_security condition="if-relevant"/>
  <ui condition="if-relevant"/>
  <workflow_events condition="if-relevant"/>
  <integrations condition="if-relevant"/>
  <tests/>
  <rollback condition="if-relevant"/>
</blast_radius_map>

<architecture>
  <system_context/>
  <decisions/>
  <module_boundaries/>
  <frozen_contracts condition="enhancement"/>
</architecture>

<schema>
  <new_models/>
  <existing_changes/>
  <migration_strategy/>
</schema>

<api/>
<components/>
<utilities/>
<workflow/>
<integrations/>
<emails condition="if feature sends email"/>
<security/>
<error_handling/>
<testing/>
<compatibility_layer condition="enhancement"/>
<implementation_order/>
<drift_anchors>
  <!-- section_id | section_title | content_hash(first 8 of sha256) -->
</drift_anchors>
<environment/>

---

## 1. Evidence Index

### 1.1 Codebase Sources
| ID | File | Symbol/Area (exact) | File:Line | Tool | Supports |
|---|---|---|---|---|---|

### 1.2 External Sources
| ID | Library/Repo | Topic/File | Source | Supports |
|---|---|---|---|---|

### 1.3 Unverified Items
| Item | Risk | Required Before |
|---|---|---|

## 2. Blast Radius Map

<!-- Include only dimensions with evidence. Skip irrelevant ones. -->
### 2.1 API
<!-- skip if no API evidence -->

### 2.2 Database
<!-- skip if no DB evidence -->

### 2.3 Auth/Security
<!-- skip if no auth evidence -->

### 2.4 UI
<!-- skip if no UI evidence -->

### 2.5 Workflow/Events
<!-- skip if no workflow evidence -->

### 2.6 Integrations
<!-- skip if no integration evidence -->

### 2.7 Tests
<!-- always include -->

### 2.8 Rollback
<!-- skip if no rollback considerations -->

## 3. Architecture Overview

### 3.1 System Context
...

### 3.2 Architecture Decisions
| Decision | Choice | Rationale | Evidence (id from 1.1/1.2) |
|---|---|---|---|

### 3.3 Module Boundaries
| Boundary | Owns | Depends On | Interface |
|---|---|---|---|

### 3.4 What Must Not Change
<!-- especially important for enhancement -->
...

## 4. Database Schema

### 4.1 New Models (mark NEW — not in current schema)
```
<!-- Use the project's actual schema format (Prisma schema, SQL DDL, SQLAlchemy model, GORM struct, ActiveRecord migration, etc.) -->
<!-- Example shape only — copy project's actual conventions -->
ModelName {
  id: primary_key
}
```

### 4.2 Existing Model Changes (must cite evidence id for current shape)
...

### 4.3 Migration Strategy
...

### 4.4 Data Preservation Rules
...

## 5. API Design

For each route:

### 5.X {METHOD} {path}

**Purpose:** ...  
**Auth:** ...  
**Auth reference:** {evidence id, file:line}  
**Request schema:**  
```{lang}
...
```

**Success response:**  
```{lang}
...
```

**Error responses:**  
```{lang}
...
```

**Side effects:**  
- ...

**Blast-radius notes:**  
- ...

**Reference implementation:** {evidence id, file:line — or "[UNVERIFIED — no reference found]"}

## 6. Component Design

For each component:

<!-- Section 6 applies only if the feature has a UI layer. Skip entirely for API-only, CLI, or backend-only features. -->
### 6.X {ComponentName} (mark NEW if not existing)

**Path:** ... (evidence id if existing)  
**Type:** Server | Client | Shared  
**Purpose:** ...  
**Props:**  
```{lang}
interface Props {
  ...
}
```

**Behavior:**  
- ...

**Existing pattern reference:** {evidence id, file:line}

## 7. Shared Utilities & Services

For each utility:

### 7.X `{functionName}` (mark NEW if not existing)

**Path:** ... (evidence id if existing)  
**Signature:**  
```{lang}
...
```

**Purpose:** ...  
**Called by:** ...  
**Evidence:** {evidence id, file:line}

## 8. Workflow, Events, and State

### 8.1 State Transitions
...

### 8.2 Side Effects
...

### 8.3 Idempotency / Retry Behavior
...

## 9. Integrations

| Integration | Function/API (exact) | Config | Failure Handling | Evidence id |
|---|---|---|---|---|

## 10. Security Design

- ...

## 11. Error Handling

| Scenario | HTTP Status | User Message | System Action |
|---|---|---|---|

## 12. Testing Requirements

### 12.1 Regression Tests
...

### 12.2 Unit Tests
...

### 12.3 Integration Tests
...

### 12.4 Contract Tests
<!-- enhancement only -->

### 12.5 Test Setup
...

## 13. Compatibility Layer
<!-- enhancement only -->

### 13.1 Frozen Contracts
| Contract | File | Dependents | Protection Test |
|---|---|---|---|

### 13.2 Transition Strategy
...

### 13.3 Rollback Plan
...

### 13.4 Cutover Criteria
...

## 14. Implementation Order

1. ...
2. ...

## 15. Environment & Config

| Variable | Purpose | Example | Required |
|---|---|---|---|

## 16. Drift Anchors

<!-- Content hashes for drift detection. Each hash = first 8 chars of SHA-256 of section body. -->
| Section | Title | Hash |
|---------|-------|------|
| 4 | Database Schema | `...` |
| 5 | API Design | `...` |
| ... | ... | `...` |
```
      </template>

      <pre-final-check priority="critical">
        Before saving, re-read sections 4-13 and verify every named file/symbol/route/field/prop appears in evidence_index (1.1/1.2) with matching file:line, or is [UNVERIFIED: name] in 1.3. Do not save a spec containing an unsourced existing-code claim.
      </pre-final-check>
    </phase>

  </flow>

  <control>
    <priority>blast-radius evidence before specification</priority>
    <failure>if missing evidence → research more; if still missing → mark [UNVERIFIED]</failure>
    <critical>
      <rule>Do not write PRD/spec before blast-radius gate passes.</rule>
      <rule>Do not silently assume compatibility safety.</rule>
      <rule>Do not omit security impact.</rule>
      <rule>Do not specify external/library APIs without verification.</rule>
      <rule priority="critical">External research (Context7 docs + octocode GitHub source + packageSearch version + WebSearch currency) is MANDATORY for every external symbol/dependency named in the spec — not optional, not skippable on "I know this". Confirm against the library's real source via octocode, not docs alone. Unverified external names are [UNVERIFIED]; unverified versions block the dependency from scope.</rule>
      <rule>For enhancements, frozen contracts must appear before implementation order.</rule>
      <rule>Never fabricate a file:line citation — a made-up citation is worse than none.</rule>
      <rule>Any name in prd.md/spec.md body text must resolve to an evidence entry; unresolved names become [UNVERIFIED], never silently "cleaned up" into a guessed real name.</rule>
      <rule>mcp__semble is mandatory for all blast-radius research. Call mcp__semble__search before every mcp__octocode__search or mcp__serena__find_symbol. Most token-efficient path.</rule>
    </critical>
  </control>

  <final>
    ## 📁 Saved
    PRD written to: `{folder}/prd.md`  
    Spec written to: `{folder}/spec.md`

    ## ▶️ Next Step
    /prd-plan {folder}/prd.md {folder}/spec.md
  </final>

</command>
