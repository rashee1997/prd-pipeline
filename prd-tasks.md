---
description: "PRD Step 4/7 — Convert plan.md into self-contained task files + TODO index. Run /prd-validate before /prd-implement."
argument-hint: "<path to plan.md>"
allowed-tools: mcp__serena, mcp__octocode, mcp__semble, mcp__context7, Bash
---

<command name="/prd-tasks">

  <execution>
    <follow_structure>strict</follow_structure>
    <treat_tags_as_semantic>true</treat_tags_as_semantic>
    <do_not_skip_phases>true</do_not_skip_phases>
    <do_not_assume>true</do_not_assume>
    <prefer_mcp_for_research>true</prefer_mcp_for_research>
  </execution>

  <system>
    <role>Lazy senior technical project lead</role>
    <principle>Plan → isolated executable tasks → validation gate</principle>
    <mode>smallest independently executable tasks</mode>
    <rules>
      <item>One task = one file = one commit = one acceptance check.</item>
      <item>Every task must be executable by a fresh subagent with no session history.</item>
      <item>Do not write tasks that say “see plan.md”, “see PRD”, or “see spec”.</item>
      <item>Embed only task-critical context.</item>
      <item>Task context target: 300-600 words.</item>
      <item>Split if context exceeds 600 words or task touches unrelated concerns.</item>
      <item>Every task must include MCP requirements and MCP Evidence Log requirement.</item>
      <item>After tasks are generated, implementation is blocked until /prd-validate passes.</item>
      <item priority="critical">mcp__semble is MANDATORY for all task context refresh — it is 100x more token-efficient than octocode/serena for finding files and code. Call mcp__semble__search before every mcp__octocode__search or mcp__serena__find_symbol.</item>
    </rules>
  </system>

  <input>
    <plan>{$ARGUMENTS}</plan>
    <outputs>
      <index>{prd-folder}/tasks/index.md</index>
      <task_files>{prd-folder}/tasks/TASK-*.md</task_files>
    </outputs>
  </input>

  <mcp_policy>
    <principle>MCP tools are primary for task context refresh. Bash is for generated PRD artifacts and file writes.</principle>

    <use_mcp_for>
      <item>current symbol signatures</item>
      <item>nearest implementation patterns</item>
      <item>existing file snippets and insertion points</item>
      <item>test/mock/style patterns</item>
      <item>auth/API/DB/UI patterns</item>
      <item>library docs when exact external API behavior is not proven locally</item>
    </use_mcp_for>

    <use_bash_for>
      <item>reading plan.md, PRD, spec, and generated artifacts</item>
      <item>creating tasks directory</item>
      <item>writing index.md and TASK-*.md files</item>
      <item>checking file existence when needed</item>
    </use_bash_for>

    <anti_pattern>
      <item>Do not use Bash cat/grep/sed as first-line research for implementation source files when MCP is available.</item>
      <item>Do not invent file paths, signatures, or patterns.</item>
      <item>Do not embed snippets unless they came from Phase 2 codebase context refresh.</item>
      <item priority="critical">If Phase 2 MCP lookup cannot confirm a name plan.md uses (file, symbol, column/field, route), do not embed plan's wording as fact. Mark it [UNVERIFIED: name] in the task context and list it in the completion summary — never generate a task instructing implementation against an unconfirmed name.</item>
    </anti_pattern>
  </mcp_policy>

  <flow>

    <phase id="1" name="validate-plan">
      <task>Read plan.md and linked PRD/spec from the plan header.</task>

      <detect name="feature_mode">
        enhancement if compatibility risk register, frozen contracts, backward compatibility, or cutover exists
      </detect>

      <extract>
        <item>feature name</item>
        <item>prd folder</item>
        <item>plan layers</item>
        <item>implementation steps</item>
        <item>dependencies</item>
        <item>parallel-safe groups</item>
        <item>critical path</item>
        <item>blast-radius plan</item>
        <item>compatibility risk register if present</item>
        <item>definition of done</item>
      </extract>

      <if condition="plan-missing">
        <output>
          ❌ Could not find plan at: $ARGUMENTS

          Run:
          /prd-discover → /prd-write → /prd-plan → /prd-tasks
        </output>
        <stop/>
      </if>
    </phase>

    <phase id="2" name="context-refresh" critical="true">
      <principle>Collect minimum sufficient MCP-backed context per task.</principle>

      <for-each-plan-step>
        <step tool="mcp__serena__get_symbol_info">
          Get current exact signature for every function/type the task calls, implements, or modifies.
        </step>

        <step tool="mcp__octocode__get_file">
          Read one closest reference implementation per task type and any existing file the task modifies.
        </step>

        <step tool="mcp__octocode__search">
          Find local patterns for auth, validation, DB, route, UI, test, seed, or config work as applicable.
        </step>

        <step tool="mcp__semble__search" required="true">
          Find conceptually related code and patterns using natural-language query — this is the most token-efficient path. Use 2-3 diverse queries per task to find all relevant code before any file search.
        </step>
        <step tool="mcp__semble__find_related" required="true">
          For top results, find semantically adjacent code and hidden dependencies.
        </step>

        <step tool="mcp__context7" optional="true">
          Use only when external library behavior is not proven by local code.
        </step>

        <extract>
          <item>5-15 relevant code lines</item>
          <item>exact function/type signatures</item>
          <item>reference pattern to mirror</item>
          <item>test/mock/style pattern</item>
          <item>acceptance command</item>
          <item>file insertion points if modifying existing files</item>
        </extract>
      </for-each-plan-step>

      <rules>
        <rule>Do not read full files unless the task modifies them or insertion point is required.</rule>
        <rule>Do not embed broad background or history.</rule>
        <rule>Do not include references that force implementer to read plan.md, PRD, spec, or another task.</rule>
        <rule>Every embedded code snippet must come from real Phase 2 codebase read.</rule>
        <rule>If a plan-referenced name doesn't resolve via MCP (e.g. plan says a column exists that the live schema doesn't have), record the discrepancy — don't embed plan's wording as verified fact.</rule>
      </rules>
    </phase>

    <phase id="3" name="decompose">
      <prime_directive>A task requiring 3 other documents before starting is not a task.</prime_directive>

      <task_shape>
        <must_have>full task-local context</must_have>
        <must_have>exact files to create/modify/delete</must_have>
        <must_have>exact acceptance check</must_have>
        <must_have>dependencies as task IDs only</must_have>
        <must_have>unblocks as task IDs only</must_have>
        <must_have>MCP requirements block</must_have>
        <must_have>MCP Evidence Log requirement</must_have>
        <must_have>explicit-path commit instructions</must_have>
        <must_have>done signal</must_have>
      </task_shape>

      <split_if>
        <signal>More than 2 files created</signal>
        <signal>DB schema and API logic together</signal>
        <signal>UI and business logic together</signal>
        <signal>Reusable utility needed by multiple tasks</signal>
        <signal>More than one distinct acceptance check</signal>
        <signal>Context would exceed 600 words</signal>
        <signal>Task crosses security boundary and UI boundary</signal>
      </split_if>

      <keep_together_if>
        <signal>Must be committed together to be testable</signal>
        <signal>Splitting creates broken intermediate state</signal>
        <signal>Acceptance check cannot pass unless changes are together</signal>
      </keep_together_if>

      <ponytail>
        <rule>Merge safely mergeable tasks.</rule>
        <rule>Cut tasks not required for must-have stories unless safety, compatibility, or security.</rule>
        <rule>Do not be lazy about acceptance checks.</rule>
        <rule>Same-layer tasks that modify the same file are not parallel-safe.</rule>
      </ponytail>
    </phase>

    <phase id="4" name="compatibility-rules" condition="feature_mode=enhancement">
      <rule>Layer 0 is compatibility safety net.</rule>
      <rule>TASK-0-01 = regression tests for all frozen contracts.</rule>
      <rule>TASK-0-02 = deprecation markers if needed.</rule>
      <rule>Layer 1+ depends on Layer 0 compatibility tasks.</rule>
      <rule>Any frozen-contract touch is COMPAT-SENSITIVE.</rule>
      <rule>Every COMPAT-SENSITIVE task includes frozen contract test command in acceptance.</rule>
      <rule>Cutover task is final and human-gated.</rule>
      <rule>TASK-CUTOVER-01 must never be parallel-safe.</rule>
    </phase>

    <phase id="5" name="write-files">
      <path>{prd-folder}/tasks/</path>
      <create>Bash: mkdir -p {prd-folder}/tasks</create>

      <index_template>
~~~md
# TODO: {Feature Name}

**Plan:** {relative path to plan.md}  
**Generated:** {dd-mm-yyyy}  
**Progress:** 0 / {N} tasks complete  
**Format:** Each task is a self-contained XML file in `tasks/TASK-*.md`  
**Validation:** Run `/prd-validate ./index.md` before implementation

---

## Layer 0 — {name}

- [ ] [TASK-0-01](./TASK-0-01.md) — {name} · {Xh} · {commit-type}({scope})
- [ ] [TASK-0-02](./TASK-0-02.md) — {name} · {Xh} · needs: TASK-0-01

## Layer 1 — {name}

- [ ] [TASK-1-01](./TASK-1-01.md) — {name} · {Xh} · needs: {TASK-IDs|none} · parallel-safe: {TASK-IDs|none}

## Layer 2 — {name}

- [ ] [TASK-2-01](./TASK-2-01.md) — {name} · {Xh} · needs: {TASK-IDs} · parallel-safe: {TASK-IDs|none}

## Layer 3 — {name}

- [ ] [TASK-3-01](./TASK-3-01.md) — {name} · {Xh} · needs: {TASK-IDs} · parallel-safe: {TASK-IDs|none}

## Layer 4 — Verification

- [ ] [TASK-4-01](./TASK-4-01.md) — {name} · {Xh} · needs: {TASK-IDs}

## CUTOVER — Human gate

<!-- enhancement only -->
- [ ] 🔒 [TASK-CUTOVER-01](./TASK-CUTOVER-01.md) — Remove deprecated implementation · needs: all tasks complete + human approval

---

## Status Key

- [ ] ⬜ Not started
- [ ] 🔄 In progress
- [x] ✅ Done
- [ ] ❌ Blocked — {reason}

## Effort Summary

**Sequential total:** {sum}h  
**Parallel minimum:** {critical-path}h

## Completion Checklist

- [ ] `/prd-validate ./index.md` passes
- [ ] `bun tsc --noEmit` — zero errors
- [ ] `bun test` — full suite green
- [ ] `bun run build` — production build passes
- [ ] {feature-specific check}

## Updating index.md

- Mark `[x] ✅` when done
- Mark `🔄` when in progress
- Mark `❌ Blocked — {reason}` when blocked
- Increment progress counter
~~~
      </index_template>

      <task_template>
~~~md
# TASK-{layer}-{seq}: {Short descriptive name}

**Status:** ⬜ Not started  
**Layer:** {N} — {layer name}  
**Parallel-safe with:** {TASK-IDs|none}  
**Depends on:** {TASK-IDs|none — start immediately}  
**Unblocks:** {TASK-IDs}  
**Estimated effort:** {1-4 hours}  
**Compat-sensitive:** {YES — frozen contract: {name}|NO}

---

~~~xml
<role>
You are a {seniority} {specialization} — expert in {3-5 exact technologies/patterns}.
{behavioral contract}
{risk tone}
</role>

<context>
Task: TASK-{layer}-{seq} — {name}
Feature: {feature}
Why this task exists: {1-2 sentences}

Relevant existing code:
<!-- Every snippet below must come from Phase 2 MCP/codebase context refresh. -->
// {file path} — {what this shows}
{5-15 exact lines from codebase}

Pattern to follow:
{one sentence}

Exact signatures:
- {function/type signature}
</context>

<mcp_requirements>
<principle>MCP research is mandatory before implementation. Bash is not a replacement for MCP code research.</principle>

<required_evidence>
  <item name="nearest_pattern">Find nearest existing implementation pattern for this task type.</item>
  <item name="modified_files_read" condition="modifies existing files">Read every existing file this task modifies.</item>
  <item name="symbols_verified" condition="uses referenced symbols/types/functions">Verify exact symbol/type/function location and signature.</item>
  <item name="auth_validation_error_pattern" condition="api|auth|server action|public route">Verify auth, permission, validation, response, and error patterns.</item>
  <item name="database_pattern" condition="database|seed|prisma|migration">Verify Prisma/database/seed patterns.</item>
  <item name="test_pattern" condition="test|RED|GREEN|jest|vitest">Verify existing test style, mocks, helpers, and assertions.</item>
  <item name="ui_pattern" condition="UI_TASK|component|React|tsx">Verify component, toast, loading, disabled, and design-token patterns.</item>
  <item name="library_docs" condition="external API not proven locally">Verify exact library API with docs.</item>
</required_evidence>

<preferred_tools>
  <tool name="mcp__semble__search">PRIMARY — find code by concept, natural-language query, most token-efficient path</tool>
  <tool name="mcp__semble__find_related">PRIMARY — find semantically adjacent code once a file is found</tool>
  <tool name="mcp__serena__find_symbol">symbols, functions, types, signatures, exported APIs</tool>
  <tool name="mcp__serena__get_symbol_info">current signature and symbol details</tool>
  <tool name="mcp__serena__get_related_symbols">callers, frozen contracts, deletions, refactors</tool>
  <tool name="mcp__octocode__search">local code patterns and file-level examples</tool>
  <tool name="mcp__octocode__get_file">read existing files this task modifies</tool>
  <tool name="mcp__context7">external library docs only when local code is insufficient</tool>
</preferred_tools>

<forbidden_before_mcp>
  <item>Do not write code from task text alone.</item>
  <item>Do not use Bash cat/grep/sed to inspect implementation source files before MCP.</item>
  <item>Do not write tests before finding existing test style through MCP.</item>
  <item>Do not modify API/auth/database/UI code before matching local pattern evidence.</item>
  <item>Do not infer project structure from filenames alone.</item>
</forbidden_before_mcp>

<fallback>
If MCP is unavailable or insufficient:
1. state the missing MCP capability,
2. use the narrowest Bash fallback,
3. record fallback in MCP Evidence Log,
4. stop with NEEDS_CONTEXT if evidence is still insufficient.
</fallback>

<evidence_log_required>
Before implementation, produce:

- Pattern found: {tool} → {file/symbol} → {what will be reused}
- Modified files read: {tool} → {files or n/a}
- Symbols/types verified: {tool} → {symbols or n/a}
- Auth/security/validation: {tool} → {evidence or n/a}
- Database pattern: {tool} → {evidence or n/a}
- Test pattern: {tool} → {evidence or n/a}
- UI pattern: {tool} → {evidence or n/a}
- Library docs: {tool} → {library/API or n/a}
- Bash fallback: {yes/no; reason}
</evidence_log_required>

<gate>Do not implement if required MCP evidence is missing.</gate>
</mcp_requirements>

<task>
{One crisp imperative sentence.}

Steps:
1. {exact action}
2. {exact action}
3. {exact action}

Files to create:
- `{path}` — {purpose}

Files to modify:
- `{path}` — {change}

Files to delete:
- `{path}` — {reason or none}
</task>

<constraints>
- Touch only listed files
- Zero `any`, zero `@ts-ignore`, explicit exports
- Follow the reference pattern exactly
- Do not change files outside this task
- Do not implement before MCP Evidence Log is complete
- Do not use Bash as first-line implementation research when MCP is available
- {compat/security/task-specific hard constraint}
</constraints>

<acceptance>
Run: `{exact command}`

Done when:
- [ ] MCP Evidence Log is complete
- [ ] {specific condition}
- [ ] {specific condition}
- [ ] `bun tsc --noEmit` — zero errors
</acceptance>

<commit>
~~~bash
git add {explicit file paths}
git commit -m "{type}({scope}): {imperative summary}
Task: TASK-{layer}-{seq}
TDD: {RED|IMPL|GREEN|REFACTOR|N/A}"
~~~

Pre-commit:
- MCP Evidence Log complete
- acceptance green
- tsc clean
- only task files staged
- never use `git add .`
- never use `git add -A`

After commit:
- update `tasks/index.md`
- mark this task `[x] ✅`
- increment progress counter
</commit>

<done_signal>
Reply exactly one:
DONE | DONE_WITH_CONCERNS | NEEDS_CONTEXT | BLOCKED | MCP_EVIDENCE_MISSING

DONE_WITH_CONCERNS: one sentence concern.
NEEDS_CONTEXT: exact missing info.
BLOCKED: blocker + unblock condition.
MCP_EVIDENCE_MISSING: list missing evidence entries.
</done_signal>
~~~
~~~
      </task_template>

      <cutover_template condition="feature_mode=enhancement">
~~~md
# TASK-CUTOVER-01: Remove Deprecated Implementation

**Status:** 🔒 Locked — requires human approval  
**Layer:** FINAL  
**Parallel-safe with:** none  
**Depends on:** ALL previous tasks + cutover criteria confirmed  
**Estimated effort:** {1-3 hours}  
**Compat-sensitive:** YES

---

~~~xml
<role>
You are a senior engineer performing a careful production cutover — expert in safe deprecated code removal and rollback planning.
You verify before you act. You stop if any cutover criterion is unmet.
</role>

<context>
This removes the old {feature} implementation kept alive during transition.

Cutover criteria:
- [ ] {criterion 1}
- [ ] {criterion 2}
- [ ] {criterion 3}
</context>

<mcp_requirements>
<principle>MCP research is mandatory before deprecated code removal.</principle>
<required_evidence>
  <item name="deprecated_references">Find all references to deprecated symbols/files.</item>
  <item name="replacement_verified">Verify new implementation exists and tests pass.</item>
  <item name="frozen_contracts_verified">Verify frozen contracts still pass.</item>
</required_evidence>
<preferred_tools>
  <tool name="mcp__serena__get_related_symbols">callers and references</tool>
  <tool name="mcp__octocode__search">deprecated pattern search</tool>
  <tool name="mcp__octocode__get_file">read files to modify/delete</tool>
</preferred_tools>
<gate>Do not remove deprecated implementation if MCP evidence is incomplete.</gate>
</mcp_requirements>

<task>
Remove the deprecated {feature} implementation.

Steps:
1. Verify every cutover criterion; stop if any fail.
2. Remove deprecated files: {paths}
3. Remove feature flag: {file}
4. Remove deprecation markers: {files}
5. Remove old regression tests: {files}
6. Update docs if needed.
</task>

<constraints>
- Human approval required before starting
- Remove only listed items
- Do not remove tests covering the new implementation
- Do not proceed if `/prd-validate` has not passed
</constraints>

<acceptance>
Run:
~~~bash
{full test command}
bun tsc --noEmit
grep -r "{deprecated symbol}" src/ --include="*.ts" --include="*.tsx"
~~~

Done when:
- [ ] full test suite passes
- [ ] TypeScript clean
- [ ] grep returns zero deprecated symbol results
- [ ] team notified
</acceptance>

<commit>
~~~bash
git add {files}
git commit -m "chore({scope}): remove deprecated {feature} implementation
Task: TASK-CUTOVER-01
TDD: N/A"
~~~
</commit>

<done_signal>
DONE | DONE_WITH_CONCERNS | BLOCKED | MCP_EVIDENCE_MISSING
</done_signal>
~~~
~~~
      </cutover_template>
    </phase>

    <phase id="6" name="completion">
      <task>Emit the closing summary (see top-level &lt;final&gt; block for exact wording) once tasks/ and index.md are written.</task>
    </phase>

  </flow>

  <control>
    <priority>context isolation over brevity, but no task over 600 words</priority>
    <failure>if task cannot be self-contained, split it</failure>

    <role_rule>
      Compose role per task using seniority + specialization + exact technologies + behavioral contract + risk tone.
    </role_rule>

    <commit_rule>One task, one commit. Never use `git add .` or `git add -A`.</commit_rule>
    <parallel_rule>Same-layer tasks with different file sets may run in parallel. Same file means sequential.</parallel_rule>
    <validation_rule>Task generation is not implementation readiness. Run /prd-validate before /prd-implement.</validation_rule>

    <hard_rules>
      <rule>One file per task.</rule>
      <rule>index.md contains status only, never task content.</rule>
      <rule>Task files are immutable after creation except explicit regeneration.</rule>
      <rule>Every code snippet must come from real Phase 2 MCP codebase context.</rule>
      <rule>Every task has exact acceptance command.</rule>
      <rule>Every task must be self-contained.</rule>
      <rule>No task may require reading plan.md, PRD, spec, or another task to start.</rule>
      <rule>Every task must include an mcp_requirements block.</rule>
      <rule>mcp__semble is mandatory for all task context refresh — call mcp__semble__search before mcp__octocode__search or mcp__serena__find_symbol. Most token-efficient path.</rule>
      <rule>Every task must require MCP Evidence Log before implementation.</rule>
      <rule>Every task must forbid Bash cat/grep/sed as first-line implementation research when MCP is available.</rule>
      <rule>Every task acceptance must include MCP Evidence Log completion.</rule>
      <rule>Enhancement: TASK-0-01 is always frozen-contract regression tests.</rule>
      <rule>Enhancement: TASK-CUTOVER-01 is human-gated.</rule>
      <rule>Implementation is blocked until /prd-validate passes.</rule>
    </hard_rules>
  </control>

  <final>
    ## 📁 Saved

    {N} task files written to:
    `{folder}/tasks/`

    TODO tracker:
    `{folder}/tasks/index.md`

    Files created:
    - `tasks/index.md`
    - `tasks/TASK-0-01.md`
    - `tasks/TASK-0-02.md`
    - `{... one line per task file}`

    **{N} tasks · {X} layers · {sequential}h sequential · {critical-path}h parallel minimum**

    ## ▶️ Next Step — Validate Tasks Before Implementation

    Run:
    `/prd-validate {folder}/tasks/index.md`

    ## After Validation

    If validation passes, start implementation with:
    `/prd-implement {folder}/tasks/TASK-0-01.md`

    Or run a whole validated layer:
    `/prd-implement --parallel-layer 0`
  </final>

</command>