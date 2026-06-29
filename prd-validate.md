---
description: "PRD Step 4a/7 — Validate generated task files before implementation. Research-backed gate for task context, dependencies, paths, acceptance checks, blast radius, compatibility, and parallel safety."
argument-hint: "<path/to/tasks/index.md | path/to/tasks/>"
allowed-tools: mcp__serena, mcp__octocode, mcp__semble, mcp__context7, Bash
---

<command name="/prd-validate">

  <execution>
    <follow_structure>strict</follow_structure>
    <treat_tags_as_semantic>true</treat_tags_as_semantic>
    <do_not_skip_phases>true</do_not_skip_phases>
    <do_not_assume>true</do_not_assume>
    <prefer_mcp_for_research>true</prefer_mcp_for_research>
    <do_not_implement_code>true</do_not_implement_code>
  </execution>

  <system>
    <role>Senior implementation-readiness reviewer</role>
    <principle>Validate tasks before implementation</principle>
    <mode>research-backed task validation gate</mode>
    <rules>
      <item>Do not implement product code.</item>
      <item>Do not mark tasks complete.</item>
      <item>Do not update task progress.</item>
      <item>Validate task files against plan, PRD, spec, and live codebase.</item>
      <item>Every task must be executable by a fresh subagent with no session history.</item>
      <item>Every dependency must be a task ID, not a document reference.</item>
      <item>Every acceptance check must be concrete and runnable.</item>
      <item>Every file path must be real, proposed-new, or explicitly justified.</item>
      <item>Blocking issue count controls final status and readiness.</item>
    </rules>
  </system>

  <input>
    <tasks>{$ARGUMENTS}</tasks>
    <accepted_forms>
      <form>path/to/tasks/index.md</form>
      <form>path/to/tasks/</form>
    </accepted_forms>
    <output>{prd-folder}/tasks/validation.md</output>
  </input>

  <mcp_policy>
    <principle>MCP tools are the primary research layer for codebase validation.</principle>

    <use_mcp_for>
      <item>symbol lookup</item>
      <item>file existence and content validation when available</item>
      <item>semantic code discovery</item>
      <item>library/API verification</item>
      <item>PR or changed-file context when supported</item>
    </use_mcp_for>

    <use_bash_for>
      <item>reading generated PRD artifacts</item>
      <item>checking local file existence</item>
      <item>checking package scripts</item>
      <item>targeted grep only after MCP evidence or for generated docs</item>
    </use_bash_for>

    <anti_pattern>
      <item>Do not replace MCP code research with broad grep/find scans.</item>
      <item>Do not infer code structure from filenames alone.</item>
      <item>Do not invent unavailable MCP tools.</item>
      <item>Do not use Bash to inspect implementation files when MCP file tools are available.</item>
    </anti_pattern>

    <fallback>
      If MCP tools are unavailable or insufficient, state that clearly in validation notes and use Bash for targeted verification.
    </fallback>
  </mcp_policy>

  <flow>

    <phase id="1" name="resolve-task-set">
      <task>Find tasks/index.md and all task files.</task>

      <resolve>
        <case type="index-path">Use the parent directory as the tasks folder.</case>
        <case type="tasks-directory">Use tasks/index.md inside that directory.</case>
      </resolve>

      <read>
        tasks/index.md
        all tasks/TASK-*.md
      </read>

      <extract>
        <item>feature name</item>
        <item>plan path</item>
        <item>task IDs</item>
        <item>task layers</item>
        <item>task statuses</item>
        <item>dependencies</item>
        <item>unblocks</item>
        <item>parallel-safe claims</item>
        <item>files to create</item>
        <item>files to modify</item>
        <item>files to delete</item>
        <item>acceptance commands</item>
        <item>commit blocks</item>
        <item>compat-sensitive flags</item>
      </extract>

      <if condition="missing-index-or-task-files">
        <output>
          ❌ Task set incomplete.

          Expected:
          - tasks/index.md
          - tasks/TASK-*.md

          Run:
          /prd-tasks {path/to/plan.md}
        </output>
        <stop/>
      </if>
    </phase>

    <phase id="2" name="load-source-documents">
      <task>Load the source documents needed to validate the task set.</task>

      <read>
        plan.md from index header
        prd.md from plan frontmatter
        spec.md from plan frontmatter
      </read>

      <validate>
        <check>Plan file exists.</check>
        <check>PRD file exists.</check>
        <check>Spec file exists.</check>
        <check>Plan contains dependency graph or implementation steps.</check>
        <check>Spec contains architecture, API, schema, testing, or blast-radius sections.</check>
      </validate>

      <extract>
        <item>plan total steps</item>
        <item>plan dependency graph</item>
        <item>plan blast-radius plan</item>
        <item>plan compatibility risk register</item>
        <item>PRD functional requirements</item>
        <item>PRD definition of done</item>
        <item>spec frozen contracts</item>
        <item>spec file paths</item>
        <item>spec API routes</item>
        <item>spec schema changes</item>
        <item>spec testing requirements</item>
      </extract>

      <if condition="missing-source-document">
        <output>
          ❌ Cannot validate tasks because a source document is missing:
          - {missing path}

          Re-run:
          /prd-write → /prd-plan → /prd-tasks
        </output>
        <stop/>
      </if>
    </phase>

    <phase id="3" name="static-task-validation">
      <task>Validate every task file without live codebase calls first.</task>

      <per-task-checks>
        <check name="identity">Filename, title, metadata, and XML task ID match.</check>
        <check name="status">Task status exists and matches index state.</check>
        <check name="context">Context is self-contained and task-specific.</check>
        <check name="context-size">Context is roughly 300-600 words unless task is intentionally simple.</check>
        <check name="role">Role is specialized and task-specific.</check>
        <check name="dependencies">Depends on contains task IDs only.</check>
        <check name="unblocks">Unblocks references real task IDs or none.</check>
        <check name="parallel">Parallel-safe claims include no overlapping file sets.</check>
        <check name="files">Files to create, modify, or delete are explicit.</check>
        <check name="acceptance">Acceptance command is exact and proves the task outcome.</check>
        <check name="commit">Commit block uses explicit file paths only.</check>
        <check name="done_signal">DONE, DONE_WITH_CONCERNS, NEEDS_CONTEXT, BLOCKED are present where applicable.</check>
      </per-task-checks>

      <blocking-if>
        <condition>Task says "see plan.md", "see spec.md", "see PRD", or requires reading another task before starting.</condition>
        <condition>Acceptance command is missing, vague, or cannot prove the task outcome.</condition>
        <condition>Commit block uses git add . or git add -A.</condition>
        <condition>Task says CREATE for a file that already exists.</condition>
        <condition>Task says MODIFY for a file that does not exist without explicitly creating it first.</condition>
        <condition>Task deletes a file without rollback or dependency check.</condition>
        <condition>Task touches a frozen contract without compat-sensitive flag and regression acceptance check.</condition>
        <condition>Task has unrelated concerns that should be split.</condition>
      </blocking-if>

      <warning-if>
        <condition>Dependency is stricter than necessary but safe.</condition>
        <condition>Context is slightly long but still executable.</condition>
        <condition>Acceptance command is valid but could be more scoped.</condition>
        <condition>Parallel-safe claim is absent but task could be parallelized.</condition>
      </warning-if>
    </phase>

    <phase id="4" name="dependency-graph-validation">
      <task>Validate task graph against the plan dependency graph.</task>

      <checks>
        <check>Every task maps to one or more plan steps.</check>
        <check>Every plan step is covered by one or more task files, or intentionally bundled.</check>
        <check>No dependency references a missing task.</check>
        <check>No circular dependencies exist.</check>
        <check>Layer ordering is respected.</check>
        <check>Same-layer parallel-safe tasks do not modify the same files.</check>
        <check>Tasks with overlapping file sets are sequential unless explicitly justified.</check>
      </checks>

      <allow>
        <item>Multiple plan steps may be bundled into one verification task if the task is self-contained.</item>
        <item>One plan step may split into multiple tasks when files, acceptance checks, or concerns differ.</item>
        <item>Over-constrained dependencies are warnings, not blockers, if they are safe and acyclic.</item>
      </allow>
    </phase>

    <phase id="5" name="compatibility-validation" condition="enhancement-or-frozen-contracts-present">
      <task>Validate enhancement safety before implementation.</task>

      <checks>
        <check>Compatibility gate task exists in earliest layer.</check>
        <check>Frozen contracts are listed in tasks or validation notes.</check>
        <check>Frozen contract regression tests are present before implementation tasks.</check>
        <check>Layer 1+ tasks depend on compatibility gate when they may touch frozen areas.</check>
        <check>Compat-sensitive tasks include frozen contract acceptance checks.</check>
        <check>Cutover task is final, human-gated, and not parallel-safe.</check>
        <check>Backward-compatible response fields are preserved when existing UI/API callers depend on them.</check>
      </checks>

      <blocking-if>
        <condition>No compatibility gate exists for an enhancement.</condition>
        <condition>A frozen contract can change before regression tests exist.</condition>
        <condition>Cutover is auto-executable or not final.</condition>
        <condition>Legacy response fields are removed before caller migration is verified.</condition>
      </blocking-if>
    </phase>

    <phase id="6" name="research-backed-codebase-validation" critical="true">
      <task>Validate task instructions against the live codebase.</task>
      <principle>Validate only what tasks reference or what blast-radius sections say may be affected.</principle>

      <referenced-symbols>
        <step tool="mcp__serena__find_symbol">
          Confirm existing functions, types, components, handlers, and exported symbols named in tasks.
        </step>
        <step tool="mcp__serena__get_symbol_info">
          Confirm signatures embedded in task context still match live code.
        </step>
        <step tool="mcp__serena__get_related_symbols">
          For frozen contracts or deleted symbols, confirm callers and downstream dependencies.
        </step>
      </referenced-symbols>

      <referenced-files>
        <step tool="mcp__octocode__get_file">
          Read files each task modifies or deletes to confirm existence, insertion points, and current snippets.
        </step>
      </referenced-files>

      <file-path-checks>
        <check>Existing files named under "modify" or "delete" exist.</check>
        <check>Files named under "create" do not already exist.</check>
        <check>If a create path exists, mark blocking and require MODIFY/APPEND wording.</check>
        <check>If a modify path does not exist, mark blocking unless the task creates it first.</check>
      </file-path-checks>

      <pattern-validation>
        <step tool="mcp__octocode__search">
          Verify referenced local patterns exist, such as auth helpers, DB imports, test mock patterns, response shapes, or UI toast patterns.
        </step>
      </pattern-validation>

      <semantic-validation optional="true">
        <step tool="mcp__semble__find_related">
          Use only if task references conceptual behavior with uncertain naming.
        </step>
      </semantic-validation>

      <library-validation optional="true">
        <step tool="mcp__context7__get_library_docs">
          Verify library APIs only when task acceptance or implementation depends on exact external API behavior.
        </step>
      </library-validation>

      <bash-targeted-checks>
        <step>Bash: test -e {existing file paths}</step>
        <step>Bash: test ! -e {new file paths}</step>
        <step>Bash: inspect package scripts only when acceptance commands reference them</step>
      </bash-targeted-checks>

      <blocking-if>
        <condition>A task's embedded signature, schema field/column, or symbol does not match what MCP/live-code lookup confirms (e.g. a column name in a task that the actual schema doesn't have).</condition>
        <condition>A task references a database field, route, or component name that returns no match in the live codebase.</condition>
      </blocking-if>
    </phase>

    <phase id="7" name="acceptance-command-validation">
      <task>Validate that task acceptance checks are concrete and likely runnable.</task>

      <checks>
        <check>Each task has one primary acceptance command unless multiple are explicitly justified.</check>
        <check>Commands use known project tooling or scripts.</check>
        <check>TypeScript check exists where source code changes.</check>
        <check>Test command is scoped enough to prove task behavior.</check>
        <check>Compat-sensitive tasks include frozen contract regression command.</check>
        <check>Manual checks are allowed only for UI/manual verification tasks and must be specific.</check>
      </checks>

      <blocking-if>
        <condition>Acceptance says "verify it works" without command or explicit manual steps.</condition>
        <condition>Acceptance command references a missing script.</condition>
        <condition>Acceptance cannot detect failure of the task's core behavior.</condition>
      </blocking-if>

      <allowed-command-examples>
        <example>bun tsc --noEmit</example>
        <example>npx tsc --noEmit</example>
        <example>bun test</example>
        <example>npx jest path/to/test.ts --no-coverage</example>
        <example>bun run build</example>
        <example>bun run lint</example>
        <example>grep -r "{symbol}" src/</example>
      </allowed-command-examples>
    </phase>

    <phase id="8" name="blast-radius-validation">
      <task>Confirm task coverage matches the blast-radius plan.</task>

      <dimensions>
        <dimension name="api">Routes added, modified, frozen, or response-compatible.</dimension>
        <dimension name="database">Schema, seed, migration, persistence, destructive changes.</dimension>
        <dimension name="auth_security">Permissions, auth checks, public/private surfaces, tokens, PII.</dimension>
        <dimension name="ui">Components, pages, handlers, visible behavior, existing UI contracts.</dimension>
        <dimension name="workflow_events">Jobs, queues, events, state transitions.</dimension>
        <dimension name="integrations">External APIs, tools, generated artifacts, reference loaders.</dimension>
        <dimension name="tests">Regression, unit, integration, contract, manual checks.</dimension>
        <dimension name="rollback">Concrete rollback steps and feasibility.</dimension>
      </dimensions>

      <checks>
        <check>Every blast-radius dimension is covered by at least one task or marked not applicable.</check>
        <check>Security-sensitive dimensions have acceptance checks.</check>
        <check>Database destructive changes have rollback or dev-only constraints.</check>
        <check>Compatibility risks have regression tests or explicit verification tasks.</check>
        <check>UI compatibility risks include caller/handler verification if response shapes change.</check>
      </checks>
    </phase>

    <phase id="9" name="status-resolution">
      <task>Resolve final validation status from collected findings.</task>

      <status_logic>
        <rule>If unresolved blocking issue count is greater than 0, status = BLOCKED.</rule>
        <rule>If unresolved blocking issue count equals 0, status = VALIDATED.</rule>
        <rule>If status = VALIDATED, implementation_readiness.ready = true.</rule>
        <rule>If status = BLOCKED, implementation_readiness.ready = false.</rule>
        <rule>Warnings do not block implementation unless explicitly promoted to blocking.</rule>
        <rule>Fixed issues must not appear under unresolved blocking issues.</rule>
        <rule>Required Fixes Before Implementation must be "None" when status = VALIDATED.</rule>
      </status_logic>

      <consistency_gate>
        <require>Frontmatter status equals validation_summary/status.</require>
        <require>validation_summary/blocking_issues equals unresolved blocking issue count.</require>
        <require>implementation_readiness/ready matches status.</require>
        <require>Markdown "Validation Summary" matches XML summary.</require>
        <require>Markdown "Implementation Readiness" matches XML readiness.</require>
        <require>Markdown "Required Fixes Before Implementation" is "None" when status is VALIDATED.</require>
        <require>Fixed issues are listed only in Fixed Issues, never in Required Fixes.</require>
      </consistency_gate>

      <if condition="consistency-gate-fails">
        <action>Fix the validation report before writing final output.</action>
      </if>
    </phase>

    <phase id="10" name="write-validation-report">
      <path>{prd-folder}/tasks/validation.md</path>

      <template>
```md
---
feature: "{feature}"
date: "{dd-mm-yyyy}"
status: "{VALIDATED|BLOCKED}"
tasks_index: "{path/to/tasks/index.md}"
plan_file: "{path/to/plan.md}"
prd_file: "{path/to/prd.md}"
spec_file: "{path/to/spec.md}"
---

<validation_summary>
  <status>{VALIDATED|BLOCKED}</status>
  <tasks_checked>{N}</tasks_checked>
  <blocking_issues>{unresolved_blocking_count}</blocking_issues>
  <warnings>{warning_count}</warnings>
  <parallel_safety>{valid|invalid}</parallel_safety>
  <compatibility_gate>{present|not_applicable|missing}</compatibility_gate>
</validation_summary>

<fixed_issues>
  <!-- Fixed issues only. Do not include unresolved blockers here. -->
  <issue id="{id}" task="{TASK-ID}" severity="fixed">
    <original_issue>{original problem}</original_issue>
    <resolution>{how it was fixed}</resolution>
  </issue>
</fixed_issues>

<blocking_issues>
  <!-- Unresolved blocking issues only. Empty when status=VALIDATED. -->
  <issue id="{id}" task="{TASK-ID}" severity="blocking">
    <description>{problem}</description>
    <required_fix>{fix required before implementation}</required_fix>
  </issue>
</blocking_issues>

<warnings>
  <warning id="{id}" task="{TASK-ID}" severity="{low|medium}">
    <description>{warning}</description>
    <suggested_fix>{suggested fix or no fix needed}</suggested_fix>
  </warning>
</warnings>

<task_validation>
  <task id="{TASK-ID}">
    <status>{pass|fail|warning}</status>
    <context_isolated>{true|false}</context_isolated>
    <dependencies_valid>{true|false}</dependencies_valid>
    <file_paths_valid>{true|false}</file_paths_valid>
    <acceptance_valid>{true|false}</acceptance_valid>
    <parallel_safe>{true|false|n/a}</parallel_safe>
    <notes>{evidence-backed notes}</notes>
  </task>
</task_validation>

<dependency_validation>
  <graph_status>{VALID|BLOCKED|VALID_WITH_WARNINGS}</graph_status>
  <missing_deps>{count}</missing_deps>
  <cycles>{count}</cycles>
  <layer_issues>{count}</layer_issues>
  <notes>
    <!-- dependency notes -->
  </notes>
</dependency_validation>

<blast_radius_validation>
  <api>{status and notes}</api>
  <database>{status and notes}</database>
  <auth_security>{status and notes}</auth_security>
  <ui>{status and notes}</ui>
  <workflow_events>{status and notes}</workflow_events>
  <integrations>{status and notes}</integrations>
  <tests>{status and notes}</tests>
  <rollback>{status and notes}</rollback>
</blast_radius_validation>

<implementation_readiness>
  <ready>{true|false}</ready>
  <start_task>{TASK-ID or none}</start_task>
  <safe_parallel_layers>
    <layer name="{layer name}">
      <tasks>{TASK IDs}</tasks>
      <note>{why safe}</note>
    </layer>
  </safe_parallel_layers>
</implementation_readiness>

---

## Validation Summary

**Status:** {✅ VALIDATED|❌ BLOCKED}  
**Tasks checked:** {N}  
**Blocking issues:** {unresolved_blocking_count}  
**Warnings:** {warning_count}  
**Parallel safety:** {✅ Valid|❌ Invalid}  
**Compatibility gate:** {✅ Present|Not applicable|❌ Missing}

## Fixed Issues

<!-- If none, write exactly: None. -->

{fixed_issues_table_or_None}

## Blocking Issues

<!-- If none, write exactly: None. -->

{blocking_issues_table_or_None}

## Warnings

<!-- If none, write exactly: None. -->

{warnings_table_or_None}

## Task Validation

| Task | Context | Deps | Files | Acceptance | Parallel | Status |
|---|---|---|---|---|---|---|
| {TASK-ID} | {✅|❌} | {✅|❌|⚠️} | {✅|❌|⚠️} | {✅|❌} | {✅|n/a|❌} | {✅ pass|❌ fail|⚠️ warning} |

## Dependency Graph Validation

**Graph status:** {VALID|BLOCKED|VALID with warnings}  
**Missing dependencies:** {count}  
**Circular dependencies:** {count}  
**Layer ordering:** {Correct|Incorrect}

{notes}

## Blast Radius Validation

### API
{notes}

### Database
{notes}

### Auth/Security
{notes}

### UI
{notes}

### Workflow/Events
{notes}

### Integrations
{notes}

### Tests
{notes}

### Rollback
{notes}

## Implementation Readiness

<!-- This section MUST match <implementation_readiness>. -->

**Ready:** {✅ Yes|❌ No}

{if_ready_start_commands}

{if_blocked_rerun_commands}

## Required Fixes Before Implementation

<!-- If status=VALIDATED, write exactly: None. -->
<!-- If status=BLOCKED, list unresolved blocking issues only. Do not list fixed issues here. -->

{required_fixes_or_None}
```
      </template>

      <render_rules>
        <rule>If status=VALIDATED, render "Blocking Issues" as "None."</rule>
        <rule>If status=VALIDATED, render "Required Fixes Before Implementation" as "None."</rule>
        <rule>If status=VALIDATED, render Ready as "✅ Yes".</rule>
        <rule>If status=VALIDATED, include start implementation command.</rule>
        <rule>If status=BLOCKED, render Ready as "❌ No".</rule>
        <rule>If status=BLOCKED, include rerun validation command.</rule>
        <rule>Never include fixed issues in Required Fixes Before Implementation.</rule>
      </render_rules>
    </phase>

  </flow>

  <control>
    <priority>implementation readiness over optimistic task generation</priority>
    <failure>if any task is not self-contained, mark BLOCKED</failure>

    <single_source_of_truth>
      <rule>Unresolved blocking issue count controls status.</rule>
      <rule>Status controls implementation_readiness.ready.</rule>
      <rule>Fixed issues are never listed as required fixes.</rule>
      <rule>Markdown summary must match XML summary.</rule>
      <rule>Markdown readiness must match XML readiness.</rule>
      <rule>Required fixes must match unresolved blocking issues only.</rule>
    </single_source_of_truth>

    <hard_rules>
      <rule>Do not implement product code.</rule>
      <rule>Do not mark tasks complete.</rule>
      <rule>Do not update task progress.</rule>
      <rule>Do not allow /prd-implement if blocking issues remain.</rule>
      <rule>Do not accept vague acceptance checks.</rule>
      <rule>Do not accept git add . or git add -A.</rule>
      <rule>Do not accept CREATE for an existing file.</rule>
      <rule>Do not accept MODIFY for a missing file unless the task also creates it.</rule>
      <rule>Do not accept parallel-safe claims when file sets overlap.</rule>
      <rule>Enhancement: do not allow implementation without compatibility gate.</rule>
    </hard_rules>
  </control>

  <final>
    <if status="VALIDATED">
      ## ✅ Tasks Validated

      Validation report written to:
      `{folder}/tasks/validation.md`

      ## ▶️ Next Step — Start Implementation

      Start one task:
      `/prd-implement {folder}/tasks/{start-task}.md`

      Or run a validated parallel layer:
      `/prd-implement --parallel-layer {layer}`
    </if>

    <if status="BLOCKED">
      ## ❌ Task Validation Blocked

      Validation report written to:
      `{folder}/tasks/validation.md`

      Fix the unresolved blocking issues, then rerun:
      `/prd-validate {folder}/tasks/index.md`
    </if>
  </final>

</command>