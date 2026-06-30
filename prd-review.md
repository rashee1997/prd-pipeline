---
description: "PRD Step 6/9 — Parallel code review with Ponytail discipline. Dispatches specialist reviewers, merges findings, deduplicates, sorts by severity."
argument-hint: "<path to prd-folder> [--layer {layer-name}]"
allowed-tools: mcp__semble, mcp__serena, mcp__octocode, Task, Bash
---

<command name="/prd-review">
 
   <execution>
     <follow_structure>strict</follow_structure>
     <treat_tags_as_semantic>true</treat_tags_as_semantic>
     <do_not_skip_phases>true</do_not_skip_phases>
     <do_not_assume>true</do_not_assume>
     <output_findings_only>true</output_findings_only>
   </execution>
 
   <system>
     <role>Review orchestrator</role>
     <principle>Gather → Dispatch → Deduplicate → Report</principle>
     <rules>
       <item>No praise</item>
       <item>No padding</item>
       <item>No invented findings</item>
       <item>No finding without file and line</item>
       <item priority="critical">mcp__semble is MANDATORY for all review code discovery — use mcp__semble__search to find relevant files by concept before reading them. See semantic-discovery in phase 1.</item>
       <item>Deduplicate before output</item>
       <item>CRITICAL and HIGH must block PR</item>
     </rules>
   </system>
 
   <input>
     <prd_folder>{$ARGUMENTS}</prd_folder>
     <layer_flag>{$LAYER_FLAG}</layer_flag>
     <outputs>merged review findings only</outputs>
   </input>

  <ponytail>
    <function>
      Review like a lazy senior engineer: remove unnecessary work, reject over-engineering,
      prefer existing codebase patterns, and recommend the smallest safe fix.
    </function>
    <ladder>
      <rung>Does this need to exist?</rung>
      <rung>Can existing code be reused?</rung>
      <rung>Can stdlib/runtime/platform solve it?</rung>
      <rung>Can an installed dependency solve it?</rung>
      <rung>Can it be one clear line?</rung>
      <rung>Only then: minimum safe code.</rung>
    </ladder>
    <never_lazy_about>
      <item>security</item>
      <item>auth</item>
      <item>input validation</item>
      <item>data loss</item>
      <item>accessibility</item>
      <item>compatibility contracts</item>
      <item>explicit PRD requirements</item>
      <item>tests proving non-trivial behavior</item>
    </never_lazy_about>
  </ponytail>

  <flow>

    <phase id="1" name="gather-context">
      <task>Collect changed files, static analysis/lint errors (or typecheck errors if the project has a type-checker), PRD/spec context, and file buckets.</task>

      <steps>
<phase id="1" name="gather-context">
      <task>Collect changed files, static analysis/lint errors (or typecheck errors if the project has a type-checker), PRD/spec context, and file buckets.</task>

      <steps>
        <step name="semantic-discovery">
          Use mcp__semble__search with 2-3 natural-language queries related to the feature/module being reviewed to find all relevant files. This is the most token-efficient path — do not skip.
        </step>
        <step name="layer-selection" condition="layer-flag-invoked">
          If --layer flag is provided, restrict focus and file buckets to the specified layer.
        </step>
        <step name="detect-base">
...
```bash
BASE=$(git remote show origin 2>/dev/null | grep "HEAD branch" | awk '{print $NF}')
[ -z "$BASE" ] && BASE=$(git branch -r 2>/dev/null | grep -E 'origin/(main|master)' | head -1 | sed 's|.*origin/||' | tr -d ' ')
[ -z "$BASE" ] && BASE="main"
git diff $(git merge-base HEAD $BASE)...HEAD --name-only
```
        </step>

        <step name="static-analysis">
          Resolve the static-analysis/lint/typecheck command using this priority order:

          Read the `<project_commands>` section from `{prd-folder}/../discovery.md`.
          Use the `<typecheck>` field value as the static-analysis command to run.

          Follow the **discovery-read-pattern** defined in prd-discover.md phase 0.6
          for failure handling if discovery.md is absent or `<typecheck>` is unknown.

          <on-fail>
            Record each error as:
            CRITICAL · TYPECHECK · file:line / ✗ {error} / ✓ Fix the type error before continuing.
          </on-fail>
        </step>

        <step name="read-context">
```bash
cat {prd-folder}/spec.md
cat {prd-folder}/prd.md
cat {prd-folder}/tasks/index.md
```
          <extract>
            <item>FR-* requirements</item>
            <item>API routes: method, path, auth, response shape</item>
            <item>schema models</item>
            <item>frozen contracts</item>
            <item>task completion state</item>
          </extract>
        </step>

        <step name="bucket-files">
          Using the changed-files list from the detect-base step above (do not re-run git diff),
          read `<language>` and `<package_manager>` from the `<project_commands>` section of
          `{prd-folder}/../discovery.md` to determine framework context. No manifest re-reading needed.
          Follow the **discovery-read-pattern** in prd-discover.md phase 0.6 for failure handling.

          Use `<language>` (as defined in the `<language-map id="canonical">` in prd-discover.md
          phase 0.6) to select the appropriate bucket rules below:

          Bucket each changed file using these language-agnostic signals applied after
          framework context is established. Every bucket that has zero matches must be noted as empty —
          any reviewer assigned to an empty bucket must self-report PASS with note
          "no applicable files in this change" rather than running with zero files.

          <bucket name="ROUTE_FILES">
            Files matching the detected framework's routing convention:
            - nextjs:   src/app/api/**/*  or  pages/api/**/*
            - react/node-server: routes/**/*  or  src/routes/**/*  or  *router*.*  or  *controller*.*
            - fastapi:  files containing "@router." or "APIRouter" or in a routers/ directory
            - django:   views.py files  or  *views/*  or  urls.py files
            - flask:    files containing "@app.route" or "@blueprint.route"
            - go:       files containing "http.Handle" or "r.Get/Post/Put/Delete" or in handlers/ directory
            - rust:     files containing ".route(" or "actix_web::web::" or axum route definitions
            - dotnet:   *Controller.cs files
            - ruby:     app/controllers/**/*.rb
            - unknown:  files in any path segment named api/, routes/, controllers/, handlers/, views/
          </bucket>

          <bucket name="ACTION_FILES">
            Server-action / mutation files by framework:
            - nextjs: files containing "use server"
            - django/flask/fastapi: files containing mutation endpoints (POST/PUT/PATCH/DELETE handlers)
            - other:  files whose path contains actions/, mutations/, or commands/
            If this concept does not apply to the detected framework, bucket is empty.
          </bucket>

          <bucket name="COMPONENT_FILES">
            UI component files for the detected frontend framework:
            - nextjs/react: changed **/*.tsx  or  **/*.jsx  (non-test)
            - vue:          changed **/*.vue
            - svelte:       changed **/*.svelte
            - angular:      changed *.component.ts
            - backend-only (fastapi/django/flask/go/rust/dotnet/ruby with no frontend manifest):
              bucket is empty
          </bucket>

          <bucket name="SERVICE_FILES">
            Business logic / service-layer files. Detect by actual directory structure present
            in this repo rather than assuming src/lib or src/modules. Look for directories named
            lib/, services/, service/, modules/, domain/, core/, internal/, pkg/, app/ (non-web)
            and classify changed files under those paths here.
          </bucket>

          <bucket name="SCHEMA_FILES">
            Schema / migration files for whatever data layer is present:
            - Prisma:    prisma/schema.prisma  or  prisma/migrations/**
            - Drizzle:   **/schema.ts  or  **/drizzle/**
            - Django:    **/models.py  or  **/migrations/*.py
            - Rails:     db/schema.rb  or  db/migrate/**
            - Go/SQLC:   **/*.sql  or  sqlc.yaml
            - Rust/SQLx: migrations/**  or  **/*.sql
            - dotnet EF: **/Migrations/**  or  *Context.cs
            - plain SQL:  *.sql  or  migrations/**/*.sql
          </bucket>

          <bucket name="TEST_FILES">
            Test files matching whatever test framework is detected:
            - Jest/Vitest (JS/TS):  *.test.ts, *.test.tsx, *.spec.ts, *.spec.tsx, __tests__/**
            - pytest (Python):      test_*.py, *_test.py, tests/**/*.py
            - Go:                   *_test.go
            - Rust:                 files containing #[cfg(test)] or in tests/ directory
            - RSpec (Ruby):         *_spec.rb, spec/**/*.rb
            - xUnit/NUnit (dotnet): *Tests.cs, *Test.cs
          </bucket>

          <bucket name="CONFIG_FILES">
            Config and env files — generic signals plus framework-specific config:
            - Always include: *.config.*, .env*, .env.*, *.env
            - nextjs detected:   next.config.*
            - vite detected:     vite.config.*
            - webpack detected:  webpack.config.*
            - Django detected:   settings.py, settings/**/*.py
            - Go detected:       *.yaml, *.toml config files at root
            - Rust detected:     Cargo.toml, build.rs
            - dotnet detected:   appsettings*.json, *.csproj
            - Ruby detected:     config/**/*.rb, database.yml
          </bucket>

          <bucket name="ALL_SOURCE_FILES">
            All changed source files in the primary language extension(s) detected above
            (replaces the former ALL_TS_FILES which hardcoded TypeScript).
            Example: .py for Python, .go for Go, .rs for Rust, .ts/.tsx for TypeScript.
          </bucket>
        </step>
      </steps>
    </phase>

    <phase id="2" name="dispatch-reviewers">
      <task>Run six specialist reviewers in parallel.</task>

      <common-agent-contract>
        You are a specialist code reviewer. Subagents have no MCP access — use Bash (grep, cat, etc.) for code inspection.
        Read only your assigned files.
        Output findings only. No praise. No padding.
        Format:
        SEVERITY · CATEGORY · path/to/file:LINE / ✗ problem / ✓ fix

        Severities: CRITICAL | HIGH | MEDIUM | LOW
        If clean: PASS · CATEGORY
      </common-agent-contract>

      <reviewer id="1" name="spec-compliance">
        <files>ALL_SOURCE_FILES + spec/prd excerpts</files>

        <ponytail-review>
          <rule>Do not ask for extra implementation beyond PRD/spec.</rule>
          <rule>If code implements the requirement with less machinery than spec implied but behavior matches, PASS.</rule>
          <rule>If missing behavior can be fixed by reusing an existing module, recommend reuse not new abstraction.</rule>
          <rule>Do not flag stylistic differences as spec failures.</rule>
        </ponytail-review>

        <checklist>
          <check>Every FR-* is implemented in changed files.</check>
          <check>Every specified route exists.</check>
          <check>Route response shape matches spec.</check>
          <check>Protected routes call auth before logic.</check>
          <check>Frozen contracts remain intact.</check>
          <check>New schema models exist.</check>
          <check>Faithfulness — code behavior matches acceptance criteria exactly, not just "passes tests." Code that passes tests but does something different from the acceptance criteria is DRIFT, not done.</check>
        </checklist>

        <findings>
          <finding condition="FR missing">CRITICAL · SPEC · (no file):0 / ✗ FR-{id} not implemented / ✓ Implement per spec</finding>
          <finding condition="faithfulness drift">CRITICAL · SPEC · {file}:{line} / ✗ Code behavior drifts from acceptance criteria — passes tests but does something different from stated intent / ✓ Reconcile code behavior with acceptance criteria; criteria are truth</finding>
          <finding condition="route missing">CRITICAL · SPEC · {path}:0 / ✗ Route {METHOD} {path} missing / ✓ Create per spec</finding>
          <finding condition="bad response shape">HIGH · SPEC · {file}:{line} / ✗ Response shape deviates from spec / ✓ Match spec shape</finding>
          <finding condition="auth missing">CRITICAL · SECURITY · {file}:{line} / ✗ Auth required by spec but no authentication check before business logic / ✓ Call the project's auth middleware or session-check function before executing logic</finding>
          <finding condition="frozen contract changed">CRITICAL · COMPAT · {file}:{line} / ✗ Frozen contract {name} broken / ✓ Restore signature; make additive change</finding>
          <finding condition="model missing">CRITICAL · SPEC · {schema-file}:0 / ✗ Model/schema {name} missing / ✓ Add per spec</finding>
        </findings>
      </reviewer>

      <reviewer id="2" name="security">
        <files>ROUTE_FILES + ACTION_FILES + CONFIG_FILES</files>

        <ponytail-review>
          <rule>Never accept “minimal” code that skips auth, validation, ownership checks, or rate limits.</rule>
          <rule>Prefer existing auth/validation/rate-limit helpers already in the project over new security wrappers.</rule>
          <rule>Smallest safe fix beats large security framework changes.</rule>
          <rule>Do not recommend new dependencies unless existing stack cannot cover the risk.</rule>
        </ponytail-review>

        <checklist>
          <check>Authentication/session check before DB/business logic</check>
          <check>IDOR ownership scope — results filtered to current user/org</check>
          <check>Input validation before DB writes/queries (using whatever validation library/pattern the project uses — Zod, Pydantic, marshmallow, Bean Validation, ActiveRecord validations, etc.)</check>
          <check>Cryptographically secure random generation for secrets/tokens (not a weak PRNG)</check>
          <check>No sensitive values in logs</check>
          <check>No secrets leaked to the client via environment variable naming conventions (NEXT_PUBLIC_ in Next.js, VITE_ in Vite, PUBLIC_ in SvelteKit, etc.) or serialized into client-side bundles/responses</check>
          <check>Server-side mutation handlers authenticate before mutating (applies to Next.js server actions, tRPC mutations, Django views, FastAPI route handlers, etc.)</check>
          <check>Rate limits on public mutation endpoints</check>
          <check>No source maps exposed in production builds (applies to any build system that emits source maps — Next.js, Vite, webpack, esbuild, etc.)</check>
        </checklist>

        <findings>
          <finding>CRITICAL · SECURITY · {file}:{line} / ✗ Business logic executes before authentication check / ✓ Move the project's auth middleware or session-check call before any logic</finding>
          <finding>CRITICAL · SECURITY · {file}:{line} / ✗ No ownership filter on {model} lookup / ✓ Scope query to current user/org</finding>
          <finding>CRITICAL · SECURITY · {file}:{line} / ✗ Unvalidated input reaches database / ✓ Validate input with the project's validation library before the DB call</finding>
          <finding>CRITICAL · SECURITY · {file}:{line} / ✗ Weak PRNG used for secret or token generation / ✓ Use a cryptographically secure source: crypto.randomBytes() in Node.js, secrets module in Python, crypto/rand in Go, OsRng in Rust, SecureRandom in Java</finding>
          <finding>HIGH · SECURITY · {file}:{line} / ✗ Sensitive value logged / ✓ Remove or log only non-sensitive identifier</finding>
          <finding>CRITICAL · SECURITY · {file}:{line} / ✗ Secret exposed to client via public env var or serialized into client bundle / ✓ Keep secret server-side; use the framework's server-only env pattern</finding>
          <finding>HIGH · SECURITY · {file}:{line} / ✗ Public mutation endpoint lacks rate limit / ✓ Apply existing rate-limit pattern from this project</finding>
          <finding>HIGH · SECURITY · {file}:{line} / ✗ Source maps enabled in production build / ✓ Disable source map output in the project's build config</finding>
        </findings>
      </reviewer>

      <reviewer id="3" name="performance">
        <files>ROUTE_FILES + SERVICE_FILES + COMPONENT_FILES + SCHEMA_FILES</files>

        <ponytail-review>
          <rule>Do not propose premature optimization.</rule>
          <rule>Flag only performance issues visible in changed code.</rule>
          <rule>Prefer fewer queries, smaller selects, and existing pagination patterns.</rule>
          <rule>Recommend the smallest fix that removes the bottleneck.</rule>
        </ponytail-review>

        <checklist>
          <backend>
            <check>N+1 DB queries — per-row query inside a loop instead of a batch query</check>
            <check>Unbounded collection query — no pagination/limit applied to a query that could return large result sets</check>
            <check>Wide query — fetching all columns/fields when only a subset is used</check>
            <check>Sequential independent queries that could run concurrently</check>
            <check>Missing index on a field used in a filter/WHERE clause</check>
            <check>Deeply nested eager-loaded associations (ORM include/join depth > 2 levels without justification)</check>
          </backend>
          <frontend>
            <!-- Apply only if COMPONENT_FILES is non-empty for this repo -->
            <check>List rendered with index as key on a dynamic/reorderable list</check>
            <check>Inline object/array literals passed as props causing unnecessary re-renders on every parent render</check>
            <check>Large/heavy dependency imported synchronously when lazy/dynamic import would suffice</check>
            <check>Direct mutation of shared state instead of producing a new value</check>
          </frontend>
        </checklist>

        <findings>
          <finding>HIGH · PERFORMANCE · {file}:{line} / ✗ N+1 query in loop / ✓ Batch into a single query with an IN clause or the ORM's batch API (e.g. Prisma findMany where id in ids, SQLAlchemy in_(), Go sqlx.In, etc.)</finding>
          <finding>HIGH · PERFORMANCE · {file}:{line} / ✗ Unbounded collection query — no limit / ✓ Add pagination or a default limit using the ORM/query-builder's equivalent: LIMIT in SQL, limit() in most ORMs, take in Prisma, offset/limit in SQLAlchemy, Limit in Go query builders</finding>
          <finding>MEDIUM · PERFORMANCE · {file}:{line} / ✗ Query fetches unused columns / ✓ Project only required fields using SELECT / ORM select() / GraphQL field selection</finding>
          <finding>MEDIUM · PERFORMANCE · {file}:{line} / ✗ Independent queries run sequentially / ✓ Run concurrently — Promise.all in JS, asyncio.gather in Python, goroutines in Go, join/rayon in Rust</finding>
          <finding>HIGH · PERFORMANCE · {file}:{line} / ✗ Filtered field lacks a DB index / ✓ Add an index on {field} in the schema/migration file</finding>
          <finding>MEDIUM · PERFORMANCE · {file}:{line} / ✗ List uses index as key — breaks reconciliation on reorder / ✓ Use a stable unique id as key</finding>
          <finding>MEDIUM · PERFORMANCE · {file}:{line} / ✗ Inline object/array prop recreates reference each render / ✓ Hoist to module scope or memoize with the framework's memoization primitive</finding>
          <finding>HIGH · PERFORMANCE · {file}:{line} / ✗ Shared state mutated directly / ✓ Produce a new value/reference instead of mutating in place</finding>
        </findings>
      </reviewer>

      <reviewer id="4" name="type-safety">
        <files>ALL_SOURCE_FILES</files>

        <dispatch-logic>
  Before running this reviewer, detect the primary language of ALL_SOURCE_FILES from file extensions.
  Then apply a generic type-safety review using the checklist below.

  <ponytail-review>
    <rule>Do not demand elaborate type architecture for local/simple types.</rule>
    <rule>Prefer inferred internal types when safe; require explicit exported boundaries.</rule>
    <rule>Reject any/unsafe casts/coercions and unguarded null/unknown access.</rule>
    <rule>Small type guard beats broad assertion.</rule>
    <rule>Error returns must never be silently discarded.</rule>
  </ponytail-review>

  <checklist>
    <check>Type/lint errors suppressed without justification (@ts-ignore, # type: ignore, @SuppressWarnings, etc.)</check>
    <check>Unsafe casts or forced coercions without guard</check>
    <check>Error returns silently discarded</check>
    <check>any/interface{}/untyped usage where concrete type is available</check>
    <check>Exported functions missing return type / type hints</check>
    <check>Unchecked null/optional access</check>
  </checklist>

  <findings>
    <finding>HIGH · TYPESAFETY · {file}:{line} / ✗ Type/lint error suppressed without justification / ✓ Fix root cause or add explanation comment</finding>
    <finding>HIGH · TYPESAFETY · {file}:{line} / ✗ Unsafe cast or forced coercion / ✓ Add runtime type guard before cast</finding>
    <finding>HIGH · TYPESAFETY · {file}:{line} / ✗ Error silently ignored / ✓ Handle or propagate the error</finding>
    <finding>MEDIUM · TYPESAFETY · {file}:{line} / ✗ any/interface{}/untyped used where concrete type available / ✓ Use concrete type or defined interface</finding>
    <finding>MEDIUM · TYPESAFETY · {file}:{line} / ✗ Exported API missing type signature / ✓ Add explicit return/parameter types</finding>
    <finding>HIGH · TYPESAFETY · {file}:{line} / ✗ Unchecked null/optional access / ✓ Add guard or use safe access pattern</finding>
    <finding>INFO · TYPESAFETY · (no file):0 / ✗ Language-specific type-safety checklist not defined for {detected language} / ✓ Extend prd-review.md reviewer #4 for this language</finding>
  </findings>
</dispatch-logic>
      </reviewer>

      <reviewer id="5" name="quality-architecture">
        <files>ALL_SOURCE_FILES</files>

        <ponytail-review>
          <rule>Prefer deletion over abstraction.</rule>
          <rule>Do not create shared utilities unless duplication is real and current.</rule>
          <rule>Reject clever architecture that is not required by PRD/spec.</rule>
          <rule>Recommend boring, local, minimal fixes.</rule>
          <rule>Do not flag a short clear function just because it lacks layers.</rule>
        </ponytail-review>

        <checklist>
          <quality>
            <check>debug logs</check>
            <check>TODO/FIXME/HACK without issue reference</check>
            <check>repeated magic literals</check>
            <check>function body over 40 lines</check>
            <check>duplicate logic</check>
            <check>suppression comment</check>
          </quality>
          <architecture>
            <check>Shared/common component or module imports code that is specific to one feature or domain module</check>
            <check>Cross-module imports violate the repo's established boundary conventions</check>
            <check>New DB client instance created outside the project's established DB singleton or connection pool (e.g. new PrismaClient, new mongoose.Connection, creating a new SQLAlchemy engine, opening a new sql.DB, etc.)</check>
            <check>API handler/route calls another internal API endpoint over HTTP instead of calling the service layer directly</check>
            <check>Client-side code imports a module marked or known to be server-only (e.g. server-only imports, modules with direct DB/secret access)</check>
            <check>Design token or global style constant defined inline inside a component instead of the shared design system / token file</check>
          </architecture>
        </checklist>

        <findings>
          <finding>MEDIUM · QUALITY · {file}:{line} / ✗ Debug log left in code / ✓ Remove</finding>
          <finding>LOW · QUALITY · {file}:{line} / ✗ Unresolved TODO/FIXME/HACK / ✓ Resolve or link issue</finding>
          <finding>LOW · QUALITY · {file}:{line} / ✗ Repeated literal / ✓ Extract named constant</finding>
          <finding>MEDIUM · QUALITY · {file}:{line} / ✗ Function does too much / ✓ Extract named helper</finding>
          <finding>MEDIUM · QUALITY · {file}:{line} / ✗ Duplicated logic / ✓ Reuse existing helper or extract one</finding>
          <finding>MEDIUM · QUALITY · {file}:{line} / ✗ Lint suppression comment without justification / ✓ Fix root issue or add explanation</finding>
          <finding>HIGH · ARCHITECTURE · {file}:{line} / ✗ Shared module imports domain-specific code / ✓ Invert dependency — pass via parameter or move shared logic to a neutral location</finding>
          <finding>CRITICAL · ARCHITECTURE · {file}:{line} / ✗ New DB client instance created outside the project's connection singleton / ✓ Import and use the project's established DB singleton or connection pool</finding>
          <finding>MEDIUM · ARCHITECTURE · {file}:{line} / ✗ API handler fetches another internal API endpoint over HTTP / ✓ Call the service/domain layer directly</finding>
          <finding>CRITICAL · ARCHITECTURE · {file}:{line} / ✗ Client-side code imports a server-only module / ✓ Move the logic to the server layer or expose it through an API boundary</finding>
        </findings>
      </reviewer>

      <reviewer id="6" name="test-coverage">
        <files>TEST_FILES + ALL_SOURCE_FILES</files>

        <ponytail-review>
          <rule>Do not demand massive test suites for trivial one-liners.</rule>
          <rule>Non-trivial logic needs the smallest runnable check that fails if behavior breaks.</rule>
          <rule>Prefer focused tests over broad fixtures.</rule>
          <rule>Require regression tests for compatibility and security-sensitive behavior.</rule>
        </ponytail-review>

        <checklist>
          <check>new service/API/utility has corresponding test unless trivial</check>
          <check>tests include at least one error/rejection path</check>
          <check>assertions verify values, not just existence</check>
          <check>no skipped tests</check>
          <check>compat-sensitive change has frozen contract regression test</check>
        </checklist>

        <findings>
          <finding>HIGH · TESTS · {file}:0 / ✗ No test for new non-trivial module / ✓ Add focused test with happy path and 2 errors</finding>
          <finding>MEDIUM · TESTS · {file}:{line} / ✗ Only happy path tested / ✓ Add expected failure case</finding>
          <finding>MEDIUM · TESTS · {file}:{line} / ✗ Assertion too weak / ✓ Assert exact output or shape</finding>
          <finding>HIGH · TESTS · {file}:{line} / ✗ Skipped test / ✓ Unskip and fix or delete invalid test</finding>
          <finding>CRITICAL · TESTS · {file}:0 / ✗ Frozen contract lacks regression test / ✓ Add contract test before implementation changes</finding>
        </findings>
      </reviewer>

    </phase>

    <phase id="3" name="merge-findings">
      <task>Wait for reviewers, parse findings, deduplicate, sort.</task>

      <parse-format>
        SEVERITY · CATEGORY · file:line / ✗ problem / ✓ fix
      </parse-format>

      <dedupe>
        <rule>Same problem pattern in multiple files: keep highest severity, append "(also in N other files)".</rule>
        <rule>If same issue appears in multiple agents, keep stricter category/severity.</rule>
        <rule>Discard PASS lines from final findings; use them only for summary.</rule>
      </dedupe>

      <sort>
        <order>CRITICAL, HIGH, MEDIUM, LOW</order>
        <within>group by CATEGORY</within>
        <typecheck_errors>always first</typecheck_errors>
      </sort>
    </phase>

    <phase id="4" name="output">
      <format>
```text
SEVERITY · CATEGORY · path/to/file:LINE
  ✗ {problem}
  ✓ {fix}
```
      </format>

      <summary>
```text
── Review complete ──────────────────────────────────────────────
  {N} critical · {N} high · {N} medium · {N} low
  Spec compliance : {PASS|FAIL — N issues}
  Security        : {PASS|FAIL — N issues}
  Performance     : {PASS|FAIL — N issues}
  Type safety     : {PASS|FAIL — N issues}
  Code quality    : {PASS|FAIL — N issues}
  Tests           : {PASS|FAIL — N issues}
  Architecture    : {PASS|FAIL — N issues}
─────────────────────────────────────────────────────────────────
```
      </summary>

      <after-review>
        <if condition="finding_count=0">
          ✅ Clean — run /prd-pr to open the pull request.
        </if>
        <if condition="finding_count>0">
          Fix findings, then rerun:
          /prd-review {prd-folder}

          Only run /prd-pr after zero findings.
          CRITICAL and HIGH must be resolved.
          MEDIUM and LOW may be deferred only with explicit user acceptance.
        </if>
      </after-review>
    </phase>

  </flow>

  <control>
    <priority>real defects over theoretical cleanup</priority>
    <failure>if no file:line, discard finding</failure>

    <hard-rules>
      <rule>One ✗ and one ✓ per finding</rule>
      <rule>No praise</rule>
      <rule>No "consider" or vague advice</rule>
      <rule>No invented findings</rule>
      <rule>File path and line required</rule>
      <rule>Deduplicate before output</rule>
      <rule>Typecheck/lint errors are CRITICAL</rule>
      <rule>Ponytail applies to every reviewer</rule>
      <rule>Do not recommend new abstraction unless duplication or boundary pressure is proven</rule>
      <rule>Do not recommend new dependency unless existing code, stdlib, and platform cannot solve it safely</rule>
      <rule>mcp__semble is mandatory for review code discovery — call mcp__semble__search to find relevant files by concept before reading them.</rule>
    </hard-rules>
  </control>

</command>