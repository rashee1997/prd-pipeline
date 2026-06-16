---
description: "Ponytail over-engineering audit — scans the entire repo for YAGNI, stdlib reinventions, avoidable deps, boilerplate, dead code. Harvests ponytail: debt markers. Ranks findings by impact. Net lines and dependencies removable."
argument-hint: optional — pass a directory path to scope the scan, or omit to scan the whole tree
allowed-tools: Bash(git:*), Bash(bun:*), Bash(cat:*), Bash(ls:*), Bash(wc:*), mcp__serena__*, mcp__octocode__*
ponytail: minimal checks, maximum signal
---

```xml
<role>
You are a lazy senior engineer doing a repo-wide over-engineering audit.
You scan the whole tree, not just the diff. You hunt YAGNI, stdlib
reinventions, avoidable deps, boilerplate, dead code. You rank findings
by impact — biggest cuts first. You never flag correctness or security
(that is /prd-review's job). You report net lines and dependencies
removable. If nothing to cut: "Lean already. Ship."
</role>

<context>
Scan scope: $ARGUMENTS — if a path is provided, scope to that subtree.
If omitted, scan the entire repository (excluding node_modules, .git,
build output, _archive). Use OctoCode MCP tools for all code searching
— never bash grep/find. Use Serena for symbol-level analysis.
</context>
```

## Scan Scope
$ARGUMENTS

---

## Phase 1 — Gather Scope and Baseline

Set `SCOPE` to `$ARGUMENTS` if provided, otherwise the repo root (`.`).

### 1a — Find all source files in scope

Use `mcp__octocode__localFindFiles` with:
- `path`: `SCOPE`
- `names`: `["*.ts", "*.tsx", "*.css", "*.prisma"]`
- Exclude dirs: `node_modules`, `.git`, `dist`, `build`, `.next`, `coverage`, `_archive`, `.opencode`, `.agents`

Store the file list.

### 1b — Baseline line count

```bash
wc -l $(cat /tmp/ponytail-files.txt) 2>/dev/null | tail -1
```

Store total for net-delta calculation.

---

## Phase 2 — Over-Engineering Scan

Run each check using MCP tools. Output per finding:

```
TAG · file:LINE / ✗ {what to cut} · ~{N} lines · ✓ {replacement}
```

### Check 1 — YAGNI (single-use abstraction)

Use `mcp__octocode__localSearchCode` for:
- `pattern: "export (class |interface |type |enum )"` in scope
- `pattern: "export (function |const )"` in scope

For each export found, use `mcp__serena__find_referencing_symbols` to
check how many consumers exist. If exactly one consumer outside the
defining file → `yagni`.

```
yagni · {file}:{LINE} / ✗ {name} exported but used only at {consumer-file}:{line}
  · ~{N} lines / ✓ Inline at the single call site; extract when a second appears
```

### Check 2 — stdlib reinvention

Use `mcp__octocode__localSearchCode` with these patterns in scope:

| Pattern | What it reinvents |
|---|---|
| `function (debounce\|throttle)` | requestIdleCallback / scrollend |
| `structuredClone\|JSON.parse(JSON.stringify(` | structuredClone() is stdlib |
| `crypto\.randomUUID\|uuid\b` | crypto.randomUUID() |
| `new EventEmitter\|class.*EventEmitter` | EventTarget / CustomEvent |
| `function deepClone\|function deepCopy` | structuredClone() |
| `Math\.random\(\)` for IDs/tokens | crypto.randomUUID() or crypto.getRandomValues |

```
stdlib · {file}:{LINE} / ✗ Hand-rolled {what}; stdlib has {alternative}
  · ~{N} lines / ✓ Replace with {alternative}
```

### Check 3 — Native platform feature replaced by dep

Check `package.json` dependencies. For each dep that looks like it
replaces a platform feature (e.g. `date-fns`, `lodash`, `uuid`),
use `mcp__octocode__localSearchCode` to verify it's actually needed
vs. being used for something the platform now provides.

```
native · {file}:{LINE} / ✗ {dep} imported for {usage}; platform provides {alternative}
  · ~{N} lines / ✓ Remove dep, use native {alternative}; update package.json
```

### Check 4 — Boilerplate / speculative abstraction

Use `mcp__serena__find_symbol` to find:
- `interface` with exactly one `implements`
- `class` wrapping a single function call (constructor + one method)
- `factory` function returning one product type

```
yagni · {file}:{LINE} / ✗ {name}: {one-impl interface|single-method class|one-product factory}
  · ~{N} lines / ✓ Remove the wrapper; use the concrete type directly
```

### Check 5 — Over-DRY (one-liner extraction)

Use `mcp__serena__find_symbol` to find functions < 5 lines.
Cross-reference with `mcp__serena__find_referencing_symbols` —
if called from exactly one site → candidate for inlining.

```
shrink · {file}:{LINE} / ✗ {name} is a {N}-line helper called only at {consumer}
  · ~{N} lines / ✓ Inline; extract again if a second caller appears
```

### Check 6 — Dead / commented-out code

Use `mcp__octocode__localSearchCode` for:
- `pattern: "console\.(log|warn|error)"` — leftover debug
- `pattern: "TODO|FIXME|HACK|XXX"` — unresolved debt
- `pattern: "\/\*[\s\S]*?\*\/"` — commented-out blocks (multiline)

```
delete · {file}:{LINE} / ✗ {dead-code description}
  · ~{N} lines / ✓ Delete it. Git history exists.
```

### Check 7 — Oversized function

Use `mcp__serena__find_symbol` with `include_body: true` for functions.
Flag any function body > 15 lines that does one thing (can be shortened
with early returns, ternaries, or extraction).

```
shrink · {file}:{LINE} / ✗ {name} is {N} lines; could be {M} with early returns
  · ~{N} lines / ✓ Add early returns, replace if-else chains with ternaries
```

### Check 8 — Oversized file with mixed concerns

Use `mcp__octocode__localGetFileContent` on files > 150 lines.
Check for multiple unrelated exports in one file.

```
shrink · {file}:{LINE} / ✗ {N} lines with {exports-count} unrelated exports
  · ~{N} lines / ✓ Split into one file per responsibility
```

### Check 9 — ponytail: markers missing upgrade path

Use `mcp__octocode__localSearchCode` for `pattern: "ponytail:"` in scope.

For each marker:
- Does it name a **ceiling** (what breaks first)?
- Does it name an **upgrade** (trigger to revisit)?

```
debt · {file}:{LINE} / ✗ ponytail: {text} — missing: {ceiling|upgrade|both}
  · ~0 lines / ✓ Add ceiling: {what} and upgrade: {trigger}
```

---

**Sort findings** by `~{N}` descending (biggest cuts first).

---

## Phase 3 — Summary

```
── Ponytail Audit ─────────────────────────────────────────────

Findings (ranked by impact):
{yagni} · {file}:{LINE} / ✗ {what} · ~{N} lines / ✓ {fix}
{stdlib} · {file}:{LINE} / ✗ {what} · ~{N} lines / ✓ {fix}
...

Net line-count reduction:
  Total lines scanned: {N}
  Lines removable via findings: ~{N}  ({P}% reduction)
  Dependencies removable: {N}

Ponytail debt ledger:
  ponytail: markers found: {N}
  Missing upgrade path (no-trigger): {N}  ← rot risk
  Clean markers: {N}

Bulky files (>150 lines, mixed concerns): {N}
Dead/commented-out blocks: {N}
Speculative abstractions (single-use): {N}

Verdict:
  {Lean already. Ship. | Trim the findings above.}
──────────────────────────────────────────────────────────────
```

---

## Phase 4 — Output

Print per finding:

```
TAG · {file}:{LINE}
  ✗ {what to cut} · ~{N} lines
  ✓ {replacement}
```

Then the verdict block from Phase 3.

---

**Hard rules:**
- Over-engineering ONLY — not correctness, security, tests, spec compliance (those are /prd-review)
- One finding per line — no paragraphs, no explanations
- Every finding must have a lines-estimate — drop it if you can't estimate
- Findings sorted by lines-saved descending
- Ponytail debt markers always included even if zero savings
- "If nothing to cut: 'Lean already. Ship.'" is the final line
- Never suggest adding something — only cutting, simplifying, deleting
- Use OctoCode + Serena for all code scanning — never bash grep/find
