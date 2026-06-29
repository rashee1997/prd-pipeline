---
description: "PRD Step 6/7 — Parallel code review with Ponytail discipline. Dispatches specialist reviewers, merges findings, deduplicates, sorts by severity."
argument-hint: "<path to prd-folder>"
allowed-tools: Bash, mcp__serena, mcp__octocode, mcp__semble
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
      <item priority="critical">mcp__semble is MANDATORY for all review code discovery — use mcp__semble__search to find all relevant files before reading them. It is the most token-efficient path to finding changed code by concept.</item>
      <item>Deduplicate before output</item>
      <item>CRITICAL and HIGH must block PR</item>
    </rules>
  </system>

  <input>
    <prd_folder>{$ARGUMENTS}</prd_folder>
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
      <task>Collect changed files, typecheck/lint errors, PRD/spec context, and file buckets.</task>

      <steps>
        <step name="semantic-discovery">
          Use mcp__semble__search with 2-3 natural-language queries related to the feature/module being reviewed to find all relevant files. This is the most token-efficient path — do not skip.
        </step>

        <step name="detect-base">
```bash
BASE=$(git remote show origin 2>/dev/null | grep "HEAD branch" | awk '{print $NF}')
[ -z "$BASE" ] && BASE=$(git branch -r 2>/dev/null | grep -E 'origin/(main|master)' | head -1 | sed 's|.*origin/||' | tr -d ' ')
[ -z "$BASE" ] && BASE="main"
git diff $(git merge-base HEAD $BASE)...HEAD --name-only
```
        </step>

        <step name="typecheck">
          Resolve the typecheck/lint command using this priority order:

          1. Read {prd-folder}/spec.md, {prd-folder}/tasks/index.md, and any task files for
             a documented typecheck, build, or lint command (same way prd-implement.md reads
             "the typecheck command from this task's acceptance criteria"). Use that command.

          2. If no command is documented there, detect the project type from manifest files
             in the repo root and run the appropriate native check:
```bash
# Detect project type and run appropriate check command
TYPECHECK_CMD=""

# Check manifest files in priority order
if [ -f "package.json" ]; then
  # Node/TypeScript project — prefer script from package.json
  if grep -q '"typecheck"' package.json 2>/dev/null; then
    RUNNER=$([ -f "bun.lock" ] || [ -f "bun.lockb" ] && echo "bun" || ([ -f "pnpm-lock.yaml" ] && echo "pnpm" || ([ -f "yarn.lock" ] && echo "yarn" || echo "npm")))
    TYPECHECK_CMD="$RUNNER run typecheck"
  elif grep -q '"type-check"' package.json 2>/dev/null; then
    RUNNER=$([ -f "bun.lock" ] || [ -f "bun.lockb" ] && echo "bun" || ([ -f "pnpm-lock.yaml" ] && echo "pnpm" || ([ -f "yarn.lock" ] && echo "yarn" || echo "npm")))
    TYPECHECK_CMD="$RUNNER run type-check"
  elif grep -q '"lint"' package.json 2>/dev/null; then
    RUNNER=$([ -f "bun.lock" ] || [ -f "bun.lockb" ] && echo "bun" || ([ -f "pnpm-lock.yaml" ] && echo "pnpm" || ([ -f "yarn.lock" ] && echo "yarn" || echo "npm")))
    TYPECHECK_CMD="$RUNNER run lint"
  elif [ -f "tsconfig.json" ]; then
    # TypeScript project without a named script — pick the installed runner
    if [ -f "bun.lock" ] || [ -f "bun.lockb" ]; then
      TYPECHECK_CMD="bun tsc --noEmit"
    elif [ -f "pnpm-lock.yaml" ]; then
      TYPECHECK_CMD="pnpm exec tsc --noEmit"
    elif [ -f "yarn.lock" ]; then
      TYPECHECK_CMD="yarn tsc --noEmit"
    else
      TYPECHECK_CMD="npx tsc --noEmit"
    fi
  fi
elif [ -f "pyproject.toml" ] || [ -f "setup.py" ] || [ -f "setup.cfg" ]; then
  # Python project
  if command -v mypy >/dev/null 2>&1; then
    TYPECHECK_CMD="mypy ."
  elif command -v ruff >/dev/null 2>&1; then
    TYPECHECK_CMD="ruff check ."
  elif command -v flake8 >/dev/null 2>&1; then
    TYPECHECK_CMD="flake8 ."
  fi
elif [ -f "go.mod" ]; then
  # Go project
  if command -v golangci-lint >/dev/null 2>&1; then
    TYPECHECK_CMD="golangci-lint run"
  else
    TYPECHECK_CMD="go vet ./..."
  fi
elif [ -f "Cargo.toml" ]; then
  # Rust project
  if command -v cargo-clippy >/dev/null 2>&1 || rustup component list --installed 2>/dev/null | grep -q clippy; then
    TYPECHECK_CMD="cargo clippy -- -D warnings"
  else
    TYPECHECK_CMD="cargo check"
  fi
elif ls *.csproj 2>/dev/null | head -1 | grep -q '.csproj'; then
  TYPECHECK_CMD="dotnet build --no-restore"
elif [ -f "Gemfile" ]; then
  # Ruby project
  if grep -q 'sorbet\|tapioca' Gemfile 2>/dev/null || [ -f ".sorbet" ]; then
    TYPECHECK_CMD="bundle exec srb tc"
  elif command -v rubocop >/dev/null 2>&1; then
    TYPECHECK_CMD="bundle exec rubocop --no-color"
  fi
fi

if [ -n "$TYPECHECK_CMD" ]; then
  echo "Typecheck command used: $TYPECHECK_CMD"
  eval "$TYPECHECK_CMD" 2>&1
else
  echo "NO_TYPECHECK_CMD"
fi
```

          3. If TYPECHECK_CMD could not be determined (output is "NO_TYPECHECK_CMD"), skip
             execution and emit this finding:
             INFO · ENVIRONMENT · (no file):0 / ✗ No typecheck/lint command found for this project / ✓ Document the project's check command in spec.md or a CLAUDE.md-equivalent file

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
          detect framework and language from repo root manifest files, then assign each changed
          file to the appropriate bucket using the signals below.

```bash
# Detect primary framework/language from manifest files
FRAMEWORK="unknown"
PRIMARY_EXT=""

if [ -f "package.json" ]; then
  if grep -qE '"next"' package.json 2>/dev/null; then
    FRAMEWORK="nextjs"
  elif grep -qE '"react"' package.json 2>/dev/null; then
    FRAMEWORK="react"
  elif grep -qE '"vue"' package.json 2>/dev/null; then
    FRAMEWORK="vue"
  elif grep -qE '"svelte"' package.json 2>/dev/null; then
    FRAMEWORK="svelte"
  elif grep -qE '"express"|"fastify"|"hapi"|"koa"' package.json 2>/dev/null; then
    FRAMEWORK="node-server"
  else
    FRAMEWORK="node"
  fi
  PRIMARY_EXT="ts tsx js jsx mjs"
  [ -f "tsconfig.json" ] && PRIMARY_EXT="ts tsx"
elif [ -f "pyproject.toml" ] || [ -f "setup.py" ] || [ -f "setup.cfg" ]; then
  if grep -qE 'fastapi|starlette' pyproject.toml setup.py setup.cfg 2>/dev/null; then
    FRAMEWORK="fastapi"
  elif grep -qE 'django' pyproject.toml setup.py setup.cfg 2>/dev/null; then
    FRAMEWORK="django"
  elif grep -qE 'flask' pyproject.toml setup.py setup.cfg 2>/dev/null; then
    FRAMEWORK="flask"
  else
    FRAMEWORK="python"
  fi
  PRIMARY_EXT="py"
elif [ -f "go.mod" ]; then
  FRAMEWORK="go"
  PRIMARY_EXT="go"
elif [ -f "Cargo.toml" ]; then
  FRAMEWORK="rust"
  PRIMARY_EXT="rs"
elif ls *.csproj 2>/dev/null | head -1 | grep -q '.csproj'; then
  FRAMEWORK="dotnet"
  PRIMARY_EXT="cs"
elif [ -f "Gemfile" ]; then
  FRAMEWORK="ruby"
  PRIMARY_EXT="rb"
fi
echo "Detected framework: $FRAMEWORK  Primary ext: $PRIMARY_EXT"
```

          Bucket each changed file using these language-agnostic signals applied after
          framework detection. Every bucket that has zero matches must be noted as empty —
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
        You are a specialist code reviewer. Read only your assigned files.
        Output findings only. No praise. No padding.
        Format:
        SEVERITY · CATEGORY · path/to/file.ts:LINE / ✗ problem / ✓ fix

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
          <finding condition="auth missing">CRITICAL · SECURITY · {file}:{line} / ✗ Auth required by spec but auth() not called / ✓ Call auth() before logic</finding>
          <finding condition="frozen contract changed">CRITICAL · COMPAT · {file}:{line} / ✗ Frozen contract {name} broken / ✓ Restore signature; make additive change</finding>
          <finding condition="model missing">CRITICAL · SPEC · {schema-file}:0 / ✗ Model/schema {name} missing / ✓ Add per spec</finding>
        </findings>
      </reviewer>

      <reviewer id="2" name="security">
        <files>ROUTE_FILES + ACTION_FILES + CONFIG_FILES</files>

        <ponytail-review>
          <rule>Never accept “minimal” code that skips auth, validation, ownership checks, or rate limits.</rule>
          <rule>Prefer existing auth/Zod/rate-limit helpers over new security wrappers.</rule>
          <rule>Smallest safe fix beats large security framework changes.</rule>
          <rule>Do not recommend new dependencies unless existing stack cannot cover the risk.</rule>
        </ponytail-review>

        <checklist>
          <check>auth() before DB/business logic</check>
          <check>IDOR ownership scope</check>
          <check>Zod validation before DB writes/queries</check>
          <check>crypto-safe token generation</check>
          <check>no sensitive logs</check>
          <check>no NEXT_PUBLIC secrets</check>
          <check>server actions authenticate before mutation</check>
          <check>rate limits on public mutation endpoints</check>
          <check>no production source maps</check>
        </checklist>

        <findings>
          <finding>CRITICAL · SECURITY · {file}:{line} / ✗ Logic runs before auth() / ✓ Move auth() before logic</finding>
          <finding>CRITICAL · SECURITY · {file}:{line} / ✗ No ownership filter on {model} lookup / ✓ Scope query to current user/org</finding>
          <finding>CRITICAL · SECURITY · {file}:{line} / ✗ Unvalidated input reaches database / ✓ Validate with z.object(...).safeParse()</finding>
          <finding>CRITICAL · SECURITY · {file}:{line} / ✗ Math.random() used for secret/token / ✓ Use crypto.randomBytes()</finding>
          <finding>HIGH · SECURITY · {file}:{line} / ✗ Sensitive value logged / ✓ Remove or log non-sensitive id</finding>
          <finding>CRITICAL · SECURITY · {file}:{line} / ✗ Secret exposed via NEXT_PUBLIC_ / ✓ Keep secret server-side</finding>
          <finding>HIGH · SECURITY · {file}:{line} / ✗ Public mutation endpoint lacks rate limit / ✓ Apply existing rate-limit pattern</finding>
          <finding>HIGH · SECURITY · {file}:{line} / ✗ Production source maps enabled / ✓ Disable productionBrowserSourceMaps</finding>
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
            <check>N+1 DB queries</check>
            <check>unbounded findMany</check>
            <check>wide query without select</check>
            <check>sequential independent queries</check>
            <check>missing index for filter field</check>
            <check>deep include nesting</check>
          </backend>
          <frontend>
            <check>key=index on dynamic list</check>
            <check>inline object/array props causing rerenders</check>
            <check>heavy client import without dynamic import</check>
            <check>direct state mutation</check>
          </frontend>
        </checklist>

        <findings>
          <finding>HIGH · PERFORMANCE · {file}:{line} / ✗ N+1 query in loop / ✓ Batch with findMany({ where: { id: { in: ids } } })</finding>
          <finding>HIGH · PERFORMANCE · {file}:{line} / ✗ findMany() has no limit / ✓ Add take/skip with default limit</finding>
          <finding>MEDIUM · PERFORMANCE · {file}:{line} / ✗ Fetches unused columns / ✓ Add select for used fields</finding>
          <finding>MEDIUM · PERFORMANCE · {file}:{line} / ✗ Independent queries run sequentially / ✓ Use Promise.all</finding>
          <finding>HIGH · PERFORMANCE · {file}:{line} / ✗ WHERE field lacks index / ✓ Add @@index([{field}])</finding>
          <finding>MEDIUM · PERFORMANCE · {file}:{line} / ✗ key=index on dynamic list / ✓ Use stable id key</finding>
          <finding>MEDIUM · PERFORMANCE · {file}:{line} / ✗ Inline object prop recreates each render / ✓ Hoist constant or useMemo</finding>
          <finding>HIGH · PERFORMANCE · {file}:{line} / ✗ State mutated directly / ✓ Create new state reference</finding>
        </findings>
      </reviewer>

      <reviewer id="4" name="type-safety">
        <files>ALL_SOURCE_FILES</files>

        <dispatch-logic>
          Before running this reviewer, detect the primary language of ALL_SOURCE_FILES
          (TypeScript/Flow → ts/tsx/flow extensions; Python → .py; Go → .go; Rust → .rs;
          Java/Kotlin → .java/.kt; Ruby → .rb; plain JS → .js/.mjs/.cjs with no tsconfig.json).

          Then apply the matching sub-reviewer below. Name the reviewer in output as
          "type-safety ({detected language})" so the summary table category is accurate.

          <sub-reviewer language="TypeScript" output-name="type-safety (TypeScript)">
            <!-- Dispatched only when ALL_SOURCE_FILES are primarily .ts/.tsx or Flow-typed -->
            <ponytail-review>
              <rule>Do not demand elaborate type architecture for local/simple types.</rule>
              <rule>Prefer inferred internal types when safe; require explicit exported boundaries.</rule>
              <rule>Reject any, unsafe casts, and unguarded unknown.</rule>
              <rule>Small type guard beats broad assertion.</rule>
            </ponytail-review>
            <checklist>
              <check>any in signatures/variables</check>
              <check>double assertion</check>
              <check>@ts-ignore or unexplained @ts-expect-error</check>
              <check>unsafe non-null assertion</check>
              <check>exported function missing return type</check>
              <check>unknown accessed without guard</check>
            </checklist>
            <findings>
              <finding>HIGH · TYPESCRIPT · {file}:{line} / ✗ any disables type checking / ✓ Replace with real type or unknown + guard</finding>
              <finding>HIGH · TYPESCRIPT · {file}:{line} / ✗ Double assertion bypasses checker / ✓ Fix upstream type</finding>
              <finding>HIGH · TYPESCRIPT · {file}:{line} / ✗ Type error suppressed / ✓ Fix root type mismatch or justify</finding>
              <finding>MEDIUM · TYPESCRIPT · {file}:{line} / ✗ Unsafe non-null assertion / ✓ Add null check</finding>
              <finding>MEDIUM · TYPESCRIPT · {file}:{line} / ✗ Exported function lacks return type / ✓ Add explicit return type</finding>
              <finding>HIGH · TYPESCRIPT · {file}:{line} / ✗ unknown accessed without narrowing / ✓ Add type guard</finding>
            </findings>
          </sub-reviewer>

          <sub-reviewer language="Python" output-name="type-safety (Python)">
            <!-- Dispatched when ALL_SOURCE_FILES are primarily .py -->
            <ponytail-review>
              <rule>Do not demand type annotations on trivial one-liners or private helpers.</rule>
              <rule>Require type hints on all public function signatures.</rule>
              <rule>Reject bare except and unchecked Optional/None access on public API paths.</rule>
            </ponytail-review>
            <checklist>
              <check>public function missing type hints on parameters or return value</check>
              <check>use of Any from typing without justification</check>
              <check>bare except: clause (catches BaseException silently)</check>
              <check>unchecked Optional/None access (obj.attr where obj could be None)</check>
              <check>mypy/pyright suppression: # type: ignore without explanation</check>
            </checklist>
            <findings>
              <finding>MEDIUM · TYPESAFETY · {file}:{line} / ✗ Public function missing type hints / ✓ Add parameter and return type annotations</finding>
              <finding>HIGH · TYPESAFETY · {file}:{line} / ✗ Any used without justification / ✓ Replace with concrete type or TypeVar</finding>
              <finding>HIGH · TYPESAFETY · {file}:{line} / ✗ bare except: silently swallows all exceptions / ✓ Catch specific exception class</finding>
              <finding>HIGH · TYPESAFETY · {file}:{line} / ✗ Unchecked None access / ✓ Guard with if x is not None or assert</finding>
              <finding>HIGH · TYPESAFETY · {file}:{line} / ✗ Type error suppressed via # type: ignore / ✓ Fix root type mismatch or add explanation comment</finding>
            </findings>
          </sub-reviewer>

          <sub-reviewer language="Go" output-name="type-safety (Go)">
            <!-- Dispatched when ALL_SOURCE_FILES are primarily .go -->
            <ponytail-review>
              <rule>Errors must be handled explicitly; _ = err is almost always wrong.</rule>
              <rule>Type assertions must use the two-return form unless the type is guaranteed.</rule>
              <rule>interface{}/any overuse signals a missing design decision.</rule>
            </ponytail-review>
            <checklist>
              <check>ignored errors: _ = err or err discarded without comment</check>
              <check>unsafe type assertion without ok-check: x.(T) in non-test code</check>
              <check>interface{} or any overuse where a concrete type is available</check>
              <check>unchecked nil pointer dereference (method call on potentially nil pointer)</check>
            </checklist>
            <findings>
              <finding>HIGH · TYPESAFETY · {file}:{line} / ✗ Error silently discarded / ✓ Handle or explicitly log and return</finding>
              <finding>HIGH · TYPESAFETY · {file}:{line} / ✗ Type assertion without ok-check panics on wrong type / ✓ Use x, ok := v.(T); if !ok { ... }</finding>
              <finding>MEDIUM · TYPESAFETY · {file}:{line} / ✗ interface{}/any used where concrete type possible / ✓ Define a concrete type or interface with methods</finding>
              <finding>HIGH · TYPESAFETY · {file}:{line} / ✗ Potential nil dereference / ✓ Add nil guard before access</finding>
            </findings>
          </sub-reviewer>

          <sub-reviewer language="Rust" output-name="type-safety (Rust)">
            <!-- Dispatched when ALL_SOURCE_FILES are primarily .rs -->
            <ponytail-review>
              <rule>unwrap()/expect() in non-test code must be justified.</rule>
              <rule>unsafe blocks require a comment explaining the invariant upheld.</rule>
              <rule>Silently ignored Results compound into hard-to-debug failures.</rule>
            </ponytail-review>
            <checklist>
              <check>unwrap() or expect() in non-test code without justification comment</check>
              <check>unsafe block without a SAFETY comment explaining the upheld invariant</check>
              <check>ignored Result via let _ = or without ? or explicit handling</check>
            </checklist>
            <findings>
              <finding>HIGH · TYPESAFETY · {file}:{line} / ✗ unwrap()/expect() in non-test code without justification / ✓ Handle the Err case or add // SAFETY: comment</finding>
              <finding>HIGH · TYPESAFETY · {file}:{line} / ✗ unsafe block lacks SAFETY comment / ✓ Add // SAFETY: comment explaining the invariant</finding>
              <finding>HIGH · TYPESAFETY · {file}:{line} / ✗ Result silently discarded / ✓ Propagate with ? or handle explicitly</finding>
            </findings>
          </sub-reviewer>

          <sub-reviewer language="Java" output-name="type-safety (Java)">
            <!-- Dispatched when ALL_SOURCE_FILES are primarily .java -->
            <ponytail-review>
              <rule>Raw types erase generic safety — always parameterize.</rule>
              <rule>Unchecked casts must be preceded by instanceof.</rule>
              <rule>@SuppressWarnings must carry an explanation comment.</rule>
            </ponytail-review>
            <checklist>
              <check>raw types (List, Map, Set without type parameter)</check>
              <check>unchecked cast without preceding instanceof check</check>
              <check>@SuppressWarnings without justification comment</check>
              <check>nullable method return accessed without null-check</check>
            </checklist>
            <findings>
              <finding>HIGH · TYPESAFETY · {file}:{line} / ✗ Raw type used / ✓ Add generic type parameter</finding>
              <finding>HIGH · TYPESAFETY · {file}:{line} / ✗ Unchecked cast without instanceof guard / ✓ Add instanceof check before cast</finding>
              <finding>MEDIUM · TYPESAFETY · {file}:{line} / ✗ @SuppressWarnings without explanation / ✓ Add comment justifying suppression</finding>
              <finding>HIGH · TYPESAFETY · {file}:{line} / ✗ Nullable access without null-check / ✓ Add null guard or use Optional</finding>
            </findings>
          </sub-reviewer>

          <sub-reviewer language="Kotlin" output-name="type-safety (Kotlin)">
            <!-- Dispatched when ALL_SOURCE_FILES are primarily .kt -->
            <ponytail-review>
              <rule>!! is a code smell in non-test Kotlin; handle null explicitly.</rule>
              <rule>Unchecked casts and @Suppress must be justified.</rule>
            </ponytail-review>
            <checklist>
              <check>!! (non-null assertion) in non-test code without justification</check>
              <check>unchecked cast (as T) without is T guard</check>
              <check>@Suppress without explanation comment</check>
            </checklist>
            <findings>
              <finding>HIGH · TYPESAFETY · {file}:{line} / ✗ Unjustified !! forces NullPointerException on null / ✓ Use safe call ?. or explicit null check</finding>
              <finding>HIGH · TYPESAFETY · {file}:{line} / ✗ Unchecked cast without is-check / ✓ Guard with if (x is T)</finding>
              <finding>MEDIUM · TYPESAFETY · {file}:{line} / ✗ @Suppress without justification / ✓ Add explanation comment</finding>
            </findings>
          </sub-reviewer>

          <sub-reviewer language="Ruby" output-name="type-safety (Ruby)">
            <!-- Dispatched when ALL_SOURCE_FILES are primarily .rb -->
            <dispatch-condition>
              If the repo uses Sorbet (Gemfile contains sorbet or tapioca, or .sorbet/ exists)
              or RBS signatures are present: check for missing sig {} blocks on public methods.
              Otherwise: skip type-hint checks and check only the items below.
            </dispatch-condition>
            <checklist>
              <check>rescue Exception (catches SignalException, SystemExit — almost always wrong)</check>
              <check>method_missing without respond_to_missing? override</check>
              <check>public method missing Sorbet sig {} if repo uses Sorbet</check>
            </checklist>
            <findings>
              <finding>HIGH · TYPESAFETY · {file}:{line} / ✗ rescue Exception catches too broadly / ✓ Rescue StandardError or a specific exception class</finding>
              <finding>MEDIUM · TYPESAFETY · {file}:{line} / ✗ method_missing without respond_to_missing? / ✓ Override respond_to_missing? alongside method_missing</finding>
              <finding>MEDIUM · TYPESAFETY · {file}:{line} / ✗ Public method missing Sorbet type signature / ✓ Add sig { params(...).returns(...) } block</finding>
            </findings>
          </sub-reviewer>

          <sub-reviewer language="JavaScript (plain)" output-name="type-safety (JavaScript)">
            <!-- Dispatched when primary extension is .js/.mjs/.cjs AND no tsconfig.json exists -->
            <ponytail-review>
              <rule>No TypeScript any-style checks — JS has no type system to enforce.</rule>
              <rule>Check for == vs ===, missing JSDoc on exports if repo convention uses it,
                    and unchecked null/undefined access on chained calls.</rule>
            </ponytail-review>
            <checklist>
              <check>== instead of === (except intentional null-coalescing == null)</check>
              <check>exported function missing JSDoc if repo convention uses JSDoc</check>
              <check>chained property access on value that may be null/undefined without guard</check>
            </checklist>
            <findings>
              <finding>MEDIUM · TYPESAFETY · {file}:{line} / ✗ Loose equality == used / ✓ Use === unless intentionally coercing null/undefined</finding>
              <finding>LOW · TYPESAFETY · {file}:{line} / ✗ Exported function missing JSDoc / ✓ Add @param and @returns JSDoc comment</finding>
              <finding>HIGH · TYPESAFETY · {file}:{line} / ✗ Unchecked null/undefined access / ✓ Add guard or use optional chaining ?.</finding>
            </findings>
          </sub-reviewer>

          <sub-reviewer language="other" output-name="type-safety ({detected language})">
            <!-- Dispatched when primary language does not match any sub-reviewer above -->
            <ponytail-review>
              <rule>Apply only the generalizable type-safety concerns below.</rule>
            </ponytail-review>
            <checklist>
              <check>suppressed type or lint errors (# type: ignore, @SuppressWarnings, eslint-disable, etc.)</check>
              <check>unsafe casts or forced type coercions without guard</check>
              <check>ignored errors (error return discarded, exception caught and swallowed)</check>
            </checklist>
            <findings>
              <finding>HIGH · TYPESAFETY · {file}:{line} / ✗ Type/lint error suppressed without justification / ✓ Fix root cause or add explanation comment</finding>
              <finding>HIGH · TYPESAFETY · {file}:{line} / ✗ Unsafe cast or forced coercion / ✓ Add runtime type guard before cast</finding>
              <finding>HIGH · TYPESAFETY · {file}:{line} / ✗ Error silently ignored / ✓ Handle or propagate the error</finding>
              <finding>INFO · TYPESAFETY · (no file):0 / ✗ No language-specific strictness checklist defined for {detected language} / ✓ Extend prd-review.md reviewer #4 for this language</finding>
            </findings>
          </sub-reviewer>
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
            <check>shared component imports module-specific code</check>
            <check>cross-module imports violate boundaries</check>
            <check>new PrismaClient outside db singleton</check>
            <check>API route fetches internal API route</check>
            <check>client imports server-only module</check>
            <check>design token defined in component</check>
          </architecture>
        </checklist>

        <findings>
          <finding>MEDIUM · QUALITY · {file}:{line} / ✗ Debug log left in code / ✓ Remove</finding>
          <finding>LOW · QUALITY · {file}:{line} / ✗ Unresolved TODO/FIXME/HACK / ✓ Resolve or link issue</finding>
          <finding>LOW · QUALITY · {file}:{line} / ✗ Repeated literal / ✓ Extract named constant</finding>
          <finding>MEDIUM · QUALITY · {file}:{line} / ✗ Function does too much / ✓ Extract named helper</finding>
          <finding>MEDIUM · QUALITY · {file}:{line} / ✗ Duplicated logic / ✓ Reuse existing helper or extract one</finding>
          <finding>MEDIUM · QUALITY · {file}:{line} / ✗ ESLint rule disabled / ✓ Fix root issue</finding>
          <finding>HIGH · ARCHITECTURE · {file}:{line} / ✗ Shared component imports module code / ✓ Invert with prop or move shared logic to lib</finding>
          <finding>CRITICAL · ARCHITECTURE · {file}:{line} / ✗ New PrismaClient outside db singleton / ✓ Import db singleton</finding>
          <finding>MEDIUM · ARCHITECTURE · {file}:{line} / ✗ API route calls internal API / ✓ Call service directly</finding>
          <finding>CRITICAL · ARCHITECTURE · {file}:{line} / ✗ Client imports server-only module / ✓ Move logic server-side/API</finding>
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
SEVERITY · CATEGORY · path/to/file.ts:LINE
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
      <rule>mcp__semble is mandatory for all review code discovery — call mcp__semble__search to find relevant files by concept before reading them. Most token-efficient path.</rule>
    </hard-rules>
  </control>

</command>