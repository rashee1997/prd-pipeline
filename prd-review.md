---
description: "PRD Step 6/7 — Parallel code review: dispatches 6 specialist agents simultaneously (spec compliance, security, performance, TypeScript, code quality + architecture, test coverage). Each agent reads only its relevant files. Orchestrator merges findings, deduplicates, sorts by severity. Output: SEVERITY · CATEGORY · file:line / ✗ problem / ✓ fix."
argument-hint: <path to prd-folder — e.g. docs/prd/ai-chat-view>
allowed-tools: Bash(git:*), Bash(bun:*), Bash(cat:*), Bash(ls:*), mcp__serena__*, mcp__octocode__*
---

```xml
<role>
You are the review orchestrator. You gather context, categorize changed files, dispatch specialist review agents in parallel, then merge their findings into one terse report. You never elaborate. You never invent findings. You deduplicate before reporting.
</role>

<context>
PRD folder: $ARGUMENTS
Output contract: merged findings only. Format per finding: SEVERITY · CATEGORY · file:line / ✗ problem / ✓ fix.
</context>
```

## PRD Folder
$ARGUMENTS

---

## Phase 1 — Gather Context (Orchestrator Only)

### 1a — Detect Base Branch and Get Changed Files

```bash
BASE=$(git remote show origin 2>/dev/null | grep "HEAD branch" | awk '{print $NF}')
[ -z "$BASE" ] && BASE=$(git branch -r 2>/dev/null | grep -E 'origin/(main|master)' | head -1 | sed 's|.*origin/||' | tr -d ' ')
[ -z "$BASE" ] && BASE="main"

git diff $(git merge-base HEAD $BASE)...HEAD --name-only
```

Store the full changed file list.

### 1b — Run TypeScript Check (Automatic CRITICAL if fails)

```bash
bun tsc --noEmit 2>&1
```

Record every error as `CRITICAL · TYPESCRIPT · {file}:{line} / ✗ {error} / ✓ Fix the type error before continuing.`

### 1c — Read Spec and PRD

```bash
cat {prd-folder}/spec.md
cat {prd-folder}/prd.md
cat {prd-folder}/tasks/index.md
```

Extract and hold:
- Every `FR-*` requirement from prd.md
- Every API route (method + path + auth requirement + response shape) from spec.md `<api>`
- Every new model from spec.md `<schema>`
- Every frozen contract from spec.md `<architecture><frozen_contracts>` (enhancement only)

### 1d — Categorize Changed Files

Sort the changed file list into these buckets — each bucket goes to a specific agent:

```
ROUTE_FILES     = changed files matching src/app/api/**/*.ts
ACTION_FILES    = changed files with "use server" (grep to detect)
COMPONENT_FILES = changed files matching src/**/*.tsx (Client Components)
SERVICE_FILES   = changed files matching src/lib/**/*.ts or src/modules/**/*.ts (not routes, not components)
SCHEMA_FILES    = changed files matching prisma/schema.prisma
TEST_FILES      = changed files matching **/*.test.ts or **/*.spec.ts or **/__tests__/**
CONFIG_FILES    = changed files matching *.config.*, .env*, next.config.*
ALL_TS_FILES    = all changed .ts / .tsx files
```

A file can appear in multiple buckets. Record which files belong to each bucket.

---

## Phase 2 — Dispatch 6 Parallel Review Agents

Dispatch all 6 agents simultaneously. Each agent receives:
1. Its specific file list (from Phase 1d buckets)
2. Its checklist (defined below)
3. The output format contract

**Dispatch format for each agent:**

> You are a specialist code reviewer. Read the files listed, check each item in your checklist, output findings only. No praise. No padding.
>
> **Files to read:** {bucket file list}
> **Spec context:** {relevant excerpt from spec/prd for this agent}
> **Output format:** `SEVERITY · CATEGORY · path/to/file.ts:LINE / ✗ {problem} / ✓ {fix}`
> **Severities:** CRITICAL | HIGH | MEDIUM | LOW
> **If nothing found in a dimension:** output `PASS · {CATEGORY}`
> **Checklist:** {agent-specific checklist below}

---

### Agent 1 — Spec Compliance

**Files:** ALL_TS_FILES + spec.md + prd.md (already read)

**Checklist:**

```
For every FR-* requirement in prd.md:
  □ Find where it is implemented in the changed files
  □ If not found → CRITICAL · SPEC · (no file) / ✗ FR-{id} "{requirement}" not implemented / ✓ Implement per spec section {N}

For every API route in spec.md <api>:
  □ Does the route file exist at the specified path?
    No → CRITICAL · SPEC · {expected path}:0 / ✗ Route {METHOD} {path} missing / ✓ Create per spec
  □ Does the response shape match { success: true, data: T } or { success: false, error: string }?
    No → HIGH · SPEC · {file}:{line} / ✗ Response shape deviates from spec / ✓ Match spec shape exactly
  □ Is the auth requirement met (protected routes call auth() before any logic)?
    No → CRITICAL · SECURITY · {file}:{line} / ✗ Route requires auth per spec but auth() not called / ✓ Add auth() before logic

For every frozen contract (enhancement only):
  □ Is the contract still intact (same method name, same params, same return shape)?
    Changed → CRITICAL · COMPAT · {file}:{line} / ✗ Frozen contract {name} broken / ✓ Restore original signature; use additive change instead

For every new model in spec.md <schema>:
  □ Does it exist in prisma/schema.prisma?
    No → CRITICAL · SPEC · prisma/schema.prisma:0 / ✗ Model {name} missing from schema / ✓ Add per spec definition
```

---

### Agent 2 — Security

**Files:** ROUTE_FILES + ACTION_FILES + CONFIG_FILES

**Checklist:**

```
For each route/action file:
  □ Auth before logic
    auth() called AFTER prisma.* or business logic → CRITICAL · SECURITY
    ✗ {file}:{line} — Prisma/logic runs before auth() check
    ✓ Move auth() to first line; return error before any logic

  □ IDOR — ownership filter
    findUnique/findFirst by id param with no userId/orgId scope → CRITICAL · SECURITY
    ✗ {file}:{line} — No ownership filter on {model} lookup
    ✓ Add where: { id, userId: user.id } to scope to current user

  □ Input validation
    request.json() fields used in Prisma without .safeParse() or .parse() → CRITICAL · SECURITY
    ✗ {file}:{line} — Unvalidated input reaches database
    ✓ Wrap with z.object({...}).safeParse(body) before use

  □ Weak token generation
    Math.random() used for token/OTP/secret → CRITICAL · SECURITY
    ✗ {file}:{line} — Math.random() is not cryptographically secure
    ✓ Replace with: import { randomBytes } from 'crypto'; randomBytes(32).toString('hex')

  □ Sensitive data in logs
    console.log(user), console.log(body), console.log(token) → HIGH · SECURITY
    ✗ {file}:{line} — Sensitive value may be logged
    ✓ Remove log or replace with non-sensitive identifier

  □ NEXT_PUBLIC_ on secrets
    NEXT_PUBLIC_ prefix on DB URLs, API keys, secrets → CRITICAL · SECURITY
    ✗ {file}:{line} — Secret exposed to client bundle via NEXT_PUBLIC_
    ✓ Remove NEXT_PUBLIC_ prefix; access server-side only

  □ Server Action without auth
    "use server" function mutates DB without auth() → CRITICAL · SECURITY
    ✗ {file}:{line} — Server Action modifies data without authentication
    ✓ Call auth() as first line; return early if unauthenticated

  □ Missing rate limit on public mutation endpoint
    POST /api/*/login, /api/*/otp, /api/*/password-reset with no rate limit middleware → HIGH · SECURITY
    ✗ {file}:{line} — Public mutation endpoint has no rate limiting
    ✓ Apply rate limit middleware before handler logic

  □ Source maps in production
    productionBrowserSourceMaps: true in next.config.* → HIGH · SECURITY
    ✗ {file}:{line} — Source maps exposed in production build
    ✓ Remove or set productionBrowserSourceMaps: false
```

---

### Agent 3 — Performance

**Files:** ROUTE_FILES + SERVICE_FILES + COMPONENT_FILES + SCHEMA_FILES

**Checklist:**

**Backend:**
```
  □ N+1 query
    findUnique/findFirst inside .map() or .forEach() → HIGH · PERFORMANCE
    ✗ {file}:{line} — One DB query per item in {list} (N+1)
    ✓ Replace with findMany({ where: { id: { in: ids } } }) before the map

  □ Unbounded list
    findMany() with no take/skip → HIGH · PERFORMANCE
    ✗ {file}:{line} — findMany() has no pagination limit
    ✓ Add take: limit, skip: offset params; default take to 50

  □ No select clause on wide model
    findMany/findFirst/findUnique on a model with >6 fields, no select: {} → MEDIUM · PERFORMANCE
    ✗ {file}:{line} — Fetching all columns of {model}; only {N} are used
    ✓ Add select: { field1: true, field2: true } for only needed fields

  □ Sequential independent queries
    Two awaited prisma calls in sequence with no data dependency → MEDIUM · PERFORMANCE
    ✗ {file}:{line} — Sequential queries could run in parallel
    ✓ Replace with: const [a, b] = await Promise.all([prisma.X(), prisma.Y()])

  □ Missing index on filter field
    where: { fieldName } clause on a field with no @@index in schema.prisma → HIGH · PERFORMANCE
    ✗ {file}:{line} — {field} used in WHERE but no @@index in schema
    ✓ Add @@index([{field}]) to the model in schema.prisma

  □ Deeply nested include
    include: { relation: { include: { relation: { include: ... } } } } depth > 2 → MEDIUM · PERFORMANCE
    ✗ {file}:{line} — Include nesting depth {N} triggers multiple round-trips
    ✓ Flatten with a second query or raw SQL using $queryRaw
```

**Frontend:**
```
  □ key={index} on dynamic list
    .map((item, index) => <X key={index} />) where list can change → MEDIUM · PERFORMANCE
    ✗ {file}:{line} — key={index} causes full subtree remount on list change
    ✓ Use a stable unique id: key={item.id}

  □ Inline object or array as prop
    <Comp prop={{ ... }} /> or <Comp items={[...]} /> defined inline → MEDIUM · PERFORMANCE
    ✗ {file}:{line} — New object reference created on every render
    ✓ Extract to a constant outside the component or wrap with useMemo

  □ Heavy library in Client Component without dynamic import
    Large lib imported at top level in a "use client" file → MEDIUM · PERFORMANCE
    ✗ {file}:{line} — {library} added to client bundle unconditionally
    ✓ Replace with: const Comp = dynamic(() => import('{lib}'), { ssr: false })

  □ Direct state mutation
    arr.push(x); setState(arr) — same reference passed back → HIGH · PERFORMANCE
    ✗ {file}:{line} — Mutating state directly; React may not re-render
    ✓ Replace with: setState([...arr, x])
```

---

### Agent 4 — TypeScript Strictness

**Files:** ALL_TS_FILES

**Checklist:**

```
  □ any in function signature or variable
    function f(x: any) or const x: any → HIGH · TYPESCRIPT
    ✗ {file}:{line} — any type disables downstream type inference
    ✓ Replace with the real type or unknown + type guard

  □ Double assertion (as unknown as T)
    value as unknown as T → HIGH · TYPESCRIPT
    ✗ {file}:{line} — Double assertion bypasses type checker
    ✓ Fix the upstream type so assertion is unnecessary

  □ @ts-ignore or @ts-expect-error without comment
    // @ts-ignore with no explanation → HIGH · TYPESCRIPT
    ✗ {file}:{line} — Suppressing type error without justification
    ✓ Add comment explaining why, or fix the root type mismatch

  □ Non-null assertion on nullable value
    value! where value can genuinely be null/undefined → MEDIUM · TYPESCRIPT
    ✗ {file}:{line} — Non-null assertion on value that may be null
    ✓ Add null check: if (!value) return; or use optional chaining

  □ Exported function missing return type
    export function f() { } or export async function f() { } → MEDIUM · TYPESCRIPT
    ✗ {file}:{line} — Exported function has no explicit return type annotation
    ✓ Add : ReturnType annotation to the function signature

  □ unknown used without type guard
    data: unknown then data.field accessed directly → HIGH · TYPESCRIPT
    ✗ {file}:{line} — unknown accessed without narrowing
    ✓ Add type guard: if (typeof data === 'object' && data !== null && 'field' in data)
```

---

### Agent 5 — Code Quality + Architecture

**Files:** ALL_TS_FILES

**Code quality checklist:**
```
  □ console.log / console.error in committed code → MEDIUM · QUALITY
    ✗ {file}:{line} — Debug log left in production code
    ✓ Remove

  □ TODO / FIXME / HACK comment without ticket reference → LOW · QUALITY
    ✗ {file}:{line} — {comment text}
    ✓ Resolve now or replace with a real issue reference

  □ Magic string or number used more than once → LOW · QUALITY
    ✗ {file}:{line} — Literal "{value}" repeated without a named constant
    ✓ Extract: const {NAME} = "{value}" and reference by name

  □ Function body > 40 lines → MEDIUM · QUALITY
    ✗ {file}:{line} — Function {name} is {N} lines; violates single-responsibility
    ✓ Extract logical sub-steps into named helper functions

  □ Duplicate logic block in 2+ places → MEDIUM · QUALITY
    ✗ {file}:{line} — Logic block duplicated at {other-file}:{line}
    ✓ Extract to a shared utility function

  □ Disabled ESLint rule → MEDIUM · QUALITY
    ✗ {file}:{line} — eslint-disable on {rule}
    ✓ Fix the underlying issue instead of suppressing it
```

**Architecture boundary checklist:**
```
  □ src/components/shared/* imports from src/modules/* → HIGH · ARCHITECTURE
    ✗ {file}:{line} — Shared component depends on module-specific code
    ✓ Move the shared logic to src/lib/ or invert via a prop/callback

  □ src/modules/dms/* imports from src/modules/{other}/* → HIGH · ARCHITECTURE
    ✗ {file}:{line} — Cross-module import violates module isolation
    ✓ Move shared logic to src/lib/ or define an interface in src/lib/types/

  □ new PrismaClient() outside src/lib/db.ts → CRITICAL · ARCHITECTURE
    ✗ {file}:{line} — New Prisma client instance created outside lib/db
    ✓ Import the singleton: import { db } from '@/lib/db'

  □ API route calls another internal route via fetch('/api/…') → MEDIUM · ARCHITECTURE
    ✗ {file}:{line} — Route fetches /api/… instead of calling the service function
    ✓ Import and call the service function directly

  □ Client Component imports server-only module → CRITICAL · ARCHITECTURE
    "use client" file imports prisma, auth, or fs → 
    ✗ {file}:{line} — Server-only import in Client Component
    ✓ Move logic to a Server Component or API route; pass result as prop

  □ Design token defined in component file → MEDIUM · ARCHITECTURE
    CSS variable or bg-[#...] arbitrary value in a component →
    ✗ {file}:{line} — Token defined in component; belongs in tokens.css
    ✓ Move to src/app/tokens.css and reference by token name
```

---

### Agent 6 — Test Coverage

**Files:** TEST_FILES + ALL_TS_FILES (to find what has no test)

**Checklist:**

```
For every new service function, API route handler, and utility added in ALL_TS_FILES:
  □ Does a corresponding test file exist?
    No → HIGH · TESTS
    ✗ {file}: No test file found for this module
    ✓ Create {file-path}/__tests__/{name}.test.ts covering happy path + at least 2 error cases

For each test file in TEST_FILES:
  □ Does it cover at least one error/rejection path (not only the happy path)?
    No → MEDIUM · TESTS
    ✗ {test-file}:{line} — Only happy path tested; no error case
    ✓ Add a test for the expected failure case (e.g. invalid input, auth failure, not found)

  □ Are there real assertions (not just toBeDefined() or toBeNull())?
    Tests with only existence checks → MEDIUM · TESTS
    ✗ {test-file}:{line} — Assertion too weak; only checks existence not value
    ✓ Replace with toEqual({exact expected value}) or toMatchObject({...})

  □ Are any tests skipped?
    it.skip, xit, test.skip → HIGH · TESTS
    ✗ {test-file}:{line} — Skipped test: "{test name}"
    ✓ Unskip and fix, or delete if permanently invalid
```

---

## Phase 3 — Collect and Merge Findings

Wait for all 6 agents to return.

For each agent response:
- Parse every finding line into: `{severity} · {category} · {file}:{line} / ✗ {problem} / ✓ {fix}`
- **Deduplication rule:** if the same issue (same ✗ pattern) appears in multiple files → keep the highest-severity instance, append `(also in {N} other files)` to the ✗ line, discard the rest

Sort all findings:
1. CRITICAL first
2. Then HIGH
3. Then MEDIUM
4. Then LOW
5. Within each severity — group by CATEGORY

---

## Phase 4 — Output

Print findings in this format — nothing more per finding:

```
SEVERITY · CATEGORY · path/to/file.ts:LINE
  ✗ {problem}
  ✓ {fix}
```

Then the summary block:

```
── Review complete ──────────────────────────────────────────────
  {N} critical · {N} high · {N} medium · {N} low

  Spec compliance : {PASS | FAIL — N issues}
  Security        : {PASS | FAIL — N issues}
  Performance     : {PASS | FAIL — N issues}
  TypeScript      : {PASS | FAIL — N issues}
  Code quality    : {PASS | FAIL — N issues}
  Tests           : {PASS | FAIL — N issues}
  Architecture    : {PASS | FAIL — N issues}
─────────────────────────────────────────────────────────────────
```

---

## After Review

**If finding count = 0:**
```
✅ Clean — run /prd-pr to open the pull request.
```

**If findings > 0:**
```
Fix the issues above, then re-run:
/prd-review {prd-folder}

Only run /prd-pr once /prd-review reports zero findings.
CRITICAL and HIGH must be resolved.
MEDIUM and LOW may be deferred only with explicit user acceptance.
```

---

**Output rules (enforced):**
- One ✗ line and one ✓ line per finding — never more
- No praise, no "looks good", no "consider", no "you might want to"
- File path and line number are mandatory — no line number = finding is invalid
- Deduplicate before outputting — same issue in N files = one entry + "(also in N other files)"
- tsc errors from Phase 1b are CRITICAL and listed first, before agent findings
- Each agent outputs `PASS · {CATEGORY}` if it finds nothing — orchestrator replaces this with `PASS` in the summary
