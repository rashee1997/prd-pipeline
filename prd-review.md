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
      <task>Collect changed files, TypeScript errors, PRD/spec context, and file buckets.</task>

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

        <step name="typescript-check">
```bash
bun tsc --noEmit 2>&1
```
          <on-fail>
            Record each error as:
            CRITICAL · TYPESCRIPT · file:line / ✗ {error} / ✓ Fix the type error before continuing.
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
          <bucket name="ROUTE_FILES">src/app/api/**/*.ts</bucket>
          <bucket name="ACTION_FILES">files containing "use server"</bucket>
          <bucket name="COMPONENT_FILES">changed src/**/*.tsx</bucket>
          <bucket name="SERVICE_FILES">changed src/lib/**/*.ts or src/modules/**/*.ts</bucket>
          <bucket name="SCHEMA_FILES">prisma/schema.prisma</bucket>
          <bucket name="TEST_FILES">*.test.ts, *.spec.ts, __tests__/**</bucket>
          <bucket name="CONFIG_FILES">*.config.*, .env*, next.config.*</bucket>
          <bucket name="ALL_TS_FILES">all changed .ts/.tsx files</bucket>
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
        <files>ALL_TS_FILES + spec/prd excerpts</files>

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
          <finding condition="model missing">CRITICAL · SPEC · prisma/schema.prisma:0 / ✗ Model {name} missing / ✓ Add per spec</finding>
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

      <reviewer id="4" name="typescript-strictness">
        <files>ALL_TS_FILES</files>

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
      </reviewer>

      <reviewer id="5" name="quality-architecture">
        <files>ALL_TS_FILES</files>

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
        <files>TEST_FILES + ALL_TS_FILES</files>

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
        <typescript_errors>always first</typescript_errors>
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
  TypeScript      : {PASS|FAIL — N issues}
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
      <rule>TypeScript errors are CRITICAL</rule>
      <rule>Ponytail applies to every reviewer</rule>
      <rule>Do not recommend new abstraction unless duplication or boundary pressure is proven</rule>
      <rule>Do not recommend new dependency unless existing code, stdlib, and platform cannot solve it safely</rule>
      <rule>mcp__semble is mandatory for all review code discovery — call mcp__semble__search to find relevant files by concept before reading them. Most token-efficient path.</rule>
    </hard-rules>
  </control>

</command>