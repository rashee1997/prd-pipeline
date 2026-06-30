# PRD Command Pipeline — Audit Report

**Date:** 30-06-2025
**Scope:** All 9 commands in `.claude/commands/prd/` — `prd-discover`, `prd-write`, `prd-plan`, `prd-tasks`, `prd-validate`, `prd-implement`, `prd-review`, `prd-pr`, `prd-release`.
**Lens:** correctness, strictness, token efficiency, cross-command consistency, duplicated instructions.
**Note:** This is a findings-only report. No command files were edited (per request). `prd-write.md` already received the external-research enforcement fix in a prior commit; that is excluded from "open" findings below.

---

## Severity summary

| # | Finding | Severity | Commands affected |
|---|---|---|---|
| F1 | Invented MCP tool names that don't exist | **CRITICAL** | discover, plan, tasks, validate, implement, (write: fixed) |
| F2 | Parallel sub-agent dispatch with no `Task`/`Agent` tool allowed | **HIGH** | review, implement |
| F3 | MCP-mandatory gate contradicts "subagents can't call MCP" | **HIGH** | review, implement |
| F4 | External library/version re-verification missing downstream | **HIGH** | plan, tasks, validate, implement |
| F5 | Step-numbering scheme inconsistent (`/7` with 9 commands) | MEDIUM | all |
| F6 | `allowed-tools` frontmatter inconsistent across commands | MEDIUM | all |
| F7 | Semble-mandate text duplicated 2–3× per command | MEDIUM (tokens) | all |
| F8 | MCP Evidence Log template triplicated | MEDIUM (tokens) | tasks, implement |
| F9 | `prd-review` reviewer #4 inlines 8 language sub-reviewers | MEDIUM (tokens) | review |
| F10 | Two divergent complexity taxonomies | LOW | discover vs plan |
| F11 | "Never run lint" conflicts with host project rule | LOW | implement |
| F12 | Misc (node-only version bump, mixed dates, Bash-find vs MCP) | LOW | release, implement |

---

## F1 — Invented MCP tool names (CRITICAL)

The commands instruct the agent to call MCP tools that **do not exist**. The real tools (from the live MCP servers) are:

| Used in commands (WRONG) | Actual tool |
|---|---|
| `mcp__serena__get_overview` | `mcp__serena__get_symbols_overview` |
| `mcp__serena__get_symbol_info` | `mcp__serena__find_symbol` (with `include_info: true`) — no `get_symbol_info` exists |
| `mcp__serena__get_related_symbols` | `mcp__serena__find_referencing_symbols` — no `get_related_symbols` exists |
| `mcp__octocode__search` | `mcp__octocode__localSearchCode` (local) / `githubSearchCode` (remote) |
| `mcp__octocode__get_file` | `mcp__octocode__localGetFileContent` / `githubGetFileContent` |
| `mcp__octocode__get_file_tree` | `mcp__octocode__localViewStructure` / `githubViewRepoStructure` |
| `mcp__octocode__github_search` | `mcp__octocode__githubSearchCode` |
| `mcp__context7__resolve_library_id` | `mcp__context7__resolve-library-id` |
| `mcp__context7__get_library_docs` | `mcp__context7__query-docs` |

**Where:**
- `prd-discover.md` — phase 1 `project-overview` (`get_overview`), `related-code` (`octocode__search`, `octocode__get_file`), `external-pattern-research` (`context7__resolve_library_id`, `get_library_docs`, `octocode__github_search`).
- `prd-plan.md` — phase 2 (`octocode__get_file_tree`, `octocode__search`, `serena__get_related_symbols`, `context7__get_library_docs`).
- `prd-tasks.md` — phase 2 (`serena__get_symbol_info`, `octocode__get_file`, `octocode__search`); `preferred_tools` block (`octocode__search`, `octocode__get_file`).
- `prd-validate.md` — phase 6 (`serena__get_symbol_info`, `serena__get_related_symbols`, `octocode__get_file`, `octocode__search`, `context7__get_library_docs`).
- `prd-implement.md` — phase 7 (`serena__get_symbol_info`, `serena__get_related_symbols`, `octocode__search`, `octocode__get_file`, `octocode__github_search`, `context7__get_library_docs`).

**Impact:** Every "MANDATORY MCP" instruction that names a nonexistent tool is unfollowable. The agent silently degrades to Bash/guessing — the exact failure the commands try to prevent. This is the highest-leverage fix.

**Recommendation:** Define the tool names **once** in a canonical block in `prd-discover.md` (like the existing `language-map`/`discovery-read-pattern`) and have every command reference it. Map: discovery/semantic → `semble.search`/`find_related`; symbols → `serena.find_symbol`/`get_symbols_overview`/`find_referencing_symbols`; local read/search → `octocode.localSearchCode`/`localGetFileContent`/`localViewStructure`; external → `octocode.githubSearchCode`/`githubGetFileContent`/`packageSearch` + `context7.resolve-library-id`/`query-docs`.

---

## F2 — Parallel sub-agent dispatch with no agent tool (HIGH)

- `prd-review.md` phase 2: "Run six specialist reviewers in parallel" with a `common-agent-contract`.
- `prd-implement.md` phase 5: "Dispatch each task concurrently with subagent_contract."

Neither command lists `Task` (or `Agent`) in `allowed-tools`:
- review: `Bash, mcp__serena, mcp__octocode, mcp__semble`
- implement: `mcp__serena, mcp__octocode, mcp__semble, mcp__context7, Bash`

**Impact:** The core parallelism the commands describe can't be invoked. The agent either runs reviewers/tasks sequentially in-process (silently violating the "parallel" design) or stalls.

**Recommendation:** Add the agent-dispatch tool to `allowed-tools` for review and implement, or rewrite both to state they run sequentially in-process.

---

## F3 — MCP-mandatory gate vs. "subagents can't call MCP" (HIGH)

The host project's `CLAUDE.md` states: *"Sub-agents cannot call MCP tools — they fall back to the Bash CLI (`semble search …`)."*

But:
- `prd-implement.md` phase 5: "Reject subagent output without MCP Evidence Log"; subagent_contract: "Use MCP before implementation."
- `prd-review.md` reviewers are subagents expected to use `mcp__semble`.

**Impact:** If subagents genuinely can't reach MCP, the hard "MCP Evidence Log or reject" gate is unsatisfiable, forcing an undocumented CLI fallback. The commands never tell subagents to use the `semble`/`uvx … semble` CLI fallback that CLAUDE.md documents.

**Recommendation:** In the subagent contracts, replace "use MCP" with "use the semble/octocode **CLI** fallback (subagents have no MCP); record CLI evidence in the Evidence Log." Keep the MCP mandate for the main agent only.

---

## F4 — No downstream external/version re-verification (HIGH)

`prd-write` now mandates Context7 + octocode-source + packageSearch + WebSearch for external symbols. But the agent that **writes code** (`prd-implement`) only has `library_docs condition="external library API not locally proven"` (with the wrong `get_library_docs` name, F1), and **no version/breaking-change recheck**. `prd-plan`/`prd-tasks`/`prd-validate` never re-confirm a pinned dependency version or that it's installed.

**Impact:** A version pinned in spec weeks earlier is used verbatim at implement time with no currency check; a hallucinated/renamed external API in spec survives to code.

**Recommendation:**
- `prd-validate`: add a check that every dependency named in spec exists in the manifest at the pinned version (or is flagged NEW-TO-INSTALL), and that external API names still resolve via Context7.
- `prd-implement` phase 7: require a `packageSearch`/manifest confirmation before writing code against a new dependency.

---

## F5 — Step-numbering inconsistent (MEDIUM)

Descriptions: discover `1/7`, write `2/7`, plan `3/7`, tasks `4/7`, validate `4a/7`, implement `5/7`, review `6/7`, pr `7/7`, **release `Step 8`**. Nine commands, denominator says 7, validate is a half-step `4a`, release falls outside the scheme.

**Recommendation:** Pick one scheme — e.g. `Step N/8` with validate as `4a` explicitly "gate, not a numbered step," or renumber to `/9`. Make all descriptions agree.

---

## F6 — `allowed-tools` inconsistent (MEDIUM)

| Command | allowed-tools |
|---|---|
| discover/plan/tasks/validate/implement/release | `mcp__serena, mcp__octocode, mcp__semble, mcp__context7, Bash` |
| write | + `WebSearch, WebFetch` (added) |
| review | `Bash, mcp__serena, mcp__octocode, mcp__semble` (no context7) |
| pr | `Bash, mcp__semble__search, mcp__semble__find_related` (granular) |

Three different styles (namespace vs granular), inconsistent ordering, web tools only on `write`, missing agent tool (F2), `context7` present where unused (release rarely needs it) and absent where arguably useful (review of external API usage).

**Recommendation:** Standardize ordering and granularity; add agent tool to review/implement; drop `context7` where no phase calls it.

---

## F7 — Semble-mandate duplicated 2–3× per command (MEDIUM, tokens)

The block *"mcp__semble is MANDATORY … 100x more token-efficient … call before every octocode/serena"* appears in `<system><rules>`, again in `<control><critical>`, and sometimes in a phase, in **every** command. ~16–20 near-identical restatements pipeline-wide.

**Recommendation:** State it once in the canonical block (F1) and reference it. Net token saving across the pack is significant since these files are loaded in full on invocation.

---

## F8 — MCP Evidence Log triplicated (MEDIUM, tokens)

The Evidence Log template appears in `prd-tasks` (`task_template` `evidence_log_required` + `preferred_tools`), and twice in `prd-implement` (phase 4 `evidence_log_template`, phase 11 `output`). Three near-identical ~10-line blocks.

**Recommendation:** Define one canonical Evidence Log shape; reference it.

---

## F9 — `prd-review` reviewer #4 inlines 8 language sub-reviewers (MEDIUM, tokens)

Reviewer #4 (type-safety) hard-codes full checklists+findings for TypeScript, Python, Go, Rust, Java, Kotlin, Ruby, plain JS, and "other" — ~190 lines, of which only **one** language applies to any given repo. The whole block loads every review.

**Recommendation:** Keep only a generic checklist inline; move per-language tables to a referenced appendix, or have the agent read `<language>` from discovery and load only the matching block.

---

## F10 — Two complexity taxonomies (LOW)

- `prd-discover` phase 0.5: `trivial | simple | complex` (file-count/api/schema/auth heuristics).
- `prd-plan` phase 5: `simple | moderate | complex` (DGI step-count heuristic).

Different names, different inputs, no cross-reference. A "simple" discovery may map to "moderate" planning with no defined bridge.

**Recommendation:** Either align the labels or explicitly document the mapping (discovery complexity → plan decomposition tier).

---

## F11 — "Never run lint" vs host project rule (LOW)

`prd-implement` hard rule: "Never run lint — lint is forbidden for all agents." The host `CLAUDE.md` says "run `bun run lint` after code changes" and `/prd-review` resolves a lint/typecheck command. The pack is generic, but the absolute prohibition can conflict with a project that gates on lint.

**Recommendation:** Soften to "agents don't run full lint mid-task; lint runs as a final gate (per project policy)" so it composes with host rules.

---

## F12 — Misc (LOW)

- `prd-release` phase 7 bumps `package.json` via `node -e …` — assumes Node is installed even for Rust/Go/Python projects. Guard on language, or use the language's own version tool.
- Date formats mixed: pipeline uses `dd-mm-yyyy`; changelog/release use `yyyy-mm-dd`. Contextually fine (ISO for changelog) but worth a one-line note so it's intentional, not drift.
- `prd-implement` interactive + resolve phases use Bash `find` for `index.md`/task files — acceptable (artifact discovery, not code search) but slightly at odds with the blanket "never use Bash find for discovery"; clarify the carve-out.
- `prd-discover` phase 0.5 `skip-phase ref="research|project-overview|external-pattern-research"` targets sub-tags, not formal phase IDs; tighten the refs so skip logic is unambiguous.

---

## What's already good (keep)

- The canonical `language-map` + `discovery-read-pattern` blocks in `prd-discover` phase 0.6 are exactly the right pattern — extend it to MCP tool names (F1), the semble mandate (F7), and the Evidence Log (F8).
- Evidence-ledger / `[UNVERIFIED]` discipline is strong and consistent across discover→write→plan→tasks→validate.
- Single-source-of-truth status logic in `prd-validate` phase 9 is well specified.
- Gate chaining (validate → implement → review → pr → release) is coherent and each gate is enforced.

---

## Suggested fix order (highest leverage first)

1. **F1** — correct all MCP tool names via one canonical block. (Unblocks every "mandatory MCP" instruction.)
2. **F2 + F3** — make parallel dispatch real (add agent tool) and reconcile subagent MCP/CLI reality.
3. **F4** — add downstream external/version re-verification to validate + implement.
4. **F7 + F8 + F9** — de-duplicate into referenced canonical blocks (large token win).
5. **F5 + F6 + F10 + F11 + F12** — consistency cleanups.
