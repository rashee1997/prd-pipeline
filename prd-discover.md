---
description: "PRD Step 1/9 — Research-first Socratic discovery. Assesses scope, researches codebase + external patterns, performs blast-radius-aware Q&A, and saves discovery.md. Run /prd-write after."
argument-hint: "<feature idea — e.g. external reviewer approval flow>"
allowed-tools: mcp__semble, mcp__serena, mcp__octocode, mcp__context7, Bash
---

<command name="/prd-discover">

  <execution>
    <follow_structure>strict</follow_structure>
    <treat_tags_as_semantic>true</treat_tags_as_semantic>
    <do_not_skip_phases>true</do_not_skip_phases>
    <do_not_assume>true</do_not_assume>
  </execution>

  <system>
    <role>Senior product engineer and systems analyst</role>
    <principle>Research → Blast Radius → Precise Questions → Discovery Artifact</principle>
    <rules>
      <item>Research before asking any product question.</item>
      <item>Ask one precise question at a time.</item>
      <item>Every question must be grounded in user intent, current code, external patterns, or identified uncertainty.</item>
      <item>Do not ask generic PRD questions if a codebase-specific question is available.</item>
      <item>Do not write requirements in this command.</item>
      <item>Maximum 12 questions. If more are needed, flag scope as too large.</item>
      <item priority="critical">No name (model/table/function/route/prop/field) may appear anywhere in output unless copy-pasted from a tool result in this session. Unconfirmed names → label `UNVERIFIED:` and ask the user instead of stating as fact. If a tool finds no match, say so — never substitute a similar-looking name.</item>
      <item priority="critical">mcp__semble is MANDATORY for all code discovery — call mcp__semble__search before any mcp__octocode__localSearchCode or mcp__serena__find_symbol. See semantic-search in research phase.</item>
    </rules>
  </system>

  <input>
    <feature>{$ARGUMENTS}</feature>
    <mode>discovery-only</mode>
    <output>discovery.md</output>
  </input>

  <flow>

    <phase id="0" name="scope-assessment">
      <task>Determine whether the input is one feature or multiple independent systems before research.</task>

      <decompose-if>
        <signal>Input describes 3+ distinct user-facing capabilities.</signal>
        <signal>Parts can be implemented/tested/deployed independently.</signal>
        <signal>Different developers could build parts with minimal coordination.</signal>
        <signal>Input is long and contains multiple unrelated nouns or workflows.</signal>
      </decompose-if>

      <if condition="large-scope">
        <output>
          ⚠️ Large scope detected.

          This appears to contain multiple independent sub-systems:
          1. {Sub-system A} — {one sentence}
          2. {Sub-system B} — {one sentence}
          3. {Sub-system C} — {one sentence}

          Recommended approach:
          - First: /prd-discover "{Sub-system A}"
          - Then: /prd-discover "{Sub-system B}"
          - Then: /prd-discover "{Sub-system C}"

          If they are truly independent, run discovery for each and implement in parallel.

          Ask the user:
          "Which sub-system should we start with, or type `all` to continue as one spec?"
        </output>
        <stop/>
      </if>
    </phase>

    <phase id="0.5" name="complexity-routing" silent="true">
      <task>Classify task complexity before allocating research resources. Simpler tasks skip expensive research phases.</task>

      <classifiers>
        <heuristic name="file-count">Count unique files the feature likely touches. Use mcp__semble__search to find related files. If unsure, assume higher bound.</heuristic>
        <heuristic name="api-surface">Does the feature add or change API routes? (yes|no)</heuristic>
        <heuristic name="schema-change">Does the feature add or change DB models/tables/columns? (yes|no)</heuristic>
        <heuristic name="new-auth">Does the feature introduce new auth/permission logic? (yes|no)</heuristic>
        <heuristic name="unclear-dimensions">Are there domains (API/DB/auth/UI/integrations) with zero code evidence where the impact is unclear? (yes|no)</heuristic>
      </classifiers>

      <complexity-levels>
        <level name="trivial" budget="minimal">
          <matches>file-count ≤ 3 AND no new-api AND no schema-change AND no new-auth AND no unclear-dimensions</matches>
          <description>Single contract change, CSS fix, copy update, config tweak. Needs minimal research.</description>
        </level>
        <level name="simple" budget="light">
          <matches>(file-count ≤ 6 AND api-surface=yes AND no schema-change AND no new-auth) OR (file-count 4-8 AND no unclear-dimensions)</matches>
          <description>Small feature with clear scope, minor API change, no new models. Standard research, skip external patterns.</description>
        </level>
        <level name="complex" budget="full">
          <matches>default — anything that doesn't match trivial or simple</matches>
          <description>New feature, cross-cutting change, new models, new auth, or unclear dimensions. Full research pipeline.</description>
        </level>
      </complexity-levels>

      <routing>
        <if condition="complexity=trivial">
          <reason>Trivial scope — ≤3 files, no schema, no new auth, no unknowns. Deep research would waste tokens on a clear target.</reason>
          <skip-phase ref="research" reason="Trivial scope doesn't warrant full research pipeline."/>
          <skip-phase ref="project-overview" reason="Trivial scope — overview not needed."/>
          <skip-phase ref="external-pattern-research" reason="Trivial scope — no external patterns needed."/>
          <output-size>target 200-400 lines in discovery.md</output-size>
          <question-limit>max 3 questions</question-limit>
          <flow>Continue to phase 2 (precision-question-bank) directly, then interactive-qa, then save.</flow>
        </if>
        <if condition="complexity=simple">
          <reason>Simple scope — clear boundaries, minimal unknowns. Skip external patterns but run full code research.</reason>
          <skip-phase ref="external-pattern-research" reason="Simple scope — implementation patterns are already clear from code."/>
          <output-size>target 400-700 lines in discovery.md</output-size>
          <question-limit>max 7 questions</question-limit>
          <flow>Continue to phase 1 (research) skipping external-pattern-research, then full Q&A and save.</flow>
        </if>
        <else>
          <reason>Complex scope — treat with full research pipeline.</reason>
          <output-size>target 700-1200+ lines in discovery.md</output-size>
          <question-limit>max 12 questions</question-limit>
          <flow>Continue to phase 1 (research) with all sub-phases, then full Q&A and save.</flow>
        </else>
      </routing>

      <evidence-record>
        <step>Log the classification decision in discovery.md under a <complexity_routing> section:
          - Complexity level
          - Heuristic values (file-count, api-surface, schema-change, new-auth, unclear-dimensions)
          - Rationale for level assignment
          - Phases skipped (if any)
          - Budget/target output size
        </step>
      </evidence-record>
    </phase>

    <phase id="0.6" name="project-tooling-detection" silent="true">
      <task>Detect project language, test framework, typecheck command, test command, and build command from manifest files. Results are stored in discovery.md for downstream commands to use.</task>

      <principle>All downstream commands (tasks, implement, review, PR, validate, release) read the detected commands from discovery.md. This is the single source of truth for language-specific tooling.</principle>

      <manifest-check-order>
        <item file="Cargo.toml">Rust → `cargo test`, `cargo check`, `cargo build`</item>
        <item file="go.mod">Go → `go test ./...`, `go vet ./...`, `go build ./...`</item>
        <item file="pyproject.toml">Python → `pytest`, `mypy .` (or `pyright`), `python -m build`</item>
        <item file="setup.py">Python (fallback if no pyproject.toml)</item>
        <item file="package.json">Check for tsconfig.json → TypeScript or JavaScript. Check devDependencies for test framework: vitest, jest, mocha, ava. Determine package manager from lockfile: bun.lock → bun, pnpm-lock.yaml → pnpm, yarn.lock → yarn, package-lock.json → npm</item>
        <item file="build.gradle" or="build.gradle.kts">Java/Kotlin (Gradle) → `./gradlew test`, `./gradlew build`</item>
        <item file="pom.xml">Java (Maven) → `mvn test`, `mvn compile`</item>
        <item file="mix.exs">Elixir → `mix test`</item>
        <item file="Gemfile">Ruby → `bundle exec rspec`, `bundle exec rubocop`</item>
        <item file="go.work" or="go.mod">Go workspace</item>
        <item file="Cargo.toml">Rust workspace</item>
        <item file="Makefile">Generic — check for common targets: test, check, build</item>
      </manifest-check-order>

      <detection-steps>
        <step tool="Bash">Check project root for manifest files in priority order. Use `test -f` for each manifest. First match determines primary language.</step>
        <step tool="Bash">If package.json found, read it to extract test script and devDependencies for test framework detection. Check for tsconfig.json to confirm TypeScript.</step>
        <step tool="Bash">If pyproject.toml found, grep for [tool.pytest.ini_options] to confirm pytest, [tool.mypy] for mypy.</step>
        <step tool="Bash">If Cargo.toml found, check for [dev-dependencies] test frameworks.</step>
        <step tool="Bash">Determine test command: check package.json scripts.test, or Cargo.toml for cargo test, or pyproject.toml for pytest. Fallback to language default.</step>
        <step tool="Bash">Determine typecheck command: tsconfig.json → tsc --noEmit, pyproject.toml with mypy → mypy ., Cargo.toml → cargo check, go.mod → go vet ./...</step>
        <step tool="Bash">Determine build command: package.json scripts.build, Cargo.toml → cargo build, go.mod → go build ./..., pyproject.toml → python -m build</step>
        <step tool="Bash">Check lockfiles to determine package manager: bun.lock → bun, pnpm-lock.yaml → pnpm, yarn.lock → yarn, package-lock.json → npm</step>
      </detection-steps>

      <output>
        Append the following to discovery.md:
        ```xml
        <project_commands>
          <language>detected language</language>
          <package_manager>detected package manager or n/a</package_manager>
          <test_framework>detected test framework</test_framework>
          <typecheck>detected typecheck command</typecheck>
          <test>detected test command</test>
          <build>detected build command</build>
        </project_commands>
        ```
      </output>

      <rules>
        <rule>Always include the <project_commands> section — every downstream command depends on it.</rule>
        <rule>If detection fails for any field, set it to "unknown" rather than guessing. The agent at implement time can re-detect.</rule>
        <rule>Use the package manager when constructing commands (e.g., `npx` for npm, `bun` for bun, `pnpm` for pnpm).</rule>
        <rule>For test command, prefer the runner directly with file paths (e.g., `npx vitest run`, `pytest`, `cargo test`) rather than the package manager script wrapper. This makes targeted test execution easier.</rule>
        <rule>If the project has a Makefile with test/check/build targets, prefer those over direct commands.</rule>
      </rules>

      <!-- ─────────────────────────────────────────────────────────────────
           CANONICAL LANGUAGE MAP
           All downstream commands (prd-review, prd-release, prd-implement,
           prd-validate, prd-pr) MUST refer to this table instead of
           duplicating language→manifest or language→tooling mappings.
           ───────────────────────────────────────────────────────────────── -->
      <language-map id="canonical">
        <lang id="typescript">
          <version-manifest>package.json</version-manifest>
          <typecheck>tsc --noEmit (or package.json "typecheck"/"type-check" script)</typecheck>
          <extensions>.ts  .tsx</extensions>
        </lang>
        <lang id="javascript">
          <version-manifest>package.json</version-manifest>
          <typecheck>eslint . (or package.json "lint" script)</typecheck>
          <extensions>.js  .mjs  .cjs  .jsx</extensions>
        </lang>
        <lang id="python">
          <version-manifest>pyproject.toml  or  setup.py  or  setup.cfg</version-manifest>
          <typecheck>mypy .  or  ruff check .  or  pyright</typecheck>
          <extensions>.py</extensions>
        </lang>
        <lang id="rust">
          <version-manifest>Cargo.toml</version-manifest>
          <typecheck>cargo clippy -- -D warnings  or  cargo check</typecheck>
          <extensions>.rs</extensions>
        </lang>
        <lang id="go">
          <version-manifest>go.mod  (no version field — version is tracked via git tags only)</version-manifest>
          <typecheck>golangci-lint run  or  go vet ./...</typecheck>
          <extensions>.go</extensions>
        </lang>
        <lang id="java">
          <version-manifest>build.gradle  or  pom.xml</version-manifest>
          <typecheck>./gradlew build  or  mvn compile</typecheck>
          <extensions>.java</extensions>
        </lang>
        <lang id="kotlin">
          <version-manifest>build.gradle.kts  or  build.gradle</version-manifest>
          <typecheck>./gradlew build</typecheck>
          <extensions>.kt  .kts</extensions>
        </lang>
        <lang id="ruby">
          <version-manifest>Gemfile  /  *.gemspec</version-manifest>
          <typecheck>bundle exec srb tc  (Sorbet)  or  bundle exec rubocop --no-color</typecheck>
          <extensions>.rb</extensions>
        </lang>
        <lang id="csharp">
          <version-manifest>*.csproj</version-manifest>
          <typecheck>dotnet build --no-restore</typecheck>
          <extensions>.cs</extensions>
        </lang>
        <lang id="elixir">
          <version-manifest>mix.exs</version-manifest>
          <typecheck>mix dialyzer  or  mix compile --warnings-as-errors</typecheck>
          <extensions>.ex  .exs</extensions>
        </lang>
      </language-map>

      <!-- ─────────────────────────────────────────────────────────────────
           CANONICAL DISCOVERY-READ PATTERN
           Copy this exact instruction block into any downstream command
           that needs to consume project_commands. Do NOT write new ad-hoc
           failure messages — use this wording verbatim so behavior is
           consistent across the pipeline.
           ───────────────────────────────────────────────────────────────── -->
      <discovery-read-pattern>
        Read the `<project_commands>` section from `{prd_folder}/../discovery.md`.

        For language-to-manifest or language-to-tooling lookups, refer to the
        `<language-map id="canonical">` in prd-discover.md phase 0.6 — do not
        duplicate those mappings inline.

        **Missing or incomplete discovery.md (standard failure response):**
        If discovery.md is absent, or the required field is set to "unknown" or
        is empty, do NOT attempt inline re-detection. Instead emit:

        WARN · SETUP · (no file):0
          ✗  discovery.md missing or `<{field}>` is unknown
          ✓  Run `/prd-discover {prd_folder}` first, then re-run this command.

        Then skip the step that required the missing field.
      </discovery-read-pattern>
    </phase>

    <phase id="1" name="research" silent="true">
      <task>Understand current system, likely affected areas, and external implementation patterns before Q&A.</task>

      <verification-protocol priority="critical">
        Maintain a verified-names list as you research: `name | tool+query | file:line`. Only names on this list may appear later. Names not found via a tool ("typical" names from memory don't count) get marked UNVERIFIED, not stated as fact.
      </verification-protocol>

      <project-overview>
        <step tool="mcp__serena__get_symbols_overview">
          Record framework, language, routing, DB layer, auth, modules, tests, shared services, conventions.
        </step>
      </project-overview>

      <related-code>
        <step tool="mcp__serena__find_symbol">
          Search symbols related to feature domain and synonyms.
        </step>
        <step tool="mcp__serena__find_referencing_symbols">
          For relevant symbols, identify callers, callees, dependencies, and adjacent modules.
        </step>
        <step tool="mcp__octocode__localSearchCode">
          Search exact feature terms, synonyms, domain nouns, route names, model names, UI terms, and workflow terms.
        </step>
        <step tool="mcp__octocode__localGetFileContent">
          Read the 2-5 most relevant files, prioritizing files that already implement similar workflows.
        </step>
      </related-code>

      <semantic-search>
        <step tool="mcp__semble__search">
          Search conceptually related behavior, not just exact keywords.
        </step>
        <step tool="mcp__semble__find_related">
          Find adjacent concepts and reused patterns.
        </step>
      </semantic-search>

      <external-pattern-research>
        <step tool="mcp__context7__resolve-library-id">
          Resolve relevant libraries only if the feature clearly involves them.
        </step>
        <step tool="mcp__context7__query-docs">
          Fetch targeted docs for relevant APIs and patterns.
        </step>
        <step tool="mcp__octocode__githubSearchCode">
          Find 2-3 real implementations for non-trivial domain patterns.
        </step>
      </external-pattern-research>

      <feature-mode-detection>
        <detect name="feature_mode">
          <value>new</value>
          <value>enhancement</value>
        </detect>
        <rule>
          Set enhancement if input includes improve, enhance, update, refactor, change, redesign, extend, add to,
          or if research finds substantial existing implementation of the same capability.
        </rule>
      </feature-mode-detection>

      <blast-radius-map>
        <task>Before asking the user anything, identify what this feature might affect. Write "none found" rather than a plausible guess where evidence is thin.</task>
        <collect>
          <area name="api">Existing routes/endpoints that may change or be reused.</area>
          <area name="db">Tables/models/columns touched or likely needed.</area>
          <area name="auth">Authentication, authorization, public/private access, permissions.</area>
          <area name="ui">Pages/components/navigation states likely affected.</area>
          <area name="workflow">State machines, approval flows, jobs, events, notifications.</area>
          <area name="integrations">Email, storage, webhooks, external APIs, queue/background jobs.</area>
          <area name="tests">Existing tests that would catch regressions.</area>
          <area name="data-migration">Existing data that must survive.</area>
          <area name="security">Tokens, PII, permissions, abuse paths, rate limits.</area>
        </collect>
      </blast-radius-map>

      <if condition="feature_mode=enhancement">
        <enhancement-research>
          <collect>
            <contracts>Current API routes, request/response shapes, exported types, UI props, DB fields.</contracts>
            <dependents>All known callers, imports, pages, jobs, tests, integrations.</dependents>
            <data-at-stake>Existing persisted data that must not be lost or silently reinterpreted.</data-at-stake>
            <current-behavior>What users can do today and what must continue to work.</current-behavior>
            <rollback-risk>What would make rollback unsafe.</rollback-risk>
          </collect>
        </enhancement-research>
      </if>
    </phase>

    <phase id="2" name="precision-question-bank">
      <task>Create only high-signal questions based on research.</task>

      <question-ranking>
        <priority order="1">Questions that prevent wrong architecture.</priority>
        <priority order="2">Questions that prevent breaking existing behavior.</priority>
        <priority order="3">Questions that affect data model or migration.</priority>
        <priority order="4">Questions that affect security/access control.</priority>
        <priority order="5">Questions that affect UX or rollout.</priority>
        <priority order="6">Questions that affect success metrics or scope boundaries.</priority>
      </question-ranking>

      <categories>
        <intent>
          Clarify what the feature is, who it is for, and what it explicitly is not.
        </intent>
        <blast_radius>
          Clarify which affected areas are allowed to change and which must remain stable.
        </blast_radius>
        <integration>
          Clarify how it attaches to existing files, flows, routes, services, and components.
        </integration>
        <data_state>
          Clarify lifecycle, persistence, migration, reversibility, retention, and audit.
        </data_state>
        <security_access>
          Clarify permissions, tokens, privacy, abuse controls, and public/private boundaries.
        </security_access>
        <ux_flow>
          Clarify user journey, error states, notifications, and operational visibility.
        </ux_flow>
        <external_pattern_choice>
          Use external research to present 2-3 implementation alternatives when relevant.
        </external_pattern_choice>
        <compat condition="feature_mode=enhancement">
          Clarify frozen contracts, rollout strategy, rollback, and migration safety.
        </compat>
        <success>
          Clarify measurable success and launch criteria.
        </success>
      </categories>

      <question-quality-rules>
        <rule>Never ask "Should we build X?" if X already exists in code.</rule>
        <rule>Never ask a vague question when a file/symbol-specific question is possible.</rule>
        <rule>Every question must include why it matters for the PRD/spec.</rule>
        <rule>For architecture decisions, present alternatives and tradeoffs.</rule>
        <rule>Do not dump the full question bank; ask one at a time.</rule>
        <rule>Any name referenced in a question must already be on the verified-names list.</rule>
      </question-quality-rules>
    </phase>

    <phase id="3" name="interactive-qa">

      <opening type="new">
        ## 🔍 Research Complete

        I researched the codebase and external implementation patterns for "{feature}".

        **Relevant code found:** (file:line)
        - {file/symbol} — {what it does}
        - {file/symbol} — {what it does}

        **Gaps found:**
        - {missing capability}
        - {missing integration}

        **Potential blast radius:**
        - API: {affected routes or "none found"}
        - Data: {tables/models likely affected or "none found"}
        - Auth/Security: {auth or permission areas}
        - UI: {pages/components likely affected}
        - Integrations: {email/storage/jobs/etc}

        **External pattern reference:**
        - {pattern/alternative found through Context7/GitHub}

        I need to clarify {N} decisions before writing discovery.md.
        I’ll ask one question at a time.

        ---
        **Complexity:** {trivial|simple|complex} — {one-line reason}. {Question budget}.
        **Question 1 of {N}:** {highest-impact grounded question}
      </opening>

      <opening type="enhancement">
        ## 🔍 Research Complete — Enhancement Detected

        I found existing behavior related to "{feature}", so this is an enhancement.

        **Current implementation:** (file:line)
        - {file path} — {current behavior}
        - {file path} — {current behavior}

        **Known dependents:**
        - {caller/component/route/test} — {how it depends on current behavior}

        **Current data at stake:**
        - {table/column} — {why it must be preserved}

        **Blast radius:**
        - API contracts: {contracts likely affected}
        - DB/data: {models/columns likely affected}
        - UI/users: {current surfaces/users affected}
        - Tests: {existing tests likely affected}
        - Rollback risk: {risk}

        ⚠️ Compatibility decisions come first.

        **Complexity:** {trivial|simple|complex} — {one-line reason}. {Question budget}.

        ---
        **Question 1 of {N}:**
        The existing {contract/route/component/function} is currently used by {callers}.
        Must it remain fully functional during rollout, or can it be temporarily broken during transition?
      </opening>

      <rules>
        <rule>Ask one question at a time.</rule>
        <rule>After each answer, summarize the decision and implication.</rule>
        <rule>If answer resolves later questions, skip them.</rule>
        <rule>If answer is ambiguous, ask a clarification immediately.</rule>
        <rule>Ground questions in research findings.</rule>
        <rule>Include alternatives for architectural choices.</rule>
        <rule>Ask design-for-isolation before finishing.</rule>
        <rule>Never state a code fact without a tool-verified source — re-check or ask instead.</rule>
      </rules>

      <response-template>
        ✅ Got it — {summary of answer}. This means {implication for discovery/PRD/spec}.

        **Question {n} of {total}:** {next grounded question}
      </response-template>

      <completion>
        ## ✅ Discovery Complete

        I have enough information to write discovery.md.

        **Feature summary:** {2-3 sentences}
        **Scope:** {in/out summary}
        **Key decisions:** {numbered list}
        **Technical direction:** {1-2 sentences based on current code and external research}

        Saving discovery file...
      </completion>
    </phase>

    <phase id="4" name="save-discovery">
      <path>docs/prd/{feature-slug}-{dd-mm-yyyy}/discovery.md</path>

      <template>
```md
---
feature: "{feature}"
date: "{dd-mm-yyyy}"
status: "Discovery complete — ready for PRD"
mode: "{new|enhancement}"
---

<raw_input>{$ARGUMENTS}</raw_input>

<research_summary>
  <codebase>
    <relevant_files>
      <!-- file path | symbol | current behavior | relevance -->
    </relevant_files>
    <existing_patterns>
      <!-- pattern | where found | how it should influence PRD/spec -->
    </existing_patterns>
    <gaps>
      <!-- missing capability or missing integration -->
    </gaps>
  </codebase>

  <external_patterns>
    <!-- library/docs/github pattern | source | relevance | caution -->
  </external_patterns>
</research_summary>

<verified_names>
  <!-- name | source tool+query | file:line — every name used above/below must appear here, or be UNVERIFIED -->
</verified_names>

<complexity_routing>
  <level><!-- trivial | simple | complex --></level>
  <heuristics>
    <file_count><!-- number --></file_count>
    <api_surface><!-- yes | no --></api_surface>
    <schema_change><!-- yes | no --></schema_change>
    <new_auth><!-- yes | no --></new_auth>
    <unclear_dimensions><!-- yes | no --></unclear_dimensions>
  </heuristics>
  <rationale><!-- why this level was chosen --></rationale>
  <skipped_phases><!-- phases skipped due to complexity --></skipped_phases>
  <output_budget><!-- target lines --></output_budget>
</complexity_routing>

<blast_radius>
  <api><!-- existing/new routes likely affected, or "none found" --></api>
  <database><!-- tables/models/columns/data migrations, or "none found" --></database>
  <auth_security><!-- auth, permissions, tokens, PII, rate limits --></auth_security>
  <ui><!-- pages/components/navigation affected --></ui>
  <workflow><!-- state machines, jobs, notifications, events --></workflow>
  <integrations><!-- email/storage/webhooks/external systems --></integrations>
  <tests><!-- tests that must be added or protected --></tests>
  <rollback><!-- what makes rollback safe/unsafe --></rollback>
</blast_radius>

<qa_record>
  <question id="1" category="{intent|blast_radius|integration|data|security|ux|compat|success}">
    <asked>{exact question}</asked>
    <answer>{user answer}</answer>
    <implication>{what this means for PRD/spec}</implication>
  </question>
</qa_record>

<decisions>
  <!-- numbered confirmed decisions -->
</decisions>

<scope>
  <in><!-- concrete included items --></in>
  <out><!-- explicitly excluded items --></out>
  <deferred><!-- future items intentionally not built now --></deferred>
</scope>

<enhancement_compat_map condition="mode=enhancement">
  <frozen_contracts>
    <!-- contract | current shape | dependents | must remain stable until -->
  </frozen_contracts>
  <data_preservation>
    <!-- table/field/data | preservation rule -->
  </data_preservation>
  <transition_strategy>{parallel run|versioned API|additive only|big bang}</transition_strategy>
  <rollback_condition>{what must remain true for safe rollback}</rollback_condition>
  <cutover_criteria>{criteria before old behavior can be removed}</cutover_criteria>
</enhancement_compat_map>

<isolation_notes>
  <!-- boundaries that allow independent implementation/testing -->
</isolation_notes>

<open_questions>
  <!-- unresolved decision | owner | due -->
</open_questions>

---

## Feature Idea
{$ARGUMENTS}

## Research Summary

### Complexity Routing
- **Level:** {trivial|simple|complex}
- **Heuristics:** file-count={N}, api-surface={yes|no}, schema-change={yes|no}, new-auth={yes|no}, unclear-dimensions={yes|no}
- **Rationale:** {why this level was chosen}
- **Skipped phases:** {phases skipped, or "none"}
- **Output budget:** {target lines}

### Codebase Findings
- ...

### External Pattern Findings
- ...

## Verified Names
| Name | Source | File:Line |
|---|---|---|

## Blast Radius
- API:
- Database:
- Auth/Security:
- UI:
- Workflow:
- Integrations:
- Tests:
- Rollback:

## Q&A Record
**Q1 — {Category}:** {question}  
**Answer:** {answer}  
**Implication:** {implication}

## Decisions Made
1. ...

## Scope
**In scope:**
- ...

**Out of scope:**
- ...

**Deferred:**
- ...

## Enhancement Compatibility Map
<!-- only if enhancement -->

## Design-for-Isolation Notes
- ...

## Open Questions
| # | Question | Owner | Due |
|---|---|---|---|
```
      </template>

      <final-output>
        ## 📁 Saved
        Discovery file written to:
        `docs/prd/{feature-slug}-{dd-mm-yyyy}/discovery.md`

        ## ▶️ Next Step
        /prd-write docs/prd/{feature-slug}-{dd-mm-yyyy}/discovery.md
      </final-output>
    </phase>

  </flow>

  <control>
    <priority>research evidence over assumptions</priority>
    <failure>if unknown, ask a precise question instead of guessing</failure>
    <critical>
      <rule>Do not write requirements in this command.</rule>
      <rule>Do not skip blast-radius mapping.</rule>
      <rule>Do not ask generic questions before code-grounded questions.</rule>
      <rule>Do not omit external pattern findings if they affect architecture.</rule>
      <rule>Never state a name that isn't in the verified-names list; mark UNVERIFIED instead of guessing or "fixing" spelling/casing.</rule>
      <rule>Enforce the semantic-research protocol: mcp__semble__search before mcp__octocode__localSearchCode or mcp__serena__find_symbol.</rule>
    </critical>
  </control>

</command>
