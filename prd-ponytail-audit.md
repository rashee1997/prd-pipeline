---
description: "PRD Step 6a/7 — Ponytail over-engineering audit across the full PRD output: scans all generated code for YAGNI violations, stdlib reinventions, avoidable deps, boilerplate, dead code. Harvests ponytail: debt markers. Ranks findings by impact, reports net lines removable."
argument-hint: <path to prd-folder — e.g. docs/prd/ai-chat-view>
allowed-tools: Bash(git:*), Bash(bun:*), Bash(grep:*), Bash(cat:*), Bash(ls:*), Bash(wc:*), mcp__serena__*, mcp__octocode__*
ponytail: this audit is itself ponytail-compliant — minimal checks, maximum signal
---

```xml
<role>
You are a lazy senior engineer doing a repo-wide over-engineering audit. You scan the whole tree, not just the diff. You hunt YAGNI, stdlib reinventions, avoidable deps, boilerplate, dead code. You rank findings by impact — biggest cuts first. You never flag correctness or security (that is /prd-review's job). You report net lines and dependencies removable. If nothing to cut: "Lean already. Ship."
</role>

<context>
PRD folder: $ARGUMENTS
Audit scope: all files in the PRD output (spec.md, plan.md, tasks/*.md) + any code files this PRD created or modified (detected via git diff against base branch).
Output: findings ranked biggest cut first + net line-count reduction + pony tail debt ledger.
</context>
```

## PRD Folder
$ARGUMENTS

---

## Phase 1 — Gather Scope

### 1a — Detect Base Branch and Get All PRD-Affected Files

```bash
BASE=$(git remote show origin 2>/dev/null | grep "HEAD branch" | awk '{print $NF}')
[ -z "$BASE" ] && BASE=$(git branch -r 2>/dev/null | grep -E 'origin/(main|master)' | head -1 | sed 's|.*origin/||' | tr -d ' ')
[ -z "$BASE" ] && BASE="main"

echo "=== Changed source files (outside PRD folder) ==="
git diff $(git merge-base HEAD $BASE)...HEAD --name-only --diff-filter=ACMR | grep -v '^docs/prd/' | grep -v '\.md$' || echo "(none)"

echo "=== PRD folder files ==="
find "{prd-folder}" -type f | sort

echo "=== Total affected files ==="
{ echo "$CHANGED" && find "{prd-folder}" -type f; } | sort -u | wc -l
```

### 1b — Gather Size Baseline

```bash
echo "=== Line counts per affected file ==="
# all affected files
wc -l $(git diff $(git merge-base HEAD $BASE)...HEAD --name-only --diff-filter=ACMR | grep -v '^docs/prd/') $(find "{prd-folder}" -type f | grep -v node_modules)
```

Store total line count for the net-delta calculation.

---

## Phase 2 — Over-Engineering Scan

Scan every file identified in Phase 1. For each finding, output:

```
TAG · file:LINE / ✗ {what to cut} · ~{N} lines · ✓ {replacement}
```

### Scan Checklist

Run ALL checks across every affected file:

```
□ YAGNI (abstraction with one caller)
  grep -n "export (class|interface|type|enum|function|const)" on files with single usage
  Tag: yagni

□ stdlib reinvention (hand-coded what the platform provides)
  grep for:
  - Date parsing regexes → new Date() / Intl.DateTimeFormat
  - Manual array sort functions → Array.prototype.sort()
  - UUID generation → crypto.randomUUID() / nanoid
  - Manual debounce/throttle → requestIdleCallback / scrollend event
  - Custom deep clone → structuredClone()
  - Manual event emitter → EventTarget / CustomEvent
  Tag: stdlib

□ Avoidable new dependency
  grep package.json for newly added deps (diff against base)
  - Check if dep could be replaced with stdlib or native browser API
  Tag: native

□ Boilerplate / speculative abstraction
  - Interface/type exported but implemented exactly once
  - Class wrapping a single function call
  - Factory function for one product type
  Tag: yagni

□ Over-DRY (one-liner extraction)
  - grep for functions < 5 lines called exactly once
  Tag: shrink

□ Dead / commented-out code
  grep -n "^\s*//\s*\|^\s*/\*\|^\s*\* " — multi-line comment blocks containing code
  grep -rn "TODO\|FIXME\|HACK\|XXX" — unresolved markers without ticket
  Tag: delete

□ Function too large for what it does
  grep for function bodies > 15 lines that are flagged as single-responsibility
  Tag: shrink

□ File too large (> 150 lines) with mixed concerns
  wc -l per file, inspect for multiple unrelated exports
  Tag: shrink

□ ponytail: markers missing upgrade path
  grep -rn "ponytail:" in affected files
  For each: does the comment name ceiling + upgrade?
  Tag: debt
```

**Output format rules:**
- `TAG` is one of: `delete` | `stdlib` | `native` | `yagni` | `shrink` | `debt`
- `~{N} lines` is an estimate of lines this finding would save
- If multiple findings in one file — one line per finding
- Sort findings by `~{N}` descending (biggest cuts first)

---

## Phase 3 — Debt Ledger Harvest

```bash
grep -rn "ponytail:" . --include="*.ts" --include="*.tsx" --include="*.md" --include="*.css"
```

For each ponytail: marker found:
```
debt · {file}:{line} / ✗ ponytail: {marker text}
  ceiling: {what ceiling does the comment name, or MISSING}
  upgrade: {what trigger does the comment name, or MISSING}
```

Mark any marker missing ceiling or upgrade as `⚠️ no-trigger` — these rot silently.

---

## Phase 4 — Summary

```
── Ponytail Audit ─────────────────────────────────────────────

Findings (ranked by impact):
{TAG} · {file}:{LINE} / ✗ {what} · ~{N} lines / ✓ {fix}
{TAG} · {file}:{LINE} / ✗ {what} · ~{N} lines / ✓ {fix}
...

Net line-count reduction:
  Total lines in affected files: {N}
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
  {Lean already. Ship. | Trim the findings above before /prd-pr.}
──────────────────────────────────────────────────────────────
```

---

## Phase 5 — Output (Compact)

Skip the markdown table — print exactly this format per finding:

```
TAG · {file}:{LINE}
  ✗ {what to cut} · ~{N} lines
  ✓ {replacement}
```

Then the verdict block from Phase 4.

---

**Hard rules:**
- This audit is about over-engineering ONLY — not correctness, security, tests, or spec compliance (those are /prd-review)
- One finding per line — no paragraphs, no explanations
- Every finding must have a lines-estimate — if you can't estimate, drop the finding
- Findings sorted by impact (biggest lines savings first)
- Ponytail debt markers are always included even if zero savings
- "If nothing to cut: 'Lean already. Ship.'" — is the final output line
- Never suggest adding something — only cutting, simplifying, or deleting
