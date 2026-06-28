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
      <principle>Only validate what the PRD/spec references or what the blast-radius map says may be affected.</principle>

      <referenced-symbols>
        <step tool="mcp__serena__find_symbol">
          Confirm each existing file/function/type/component/model named in spec exists.
        </step>
        <step tool="mcp__serena__get_related_symbols">
          For changed symbols, confirm callers and downstream dependents.
        </step>
      </referenced-symbols>

      <path-conflict-check>
        <step tool="mcp__octocode__get_file_tree">
          Confirm proposed new files do not already exist and paths fit project structure.
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
          <step tool="mcp__octocode__search">Validate models/tables/migrations/queries identified in spec.</step>
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

      <complexity-calibration optional="true">
        <step tool="mcp__semble__find_related">
          Use only if effort/sequence is unclear from spec and codebase.
        </step>
      </complexity-calibration>

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
        <require>All spec-referenced existing symbols either verified or flagged as discrepancy.</require>
        <require>Blast-radius areas validated enough to sequence work safely.</require>
        <require>Enhancement frozen contracts have protection plan before implementation layers.</require>
        <require>Security-sensitive work has explicit test/verification step.</require>
        <require>Data migration or preservation work is sequenced before consumers rely on it.</require>
      </gate>

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
        <check>If any file is estimated over 300 lines, split it.</check>
      </ponytail-checks>

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
    <!-- expected | actual | impact | action -->
  </discrepancies>
</understanding_validation>

<blast_radius_plan>
  <api/>
  <database/>
  <auth_security/>
  <ui/>
  <workflow_events/>
  <integrations/>
  <tests/>
  <rollback/>
</blast_radius_plan>

<file_structure_map>
  <new_files>
    <!-- path | responsibility | exports | imports | size estimate -->
  </new_files>
  <modified_files>
    <!-- path | change | affects callers -->
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
      <create path="{path}">what this file contains</create>
      <modify path="{path}">what changes and why</modify>
    </files>
    <what_to_build>
      <!-- 3-8 concrete bullets -->
    </what_to_build>
    <blast_radius_covered>
      <!-- api/db/auth/ui/workflow/integration/test/rollback -->
    </blast_radius_covered>
    <acceptance_check>{exact command or manual check}</acceptance_check>
    <watch_out_for>
      <!-- codebase-specific gotchas -->
    </watch_out_for>
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
  <!-- risk | likelihood | impact | mitigation | owner -->
</risk_register>

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

| Item | Expected | Actual | Impact | Action |
|---|---|---|---|---|

## Blast Radius Plan

### API
...

### Database
...

### Auth/Security
...

### UI
...

### Workflow/Events
...

### Integrations
...

### Tests
...

### Rollback
...

## File Structure Map

### New Files
- `{path}` — {responsibility}

### Modified Files
- `{path}` — {change}

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

**What to build:**
- ...

**Blast radius covered:**
- ...

**TDD test spec:**
<!-- only for TEST steps -->
```ts
describe('{unit}', () => {
  it('{behavior}', () => {
    // given: {setup}
    // when: {action}
    // then: {assertion}
  })
})
```

**Acceptance check:**  
`{command}`

**Watch out for:**
- ...

## Critical Path

Step X → Step Y → Done

**Bottleneck:** Step {X} — {reason}

## Parallel Opportunities

| Track A | Track B |
|---|---|

## Risk Register

| Risk | Likelihood | Impact | Mitigation | Owner |
|---|---|---|---|---|

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
///
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