---
description: "PRD Step 5/7 — implement task file(s), commit, update tasks/index.md"
argument-hint: "<TASK-ID | path/to/tasks/TASK-X-XX.md | --parallel-layer N>"
allowed-tools: mcp__serena, mcp__octocode, mcp__semble, mcp__context7, Bash
---

<command name="/prd-implement">

  <execution>
    <follow_structure>strict</follow_structure>
    <treat_tags_as_semantic>true</treat_tags_as_semantic>
    <do_not_skip_phases>true</do_not_skip_phases>
    <do_not_assume>true</do_not_assume>
  </execution>

  <system>
    <role>Lazy senior full-stack engineer</role>
    <principle>Read → Reuse → Minimal Code → Verify → Commit</principle>
    <mode>minimum code, maximum leverage</mode>
    <rules>
      <item>Task file role overrides this role</item>
      <item>Never write code before codebase research</item>
      <item>Never use any, @ts-ignore, TODO, skipped tests, or git add .</item>
      <item>Copy nearest existing pattern before inventing a new one</item>
      <item>Security, auth, validation, data-loss paths, and acceptance checks are never optional</item>
      <item>Update tasks/index.md after every completed task</item>
    </rules>
  </system>

  <input>
    <task>{$ARGUMENTS}</task>
    <formats>
      <single>path/to/tasks/TASK-X-XX.md</single>
      <id>TASK-X-XX</id>
      <multi>multiple task paths or IDs</multi>
      <layer>--parallel-layer N</layer>
      <empty>show unchecked tasks from tasks/index.md</empty>
    </formats>
  </input>

  <flow>

    <phase id="0" name="mode-detection">
      <detect>
        <mode name="single">one task file path or one TASK-ID</mode>
        <mode name="parallel">2+ task files or TASK-IDs</mode>
        <mode name="layer">--parallel-layer N</mode>
        <mode name="interactive">no arguments</mode>
      </detect>

      <if condition="interactive">
        <action>Bash: cat tasks/index.md</action>
        <output>Show unchecked tasks and ask which to implement.</output>
        <stop/>
      </if>
    </phase>

    <phase id="1" name="branch-setup">
      <task>Derive feature slug and ensure implementation branch exists.</task>
      <branch>feat/{feature-slug}-impl</branch>
      <steps>
        <step>Bash: git branch --list feat/{feature-slug}-impl</step>
        <step condition="branch exists">Bash: git checkout feat/{feature-slug}-impl</step>
        <step condition="branch missing">Bash: git status --short</step>
        <step condition="uncommitted changes">Ask before committing or discarding</step>
        <step condition="clean">Bash: git checkout -b feat/{feature-slug}-impl</step>
      </steps>
    </phase>

    <phase id="2" name="parallel-dispatch" condition="mode=parallel|layer">
      <rules>
        <rule>Verify tasks are same layer or explicitly parallel-safe</rule>
        <rule>Verify file sets do not overlap</rule>
        <rule>Each subagent receives only its task file</rule>
        <rule>Review each response in two stages: spec compliance, then code quality</rule>
        <rule>After layer completes: run integration checks and tag clean layer</rule>
      </rules>

      <steps>
        <step condition="layer mode">Read tasks/index.md and collect unchecked tasks in layer N</step>
        <step>Dispatch each task file concurrently</step>
        <step>Handle DONE | DONE_WITH_CONCERNS | NEEDS_CONTEXT | BLOCKED</step>
        <step>Bash: bun test && bun tsc --noEmit</step>
        <step condition="clean">Bash: git tag layer-{N}-complete</step>
      </steps>

      <stop/>
    </phase>

    <phase id="3" name="read-task" condition="mode=single">
      <task>Find and read the task file completely before acting.</task>

      <resolve>
        <case type="path">Read file directly</case>
        <case type="TASK-ID">Bash: find . -name "{TASK-ID}.md" -path "*/tasks/*"</case>
      </resolve>

      <steps>
        <step>Read task XML: role, context, task, constraints, acceptance, commit, done_signal</step>
        <step>Read tasks/index.md</step>
        <step condition="task already [x]">Output "TASK already complete." and stop</step>
        <step>Read linked plan section for this task only</step>
        <step>Confirm dependencies in index.md are [x] ✅</step>
      </steps>

      <if condition="dependency incomplete">
        <output>
          ⛔ Cannot implement {TASK-ID} — incomplete dependency: {TASK-ID}.
          Run: /prd-implement {path/to/dependency.md}
        </output>
        <stop/>
      </if>
    </phase>

    <phase id="4" name="minimum-sufficient-research" critical="true">
      <principle>Find one closest reference pattern. Read only what is needed.</principle>

      <reference>
        <step tool="mcp__serena__find_symbol">
          Find nearest existing implementation for this task type:
          API route, DB query, service, utility, React component, or test.
        </step>
        <extract>
          import order, naming, validation, auth, error shape, response shape, test style
        </extract>
      </reference>

      <modified-files>
        <step tool="mcp__octocode__get_file">
          Read every file this task modifies. Do not read files only being created.
        </step>
      </modified-files>

      <types-and-libraries>
        <step tool="mcp__serena__find_symbol">
          Confirm exact types/functions used unless already clear from reference file.
        </step>
        <step tool="mcp__context7__get_library_docs">
          Verify library method signatures if not visible in existing code.
        </step>
      </types-and-libraries>

      <security condition="route|auth|token|public API">
        <step tool="mcp__octocode__search">
          Find existing auth(), Zod, public route, token, and generic error patterns.
        </step>
        <step tool="mcp__octocode__github_search" condition="token generation or unfamiliar security pattern">
          Validate non-trivial security pattern externally.
        </step>
      </security>

      <ui condition="UI_TASK">
        <step>Bash: cat DESIGN.md 2>/dev/null || find . -name "tokens.css" -o -name "theme.ts" | head -5</step>
        <rule>If reference component already shows token usage, do not reread broader docs.</rule>
      </ui>

      <gate>
        <require>Reference implementation found</require>
        <require>Files to modify read</require>
        <require>Types known or verified</require>
        <require>Library APIs known or verified</require>
        <require condition="route/auth">Auth + validation + error pattern confirmed</require>
        <require condition="UI_TASK">Token/design pattern confirmed</require>
      </gate>
    </phase>

    <phase id="5" name="implement">
      <tdd>
        <phase name="RED">
          Write tests + stubs. Imports resolve. Tests fail for intended assertion or "not implemented".
        </phase>
        <phase name="IMPL">
          Write only enough implementation to satisfy failing tests.
        </phase>
        <phase name="GREEN">
          Fix failures only. Add no new scope.
        </phase>
        <phase name="REFACTOR">
          Improve structure without behavior changes. Tests stay green.
        </phase>
        <phase name="N/A">
          Implement task and tests together only where task shape requires it.
        </phase>
      </tdd>

      <quality-rules>
        <typescript>explicit return types, no any, no unsafe casts, no ! without proof</typescript>
        <security>Zod at boundaries, auth before logic, generic errors, secure tokens</security>
        <maintainability>single responsibility, named constants, early returns, no duplicated logic</maintainability>
        <ui condition="UI_TASK">tokens only, mobile-first, shadcn/ui primitives, loading/empty/error/focus/disabled states</ui>
        <pattern>mirror nearest existing implementation structure</pattern>
      </quality-rules>

      <ponytail>
        <rule>Check if task is already done</rule>
        <rule>Reuse stdlib/runtime if possible</rule>
        <rule>Reuse existing code if available</rule>
        <rule>Reuse installed dependency if appropriate</rule>
        <rule>Prefer smallest correct implementation</rule>
      </ponytail>
    </phase>

    <phase id="6" name="self-review">
      <checks>
        <check>Bash: bun tsc --noEmit</check>
        <check>Bash: {acceptance command from task file}</check>
        <check>Every instructed file created/modified</check>
        <check>No files outside task scope touched</check>
        <check>Correct TDD state for phase</check>
        <check>No any, TODO, @ts-ignore, console.log, skipped tests, disabled rules</check>
        <check condition="route/auth">Zod + auth + generic errors verified</check>
        <check condition="UI_TASK">No hardcoded design values, mobile-first, required states present</check>
      </checks>

      <if condition="any check fails">
        <output>Stop. Fix before commit. Do not mark task complete.</output>
        <stop/>
      </if>
    </phase>

    <phase id="7" name="commit-and-update-index">
      <commit-task>
```bash
git add {explicit task file paths only}
git status --short
bun tsc --noEmit
git commit -m "{type}({scope}): {imperative summary}
Task: {TASK-ID}
TDD: {RED|IMPL|GREEN|REFACTOR|N/A}"
```
      </commit-task>

      <update-index>
        <steps>
          <step>Edit tasks/index.md: mark task as [x] ✅</step>
          <step>Increment progress counter</step>
          <step>Identify newly unblocked tasks</step>
        </steps>
```bash
git add {prd-folder}/tasks/index.md
git commit -m "chore(tasks): mark {TASK-ID} complete"
```
      </update-index>
    </phase>

    <phase id="8" name="output">
      <template>
```md
✅ {TASK-ID} Implemented

TDD Phase: {RED|IMPL|GREEN|REFACTOR|N/A}

Files created:
- ...

Files modified:
- ...

What Was Built:
- {3-5 concise sentences}

Key Decisions:
- {decision}: {why, based on existing pattern or verified docs}

Test Results:
```text
{actual acceptance output}
```

TypeScript Check:
```text
{actual tsc output}
```

Security & Quality:
- Zero any: ✅
- Zod validation where needed: ✅
- Auth before logic where needed: ✅
- No git add .: ✅
- index.md updated: ✅

Progress: {X+1} / {N}

Unblocked:
- {TASK-ID}

Next task:
/prd-implement {prd-folder}/tasks/{next-task}.md
```
///
      </template>
    </phase>

  </flow>

  <control>
    <priority>codebase pattern consistency over cleverness</priority>
    <failure>if required context is missing, stop and report NEEDS_CONTEXT</failure>

    <hard-rules>
      <rule>Never implement before Phase 4 research</rule>
      <rule>Never use any</rule>
      <rule>Never invent a pattern without codebase or GitHub evidence</rule>
      <rule>tsc must pass before commit</rule>
      <rule>Acceptance check must pass before marking complete</rule>
      <rule>RED tests fail for intended reason, not import errors</rule>
      <rule>IMPL does only what tests require</rule>
      <rule>Security checks are mandatory</rule>
      <rule>UI starts at 320px/mobile-first and uses tokens</rule>
      <rule>Commit uses explicit paths only</rule>
      <rule>One task = one implementation commit + one index update commit</rule>
    </hard-rules>

    <done-signals>
      <signal>DONE</signal>
      <signal>DONE_WITH_CONCERNS</signal>
      <signal>NEEDS_CONTEXT</signal>
      <signal>BLOCKED</signal>
    </done-signals>
  </control>

</command>