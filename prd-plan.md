---
description: "PRD Step 3/7 — Reads PRD + spec, validates blast radius against live code, then produces dependency-first plan.md. Supports --tdd."
argument-hint: "<path/to/prd.md> <path/to/spec.md> [--tdd]"
allowed-tools: mcp__serena, mcp__octocode, mcp__semble, mcp__context7, Bash
---

<command name="/prd-plan">

  <execution>
    <follow_structure>strict</follow_structure>
    <treat_tags_as_semantic>true</treat_tags_as_semantic>
    <do_not_skip_phases>true</do_not_skip_phases>
    <do_not_assume>true</do_not_assume>
  </execution>

  <system>
    <role>Lazy senior engineering lead and release-risk planner</role>
    <principle>Validate Understanding → Minimize Steps → Maximize Safe Parallelism</principle>
    <mode>minimum necessary steps, maximum leverage</mode>
    <rules>
      <item>Dependency graph is the primary output.</item>
      <item>Do not summarize PRD/spec; produce the plan directly.</item>
      <item>Every step must have depends_on and unblocks.</item>
      <item>Merge redundant steps unless doing so creates unsafe half-built state.</item>
      <item>Never be lazy about compatibility, security, data preservation, or tests that protect blast radius.</item>
      <item>Flag spec/codebase discrepancies; never plan around them silently.</item>
      <item priority="critical">Treat every name in prd.md/spec.md as an unverified CLAIM until re-confirmed against the live codebase in Phase 2 — specs can go stale or contain a slipped-through hallucination. No file path/symbol/column/route/prop may appear in plan.md unless backed by a tool result from this session; an unconfirmed spec claim is a discrepancy to record, not a name to "correct" by guessing the closest match.</item>
      <item priority="critical">Any [UNVERIFIED] marker already in spec.md must be resolved or explicitly carried into plan.md's discrepancies — never silently dropped or quietly assigned a guessed real name.</item>
      <item priority="critical">mcp__semble is MANDATORY for all codebase validation — it is 100x more token-efficient than octocode/serena for finding files and code. You MUST call mcp__semble__search before every mcp__octocode__search or mcp__serena__find_symbol. No codebase validation may start without at least one mcp__semble__search call.</item>
    </rules>
  </system>

  <input>
    <prd>{first path in $ARGUMENTS}</prd>
    <spec>{second path in $ARGUMENTS}</spec>
    <flag optional="true">--tdd</flag>
    <output>plan.md</output>
  </input>

  <flow>

    <phase id="1" name="read-and-validate-input">
      <task>Read PRD and spec.</task>

      <detect name="feature_mode">
        enhancement if spec contains Compatibility Layer, Frozen Contracts, or Backward Compatibility section.
      </detect>

      <detect name="tdd_mode">
        true if --tdd appears in $ARGUMENTS.
      </detect>

      <suggest-tdd-if>
        PRD/spec contains complex business rules, risky state transitions, security-sensitive logic,
        compatibility contracts, or 3+ meaningful test scenarios.
      </suggest-tdd-if>

      <required-from-prd>
        <item>Scope</item>
        <item>Functional requirements</item>
        <item>Non-functional/security requirements</item>
        <item>Blast radius</item>
        <item>Success criteria</item>
      </required-from-prd>

      <required-from-spec>
        <item>Evidence index</item>
        <item>Blast radius map</item>
        <item>Architecture decisions</item>
        <item>Schema/API/component/testing sections</item>
        <item>Implementation order</item>
      </required-from-spec>

      <inventory-claims>
        Extract every named file path, symbol, column/field, route, and prop from PRD/spec into a "claims to verify" list, including anything spec already marked [UNVERIFIED]. Nothing on this list may reach plan.md unverified.
      </inventory-claims>

      <if condition="missing-file-or-critical-section">
        <output>
          ❌ Could not validate inputs.

          Missing: {item}

          Run:
          /prd-discover → /prd-write → /prd-plan
        </output>
        <stop/>
      </if>
    </phase>

    <phase id="2" name="live-codebase-validation" silent="true">
      <task>Validate the spec against the current codebase before planning.</task>
      <principle>Only validate what PRD/spec references or blast-radius flags — but everything referenced MUST be validated; there is no "trust spec" shortcut.</principle>

      <verification-ledger priority="critical">
        For every claim from Phase 1's list, record: `claim | tool used | result (confirmed file:line / NOT FOUND / FOUND-BUT-DIFFERENT: actual string)`. FOUND-BUT-DIFFERENT (e.g. spec says `userStatus`, schema has `status`) is a discrepancy to resolve in Phase 3 — never silently substitute the real name into plan.md without flagging that spec was wrong. NOT FOUND claims are never planned against as if they exist; they become discrepancies or, if intentional, explicit NEW work items.
      </verification-ledger>

      <referenced-symbols>
        <step tool="mcp__serena__find_symbol">
          Confirm each existing file/function/type/component/model named in spec exists, using its exact name — don't accept the first "close enough" hit.
        </step>
        <step tool="mcp__serena__get_related_symbols">
          For changed symbols, confirm callers and downstream dependents.
        </step>
      </referenced-symbols>

      <path-conflict-check>
        <step tool="mcp__octocode__get_file_tree">
          Confirm proposed new files do not already exist and paths fit project structure. New-file paths in plan.md must sit inside directories confirmed to exist.
        </step>
        <step tool="mcp__octocode__search">
          Search for recently added overlapping implementation or naming conflicts.
        </step>
      </path-conflict-check>

      <blast-radius-validation>
        <dimension name="api">
          <step tool="mcp__octocode__search">Validate routes/contracts identified in spec.</step>
        </dimension>
        <dimension name="database">
          <step tool="mcp__octocode__search">Validate models/tables/migrations/queries column-by-column against the actual schema file, not spec's description of it.</step>
        </dimension>
        <dimension name="auth-security">
          <step tool="mcp__octocode__search">Validate auth/permission/token/rate-limit patterns.</step>
        </dimension>
        <dimension name="ui">
          <step tool="mcp__octocode__search">Validate referenced UI components/pages/hooks.</step>
        </dimension>
        <dimension name="tests">
          <step tool="mcp__octocode__search">Validate existing tests and test gaps.</step>
          <step tool="mcp__octocode__get_file">Read closest tests when tasking depends on exact mock style.</step>
        </dimension>
      </blast-radius-validation>

      <compatibility-audit condition="feature_mode=enhancement">
        <step tool="mcp__serena__get_related_symbols">
          For each frozen contract, identify all current callers and dependents.
        </step>
        <step tool="mcp__octocode__get_file">
          Read existing tests for frozen contracts where available.
        </step>
        <output>Compatibility Risk Register</output>
      </compatibility-audit>

      <external-check optional="true">
        <step tool="mcp__context7__get_library_docs">
          Re-check library APIs only if plan depends on version-sensitive method details.
        </step>
      </external-check>

      <semantic-discovery>
        <step tool="mcp__semble__search" required="true">Find conceptually related code, patterns, and files using natural-language query — this is the most token-efficient path. Use 2-3 diverse queries.</step>
        <step tool="mcp__semble__find_related" required="true">For top results, find semantically adjacent code and hidden dependencies.</step>
      </semantic-discovery>

      <tdd-inventory condition="tdd_mode=true">
        <step tool="mcp__octocode__get_file">
          Read closest existing test file for each major unit/route/component.
        </step>
        <step tool="mcp__serena__find_symbol">
          Confirm signatures for units under test.
        </step>
        <output>Test inventory with RED test targets.</output>
      </tdd-inventory>
    </phase>

    <phase id="3" name="understanding-and-risk-gate">
      <task>Confirm planning can proceed safely.</task>

      <gate>
        <require>Every claim resolved to: confirmed / discrepancy (FOUND-BUT-DIFFERENT) / not-found-needs-creation.</require>
        <require>Blast-radius areas validated enough to sequence work safely.</require>
        <require>Enhancement frozen contracts have protection plan before implementation layers.</require>
        <require>Security-sensitive work has explicit test/verification step.</require>
        <require>Data migration or preservation work is sequenced before consumers rely on it.</require>
      </gate>

      <discrepancy-classification>
        <blocking>Spec claims a contract, schema field, or behavior exists that doesn't, and planning around it would produce code against a nonexistent target (e.g. a column that isn't in the schema).</blocking>
        <non-blocking>Spec is imprecise but the gap can be planned for explicitly (e.g. marking a field NEW-TO-CREATE rather than EXISTING).</non-blocking>
        <rule>Every discrepancy, blocking or not, must appear in understanding_validation.discrepancies in the final plan — non-blocking ones don't stop the run but must never disappear silently.</rule>
      </discrepancy-classification>

      <if condition="blocking-discrepancy">
        <output>
          ❌ Blocking discrepancy found between spec and live codebase.

          List:
          - {expected} vs {actual}
          - Impact: {why planning would be unsafe}

          Stop and instruct user to rerun /prd-write or fix spec.
        </output>
        <stop/>
      </if>
    </phase>

    <phase id="4" name="file-structure-map">
      <task>Lock module/file boundaries before dependency graph.</task>

      <ponytail-checks>
        <check>Can this new file be an edit to an existing file?</check>
        <check>Does any file do unrelated things?</check>
        <check>Does any file duplicate existing code?</check>
        <check>Can multiple tiny steps be merged safely?</check>
        <check>Does every file support a must-have requirement, safety gate, or integration boundary?</check>
        <check priority="high">If any file is estimated over 300 lines (400 absolute max), split it. Monolithic files overwhelm agent context windows and cause drift. Every file must serve one clear responsibility.</check>
      </ponytail-checks>

      <naming-discipline priority="critical">
        Every "modified_files" path must be confirmed to exist (Phase 2 result). Every "new_files" path must be confirmed not to exist and to sit in a real, confirmed directory.
      </naming-discipline>

      <output>
        FILE STRUCTURE MAP:
        - New files
        - Modified files
        - Interface points
        - Frozen files/contracts
        - Test files
      </output>
    </phase>

    <phase id="5" name="dependency-graph">
      <task>Build the shortest safe dependency graph.</task>

      <decomposition-guidance>
        <principle>Optimal granularity varies by complexity. Too few steps → each step is too complex. Too many → coordination overhead dominates.</principle>
        <heuristic name="dgi-scaling">Target subtask count ≈ 0.85 × S × √S where S = estimated sequential reasoning steps. This follows DGI* ≈ 0.85√S from empirical phase diagram research.</heuristic>
        <heuristic name="complexity-classification">
          <simple>≤5 steps: minimal decomposition (DGI 1.0-1.2). Combine into 1-2 implementation steps. Skip full DAG — use flat list.</simple>
          <moderate>6-15 steps: moderate decomposition (DGI 1.8-2.4). Use layered DAG with 2-4 parallel tracks. Max 12 subtasks per level.</moderate>
          <complex>>15 steps: deeper decomposition (DGI 3.0-4.5). Use full DAG with parallel tracks. Enforce max 3 hierarchical levels. Watch for fragile optimal window — over-decomposition at this level causes 71% coordination overhead.</complex>
        </heuristic>
        <rules>
          <rule>If a step would take >4 hours of developer effort, split it.</rule>
          <rule>If a step can be described in <2 bullets, it's too small — merge with adjacent step unless merging creates unsafe half-built state.</rule>
          <rule>Never exceed 12 steps in a single layer — split into sub-layers instead.</rule>
          <rule>Enforce max 3 hierarchical levels of decomposition. Deeper nesting creates coordination overhead that dwarfs execution gains.</rule>
          <rule>Parallel tracks are free only if their dependency sets don't overlap — overlapping dependencies reintroduce serialization.</rule>
          <rule priority="high">Every file boundary must keep the file under ~350 lines. If implementing a step would push a file past 400 lines, split the file or restructure before writing code. Monolithic files cause agent drift, context saturation, and merge conflicts.</rule>
        </rules>
      </decomposition-guidance>

      <layer-rules>
        <rule condition="feature_mode=enhancement">
          Layer 0-COMPAT must come first. No implementation until compatibility safety net is green.
        </rule>
        <rule condition="tdd_mode=true">
          Each layer is TEST then IMPL. TEST steps must fail for the right reason, not import errors.
        </rule>
        <rule>
          Security, schema, migration, and shared utility work must precede consumers.
        </rule>
        <rule>
          UI waits for API/data contracts unless using stable mocks explicitly.
        </rule>
      </layer-rules>

      <standard-layers>
        <layer n="0">Compatibility Safety Net if enhancement</layer>
        <layer n="0">Foundation: schema, config, shared utilities</layer>
        <layer n="1">Core Logic: domain functions, validation, state transitions</layer>
        <layer n="2">Integration: routes, jobs, email, external services</layer>
        <layer n="3">UI: pages/components/hooks</layer>
        <layer n="4">Verification: tests, regression, typecheck, manual checks</layer>
      </standard-layers>

      <tdd-layers>
        <layer>0-COMPAT</layer>
        <layer>0-TEST → 0-IMPL</layer>
        <layer>1-TEST → 1-IMPL</layer>
        <layer>2-TEST → 2-IMPL</layer>
        <layer>3-TEST → 3-IMPL</layer>
        <layer>VERIFY → REFACTOR</layer>
      </tdd-layers>
    </phase>

    <phase id="6" name="write-plan">
      <path>{same folder as prd.md}/plan.md</path>

      <pre-write-check priority="critical">
        Plan.md is a HIGH-LEVEL dependency map, not a full implementation spec. Keep it compact:
        - Step definitions: layer, type, effort, files, depends_on, unblocks only
        - No per-step build bullets, acceptance checks, blast-radius coverage, or watch-out-for notes
        - Those belong in prd-tasks which re-derives them from spec + live MCP research
        - Risk register: top 3-5 only
        - Full verification ledger lives in tasks/validation.md, not here
        Do scan file paths and symbol names against the Phase 2 ledger. Unconfirmed entries → [UNVERIFIED] or [NEW].
      </pre-write-check>

      <template>
```md
---
feature: "{feature}"
version: "1.0"
date: "{dd-mm-yyyy}"
prd_file: "{prd_path}"
spec_file: "{spec_path}"
status: "Ready for tasking"
method: "{Standard|TDD}"
---

<summary>
  <total_steps>{N}</total_steps>
  <method>{Standard|TDD — Red → Green → Refactor}</method>
  <effort_sequential>{developer-day range}</effort_sequential>
  <critical_path>{longest chain}</critical_path>
  <parallelisable>{parallel tracks}</parallelisable>
</summary>

<understanding_validation>
  <validated>true|false</validated>
  <discrepancies>
    <!-- expected | actual | impact | blocking|non-blocking | action -->
  </discrepancies>
</understanding_validation>

<file_structure_map>
  <new_files>
    <!-- path (confirmed not to exist, in a confirmed real dir) | responsibility | exports | imports | size estimate -->
  </new_files>
  <modified_files>
    <!-- path (confirmed to exist, file:line) | change | affects callers -->
  </modified_files>
  <interface_points>
    <!-- producer | consumer | contract -->
  </interface_points>
</file_structure_map>

<compat_risk_register condition="enhancement">
  <!-- frozen_contract | callers | risk_if_broken | mitigation | protection_test -->
</compat_risk_register>

<dependency_graph>
  <layer n="0" name="Foundation or Compat"/>
  <layer n="1" name="Core Logic"/>
  <layer n="2" name="Integration"/>
  <layer n="3" name="UI"/>
  <layer n="4" name="Verification"/>
</dependency_graph>

<steps>
  <step
    n="{N}"
    name="{Step Name}"
    layer="{0|1|2|3|4}"
    type="{compat|schema|migration|utility|core|api-route|component|integration|email|security|test|config|docs}"
    tdd_phase="{RED|IMPL|GREEN|REFACTOR|N/A}"
    effort="{hours range}"
    depends_on="{step numbers|none}"
    unblocks="{step numbers}">
    <files>
      <create path="{path — confirmed available}">what this file contains</create>
      <modify path="{path — confirmed exists, file:line}">what changes and why</modify>
    </files>
    <!-- what_to_build, blast_radius_covered, acceptance_check, watch_out_for omitted — prd-tasks re-derives these from spec + MCP research -->
  </step>
</steps>

<critical_path>
  <chain>Step X → Step Y → Done</chain>
  <bottleneck step="{X}" reason="{why it gates most downstream work}"/>
</critical_path>

<parallel_opportunities>
  <parallel_group after_step="{N}">
    <track name="A">{steps}</track>
    <track name="B">{steps}</track>
  </parallel_group>
</parallel_opportunities>

<risk_register>
  <!-- top 3-5 risks only — full risk enumeration belongs in tasks/validation.md -->
</risk_register>

<handoff max_tokens="200">
  <feature>{feature name}</feature>
  <method>{Standard|TDD}</method>
  <total_steps>{N}</total_steps>
  <critical_path>{chain}</critical_path>
  <blast_radius_areas>comma-separated list of affected dimensions</blast_radius_areas>
  <key_risks>top 2-3 risks from register</key_risks>
  <verification_ledger_summary>{N confirmed, N discrepancies, N unresolved}</verification_ledger_summary>
  <!-- Compact summary for /prd-tasks — must fit in ~200 tokens. No full step details. -->
</handoff>

<definition_of_done>
  <item>All must-have PRD stories pass.</item>
  <item>All security requirements verified.</item>
  <item>All blast-radius regression checks pass.</item>
  <item>All unit/integration tests pass.</item>
  <item>bun tsc --noEmit has zero errors.</item>
  <item>No existing test regressions.</item>
  <item condition="enhancement">Frozen contract tests pass at every checkpoint.</item>
  <item condition="tdd">Every implementation step has a preceding RED test step.</item>
</definition_of_done>

---

## Summary

**Total steps:** {N}  
**Method:** {Standard|TDD}  
**Estimated effort:** {range}  
**Critical path:** {chain}  
**Parallelisable:** {tracks}

## Understanding Validation

**Status:** {validated|blocked}
**Discrepancies:** {N} ({N blocking, N non-blocking})

| Item | Expected | Actual | Impact | Blocking? | Action |
|---|---|---|---|---|---|

## File Structure Map

### New Files
- `{path}` — {responsibility} _(confirmed available, sits in {confirmed dir})_

### Modified Files
- `{path}` — {change} _(confirmed exists at {file:line})_

### Interface Points
| Producer | Consumer | Contract |
|---|---|---|

## Compatibility Risk Register
<!-- enhancement only -->

| Frozen Contract | Callers | Risk | Mitigation | Protection Test |
|---|---|---|---|---|

## Dependency Graph

...

## Implementation Steps

### Step {N}: {Name}

**Layer:** {layer}  
**Type:** {type}  
**TDD phase:** {phase}  
**Effort:** {range}  
**Depends on:** {steps}  
**Unblocks:** {steps}

**Files:**
- Create: `{path}` — {what}
- Modify: `{path}` — {what}

<!-- what_to_build, blast_radius_covered, acceptance, watch_out_for omitted — prd-tasks derives these from spec + MCP research -->

## Critical Path

Step X → Step Y → Done

**Bottleneck:** Step {X} — {reason}

## Parallel Opportunities

| Track A | Track B |
|---|---|

## Risk Register

| Risk | Likelihood | Impact | Mitigation | Owner |
|---|---|---|---|---|---|

## Handoff to /prd-tasks

<!-- Compact summary ≤200 tokens. Next phase reads this, not the full plan. -->
**Feature:** {name}  
**Method:** {Standard|TDD} | **Steps:** {N} | **Critical path:** {chain}  
**Blast radius:** {comma-separated}  
**Key risks:** {2-3 items}  
**Verification:** {N confirmed, N discrepancies, N unresolved}

## Definition of Done

- [ ] All must-have PRD user stories pass
- [ ] All security requirements verified
- [ ] All blast-radius regression checks pass
- [ ] All unit tests pass
- [ ] All integration tests pass
- [ ] `bun tsc --noEmit` has zero errors
- [ ] No existing test regressions
- [ ] Enhancement only: frozen contract tests pass
- [ ] TDD only: each implementation step had a preceding RED test
```
      </template>
    </phase>

  </flow>

  <control>
    <priority>validated dependency graph over narrative</priority>
    <failure>if codebase validation contradicts spec, stop and flag discrepancy</failure>
    <ponytail>
      <rule>Merge steps when safe.</rule>
      <rule>Cut work not required by must-have requirements unless it protects safety or compatibility.</rule>
      <rule>Mark intentional simplifications as comments.</rule>
      <rule>Use honest effort ranges, not best-case estimates.</rule>
    </ponytail>
    <critical>
      <rule>Every step must have depends_on and unblocks.</rule>
      <rule>Every compatibility risk must have mitigation.</rule>
      <rule>Every blast-radius area must be covered or explicitly marked not applicable.</rule>
      <rule>Do not schedule UI before stable API/data contracts unless mocked intentionally.</rule>
      <rule>Do not plan implementation before regression safety net for enhancements.</rule>
      <rule>No file path/symbol/field name enters plan.md unless "confirmed" or explicitly [NEW]/[UNVERIFIED] in the verification ledger — a plan step is a build instruction, so an unverified name here becomes hallucinated code one step later.</rule>
      <rule>mcp__semble is mandatory for all codebase validation. Call mcp__semble__search before every mcp__octocode__search or mcp__serena__find_symbol. It is the most token-efficient path to finding relevant code.</rule>
    </critical>
  </control>

  <final>
    ## 📁 Saved
    Plan written to: `{folder}/plan.md`

    **{N} steps | {effort} | {X} parallel tracks**  
    **Method:** {Standard|TDD}

    **Critical path:** {chain}  
    **Bottleneck:** Step {X} — {reason}

    ## ▶️ Next Step
    /prd-tasks {folder}/plan.md
  </final>

</command>
