---
description: "PRD Step 4a/7 — Validates generated task files before implementation. Research-backed checks for context isolation, dependencies, file paths, acceptance commands, blast-radius coverage, and parallel safety."
argument-hint: "<path/to/tasks/index.md | path/to/tasks/>"
allowed-tools: mcp__serena, mcp__octocode, mcp__semble, mcp__context7, Bash
---

<command name="/prd-validate">

  <execution>
    <follow_structure>strict</follow_structure>
    <treat_tags_as_semantic>true</treat_tags_as_semantic>
    <do_not_skip_phases>true</do_not_skip_phases>
    <do_not_assume>true</do_not_assume>
  </execution>

  <system>
    <role>Senior implementation-readiness reviewer</role>
    <principle>Validate before implementation</principle>
    <mode>research-backed task validation gate</mode>
    <rules>
      <item>Do not implement code.</item>
      <item>Do not modify product code.</item>
      <item>Validate tasks against plan, spec, PRD, and live codebase.</item>
      <item>Every task must be executable by a fresh subagent with no extra context.</item>
      <item>Every acceptance command must be concrete and runnable.</item>
      <item>Every dependency must reference task IDs only.</item>
      <item>Every file path must be real, proposed, or explicitly justified.</item>
      <item>Flag blocking issues before /prd-implement is allowed.</item>
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

  <flow>

    <phase id="1" name="load-task-set">
      <task>Find tasks/index.md and all tasks/TASK-*.md files.</task>

      <resolve>
        <case type="index path">Use parent directory as tasks folder.</case>
        <case type="tasks directory">Use tasks/index.md inside that directory.</case>
      </resolve>

      <read>
        tasks/index.md
        all tasks/TASK-*.md
      </read>

      <extract>
        <item>Feature name</item>
        <item>Plan path</item>
        <item>Task IDs</item>
        <item>Layers</item>
        <item>Dependencies</item>
        <item>Parallel-safe claims</item>
        <item>Acceptance commands</item>
        <item>Files to create/modify</item>
        <item>Compat-sensitive flags</item>
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
      <task>Load the source documents needed to validate tasks.</task>

      <read>
        plan.md from index header
        prd.md from plan frontmatter
        spec.md from plan frontmatter
      </read>

      <validate>
        <check>Plan file exists.</check>
        <check>PRD file exists.</check>
        <check>Spec file exists.</check>
        <check>Plan contains dependency graph.</check>
        <check>Spec contains evidence index or blast-radius map.</check>
      </validate>

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
      <task>Validate each task file without codebase calls first.</task>

      <per-task-checks>
        <check name="identity">Task ID in filename matches title and metadata.</check>
        <check name="status">Status exists and is not already inconsistent with index.</check>
        <check name="dependencies">Depends on contains task IDs only; no "see plan.md".</check>
        <check name="unblocks">Unblocks references real task IDs.</check>
        <check name="context">Context is self-contained and task-specific.</check>
        <check name="context-size">Context is roughly 300-600 words unless intentionally simple.</check>
        <check name="role">Role is specialized, not generic.</check>
        <check name="files">Files to create/modify are explicit.</check>
        <check name="acceptance">Acceptance command is exact, not vague.</check>
        <check name="commit">Commit block uses explicit file paths, never git add .</check>
        <check name="done-signal">DONE | DONE_WITH_CONCERNS | NEEDS_CONTEXT | BLOCKED present.</check>
      </per-task-checks>

      <fail-if>
        <condition>Task requires reading plan.md, PRD, spec, or another task to execute.</condition>
        <condition>Acceptance says "verify it works" without command/check.</condition>
        <condition>Task modifies broad/unlisted files.</condition>
        <condition>Task has more than one unrelated concern.</condition>
      </fail-if>
    </phase>

    <phase id="4" name="dependency-graph-validation">
      <task>Validate task graph against plan dependency graph.</task>

      <checks>
        <check>Every plan step is represented by one or more task files.</check>
        <check>Every task maps to a plan step.</check>
        <check>No dependency references a missing task.</check>
        <check>No circular dependencies.</check>
        <check>Layer ordering is respected.</check>
        <check>Same-layer parallel-safe tasks do not modify the same file.</check>
        <check>Sequential tasks are required when file sets overlap.</check>
      </checks>

      <enhancement-checks condition="compatibility risk register exists">
        <check>TASK-0-01 exists and covers frozen contract regression tests.</check>
        <check>All Layer 1+ tasks depend on compatibility gate tasks.</check>
        <check>Compat-sensitive tasks include frozen contract acceptance command.</check>
        <check>Cutover task exists only if spec requires old implementation removal.</check>
        <check>Cutover task is human-gated and final.</check>
      </enhancement-checks>
    </phase>

    <phase id="5" name="research-backed-codebase-validation" critical="true">
      <task>Validate task instructions against live codebase.</task>
      <principle>Research only what tasks reference. Do not broad-scan unnecessarily.</principle>

      <referenced-symbols>
        <step tool="mcp__serena__find_symbol">
          Confirm existing symbols/functions/types/components named in tasks.
        </step>
        <step tool="mcp__serena__get_symbol_info">
          Confirm signatures embedded in tasks still match live code.
        </step>
      </referenced-symbols>

      <referenced-files>
        <step tool="mcp__octocode__get_file">
          Read files each task modifies to confirm insertion points and snippets are current.
        </step>
      </referenced-files>

      <file-path-check>
        <step>Bash: test -e {existing file paths}</step>
        <step>Bash: test ! -e {new file paths}</step>
      </file-path-check>

      <pattern-validation>
        <step tool="mcp__octocode__search">
          Verify each task's claimed reference pattern exists in the codebase.
        </step>
      </pattern-validation>

      <library-check optional="true">
        <step tool="mcp__context7__get_library_docs">
          Re-check library APIs only if task acceptance or implementation depends on exact method behavior.
        </step>
      </library-check>

      <security-check condition="task touches auth|public API|token|permissions|PII|data deletion">
        <step tool="mcp__octocode__search">
          Verify task references existing auth, validation, error, token, or permission patterns.
        </step>
      </security-check>
    </phase>

    <phase id="6" name="acceptance-command-validation">
      <task>Validate commands are concrete and likely runnable.</task>

      <checks>
        <check>Each task has exactly one primary acceptance command unless explicitly justified.</check>
        <check>Commands use project tooling visible in package scripts or existing command patterns.</check>
        <check>TypeScript check is present where code is changed.</check>
        <check>Test command is scoped enough for task but not meaningless.</check>
        <check>Compat-sensitive task includes frozen contract regression command.</check>
      </checks>

      <allowed-command-examples>
        <example>bun tsc --noEmit</example>
        <example>bun test path/to/test</example>
        <example>bun run build</example>
        <example>grep -r "{symbol}" src/</example>
      </allowed-command-examples>

      <fail-if>
        <condition>Acceptance command is missing.</condition>
        <condition>Acceptance command is vague.</condition>
        <condition>Acceptance cannot prove the task outcome.</condition>
      </fail-if>
    </phase>

    <phase id="7" name="write-validation-report">
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
  <blocking_issues>{count}</blocking_issues>
  <warnings>{count}</warnings>
  <parallel_safety>{valid|invalid}</parallel_safety>
  <compatibility_gate>{present|not_applicable|missing}</compatibility_gate>
</validation_summary>

<blocking_issues>
  <!-- task_id | issue | required_fix -->
</blocking_issues>

<warnings>
  <!-- task_id | warning | suggested_fix -->
</warnings>

<task_validation>
  <task id="{TASK-ID}">
    <status>{pass|fail|warning}</status>
    <context_isolated>{true|false}</context_isolated>
    <dependencies_valid>{true|false}</dependencies_valid>
    <file_paths_valid>{true|false}</file_paths_valid>
    <acceptance_valid>{true|false}</acceptance_valid>
    <parallel_safe>{true|false|n/a}</parallel_safe>
    <notes>{notes}</notes>
  </task>
</task_validation>

<dependency_validation>
  <!-- graph status, missing deps, cycles, layer issues -->
</dependency_validation>

<blast_radius_validation>
  <api/>
  <database/>
  <auth_security/>
  <ui/>
  <workflow_events/>
  <integrations/>
  <tests/>
  <rollback/>
</blast_radius_validation>

<implementation_readiness>
  <ready>{true|false}</ready>
  <start_task>{TASK-0-01 or first valid unchecked task}</start_task>
  <safe_parallel_layers>
    <!-- layer | task IDs -->
  </safe_parallel_layers>
</implementation_readiness>

---

## Validation Summary

**Status:** {VALIDATED|BLOCKED}  
**Tasks checked:** {N}  
**Blocking issues:** {count}  
**Warnings:** {count}  
**Parallel safety:** {valid|invalid}  
**Compatibility gate:** {present|not applicable|missing}

## Blocking Issues

| Task | Issue | Required Fix |
|---|---|---|

## Warnings

| Task | Warning | Suggested Fix |
|---|---|---|

## Task Validation

| Task | Context | Deps | Files | Acceptance | Parallel | Status |
|---|---|---|---|---|---|---|

## Dependency Graph Validation

...

## Blast Radius Validation

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

## Implementation Readiness

**Ready:** {yes|no}

**Start here:**
`/prd-implement {folder}/tasks/{start-task}.md`

**Parallel option if ready:**
`/prd-implement --parallel-layer 0`

## Required Fixes Before Implementation

- ...
```
///
    </template>

      <if condition="blocking issues exist">
        <output>
          ❌ Task validation blocked implementation.

          Write validation.md with required fixes.
          Do not run /prd-implement until fixed.
        </output>
      </if>

      <if condition="no blocking issues">
        <output>
          ✅ Tasks validated. Implementation may begin.
        </output>
      </if>
    </phase>

  </flow>

  <control>
    <priority>implementation readiness over optimistic task generation</priority>
    <failure>if any task is not self-contained, mark BLOCKED</failure>

    <hard-rules>
      <rule>Do not implement code.</rule>
      <rule>Do not mark tasks complete.</rule>
      <rule>Do not update index progress.</rule>
      <rule>Do not allow /prd-implement if blocking issues exist.</rule>
      <rule>Do not accept vague acceptance checks.</rule>
      <rule>Do not accept missing dependency IDs.</rule>
      <rule>Do not accept parallel-safe claims when file sets overlap.</rule>
      <rule>Enhancement: do not allow implementation without compatibility gate.</rule>
    </hard-rules>
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
      `/prd-implement --parallel-layer 0`
    </if>

    <if status="BLOCKED">
      ## ❌ Task Validation Blocked

      Validation report written to:
      `{folder}/tasks/validation.md`

      Fix the blocking issues, then rerun:
      `/prd-validate {folder}/tasks/index.md`
    </if>
  </final>

</command>
``