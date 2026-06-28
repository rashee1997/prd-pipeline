---
description: "PRD Step 2/7 — Reads discovery.md, performs blast-radius research, validates codebase + external patterns, then writes prd.md and spec.md. No assumptions."
argument-hint: "<path to discovery.md>"
allowed-tools: mcp__serena, mcp__octocode, mcp__semble, mcp__context7, Bash
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
        PRD and spec must be written from verified evidence, not assumptions.
      </evidence-principle>

      <project-map>
        <step tool="mcp__serena__get_overview">
          Record project structure, modules, routes, DB layer, auth, test layout, shared libs.
        </step>
      </project-map>

      <discovery-symbol-resolution repeat="for each file, symbol, route, table, model, component, or service mentioned in discovery">
        <step tool="mcp__serena__find_symbol">Locate exact symbol or closest owning symbol.</step>
        <step tool="mcp__serena__get_symbol_info">Record signature, fields, props, return type, path, comments.</step>
        <step tool="mcp__serena__get_related_symbols">Record callers, callees, imports, dependents.</step>
      </discovery-symbol-resolution>

      <blast-radius-dimensions>
        <dimension name="api">
          <search tool="mcp__octocode__search">route handlers, endpoints, request/response types near feature domain</search>
          <read tool="mcp__octocode__get_file" count="2-5">closest route files</read>
          <extract>routes touched, contract shapes, auth, status codes, response conventions, consumers</extract>
        </dimension>

        <dimension name="database">
          <search tool="mcp__octocode__search">db/prisma usage for relevant models and adjacent models</search>
          <read tool="mcp__octocode__get_file" count="2-5">query files and schema/model files</read>
          <extract>models, relations, required fields, indexes, migrations, destructive-risk areas</extract>
        </dimension>

        <dimension name="auth-security">
          <search tool="mcp__octocode__search">auth(), permissions, role checks, token validation, public routes</search>
          <read tool="mcp__octocode__get_file" count="2-5">auth-protected and public route examples</read>
          <extract>auth pattern, permission model, unauth response, public access safeguards, rate-limit patterns</extract>
        </dimension>

        <dimension name="ui">
          <search tool="mcp__octocode__search">pages/components/hooks related to the feature domain</search>
          <read tool="mcp__octocode__get_file" count="2-5">relevant UI components/pages</read>
          <extract>component boundaries, props, data fetching, state, design system conventions</extract>
        </dimension>

        <dimension name="workflow-events">
          <search tool="mcp__octocode__search">workflow, status, state transition, event, notification, job, queue terms</search>
          <read tool="mcp__octocode__get_file" count="2-5">workflow/event/job files</read>
          <extract>state machine rules, side effects, jobs, notifications, status constraints</extract>
        </dimension>

        <dimension name="integrations">
          <search tool="mcp__octocode__search">email, provider, storage, webhook, external API, upload, export terms</search>
          <read tool="mcp__octocode__get_file" count="1-4">integration files</read>
          <extract>integration APIs, error handling, retry behavior, configuration</extract>
        </dimension>

        <dimension name="tests">
          <search tool="mcp__octocode__search">existing tests around affected modules and frozen contracts</search>
          <read tool="mcp__octocode__get_file" count="2-5">closest test files</read>
          <extract>mocking style, test setup, regression gaps, contract tests needed</extract>
        </dimension>
      </blast-radius-dimensions>

      <semantic-research>
        <step tool="mcp__semble__search">
          Search conceptual equivalents that may be named differently.
        </step>
        <step tool="mcp__semble__find_related">
          Find adjacent implementations and hidden dependencies.
        </step>
      </semantic-research>

      <external-research>
        <task>Validate non-trivial implementation approaches before specifying them.</task>

        <libraries repeat="for each library/API that spec may use">
          <step tool="mcp__context7__resolve_library_id">Resolve exact library.</step>
          <step tool="mcp__context7__get_library_docs">Fetch targeted API docs only.</step>
          <record>library, version if found, method signatures, constraints, gotchas</record>
        </libraries>

        <github repeat="for each non-trivial pattern">
          <step tool="mcp__octocode__github_search">
            Find real implementations with same framework/language/domain where possible.
          </step>
          <record>repo/file, pattern confirmed, edge cases, anti-patterns, spec section supported</record>
        </github>
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
        <require>API, DB, auth/security, UI, workflow, integration, and test impacts assessed.</require>
        <require>External patterns verified for non-trivial design choices.</require>
        <require>Enhancement frozen contracts and callers identified before PRD/spec writing.</require>
        <require>Known unknowns listed explicitly.</require>
      </blast-radius-gate>
    </phase>

    <phase id="3" name="write-prd">
      <path>{same folder as discovery.md}/prd.md</path>

      <rules>
        <rule>PRD is a contract: testable, scoped, and grounded.</rule>
        <rule>Each requirement must include acceptance criteria.</rule>
        <rule>Requirements must reflect blast-radius findings.</rule>
        <rule>Out-of-scope and deferred items must be explicit.</rule>
        <rule>Security and compatibility requirements must not be generic; they must be feature-specific.</rule>
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
  <codebase_evidence><!-- files/symbols/patterns used --></codebase_evidence>
  <external_evidence><!-- docs/github patterns used --></external_evidence>
  <known_unknowns><!-- explicitly unresolved items --></known_unknowns>
</evidence_summary>

<blast_radius>
  <api/>
  <database/>
  <auth_security/>
  <ui/>
  <workflow_events/>
  <integrations/>
  <tests/>
  <rollback/>
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
| Evidence | Source | Used For |
|---|---|---|

### 2.3 External Evidence Used
| Pattern/API | Source | Used For |
|---|---|---|

### 2.4 Known Unknowns
- ...

## 3. Blast Radius

### 3.1 API Impact
...

### 3.2 Database/Data Impact
...

### 3.3 Auth & Security Impact
...

### 3.4 UI Impact
...

### 3.5 Workflow/Event Impact
...

### 3.6 Integration Impact
...

### 3.7 Test/Regression Impact
...

### 3.8 Rollback Impact
...

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

| ID | Requirement | Acceptance Criteria | Priority | Evidence |
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
///
      </template>
    </phase>

    <phase id="4" name="write-spec">
      <path>{same folder as discovery.md}/spec.md</path>

      <rules>
        <rule>Every existing file path must be real.</rule>
        <rule>Every existing symbol must be verified.</rule>
        <rule>Every new file must be justified by module boundaries.</rule>
        <rule>Every route must cite a reference implementation.</rule>
        <rule>Every schema/query must cite existing DB pattern or verified docs.</rule>
        <rule>Every section must be implementable without reading other sections.</rule>
        <rule>Enhancement specs must begin with compatibility constraints.</rule>
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
    <!-- id | file | symbol | lines/notes | supports -->
  </codebase_sources>
  <external_sources>
    <!-- id | library/repo | topic/file | supports -->
  </external_sources>
  <unverified_items>
    <!-- item | why unresolved | required before implementation -->
  </unverified_items>
</evidence_index>

<blast_radius_map>
  <api/>
  <database/>
  <auth_security/>
  <ui/>
  <workflow_events/>
  <integrations/>
  <tests/>
  <rollback/>
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
<environment/>

---

## 1. Evidence Index

### 1.1 Codebase Sources
| ID | File | Symbol/Area | Supports |
|---|---|---|---|

### 1.2 External Sources
| ID | Library/Repo | Topic/File | Supports |
|---|---|---|---|

### 1.3 Unverified Items
| Item | Risk | Required Before |
|---|---|---|

## 2. Blast Radius Map

### 2.1 API
...

### 2.2 Database
...

### 2.3 Auth/Security
...

### 2.4 UI
...

### 2.5 Workflow/Events
...

### 2.6 Integrations
...

### 2.7 Tests
...

### 2.8 Rollback
...

## 3. Architecture Overview

### 3.1 System Context
...

### 3.2 Architecture Decisions
| Decision | Choice | Rationale | Evidence |
|---|---|---|---|

### 3.3 Module Boundaries
| Boundary | Owns | Depends On | Interface |
|---|---|---|---|

### 3.4 What Must Not Change
<!-- especially important for enhancement -->
...

## 4. Database Schema

### 4.1 New Models
```prisma
model Example {
  id String @id
}
```

### 4.2 Existing Model Changes
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
**Auth reference:** {existing file}  
**Request schema:**  
```ts
...
```

**Success response:**  
```ts
...
```

**Error responses:**  
```ts
...
```

**Side effects:**  
- ...

**Blast-radius notes:**  
- ...

**Reference implementation:** {existing route file}

## 6. Component Design

For each component:

### 6.X {ComponentName}

**Path:** ...  
**Type:** Server | Client | Shared  
**Purpose:** ...  
**Props:**  
```ts
interface Props {
  ...
}
```

**Behavior:**  
- ...

**Existing pattern reference:** ...

## 7. Shared Utilities & Services

For each utility:

### 7.X `{functionName}`

**Path:** ...  
**Signature:**  
```ts
...
```

**Purpose:** ...  
**Called by:** ...  
**Evidence:** ...

## 8. Workflow, Events, and State

### 8.1 State Transitions
...

### 8.2 Side Effects
...

### 8.3 Idempotency / Retry Behavior
...

## 9. Integrations

| Integration | Function/API | Config | Failure Handling | Evidence |
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
```
///
      </template>
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
      <rule>For enhancements, frozen contracts must appear before implementation order.</rule>
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