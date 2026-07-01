# Comparison to Other SDD Tools

The spec-driven development ecosystem has grown rapidly. Here is how PRD Pipeline compares to the most popular alternatives.

## Feature Comparison

| Feature | PRD Pipeline | GitHub Spec Kit | OpenSpec | cc-sdd | Spec-Driven Develop |
|---|---|---|---|---|---|
| **Stars** | — | 115K | 55K | ~1.8K | 912 |
| **Runtime** | None (Markdown) | Python | Node.js | Node.js | Shell |
| **Setup time** | Copy files → run | `uvx` init (~5-10 min) | `npm install` (~2 min) | `npx` init (~2 min) | Git clone (~2 min) |
| **Brownfield focus** | ✅ Primary design | 🟡 Generic | 🟡 Generic | 🟡 Generic | 🟡 Generic |
| **Evidence-first** (verify names against live code) | ✅ **Unique** — every name must trace to a tool result | ❌ Assumes spec text is correct | ❌ Assumes spec text is correct | ❌ Assumes spec text is correct | ❌ Assumes spec text is correct |
| **Blast-radius mandatory** | ✅ **Unique** — required before any spec writing | ❌ Not enforced | ❌ Not enforced | ❌ Not enforced | ❌ Not enforced |
| **Complexity routing** (skip deep phases for trivial tasks) | ✅ Proportional — trivial/simple/complex phases | ❌ Uniform depth | ❌ Uniform depth | ❌ Uniform depth | ❌ Uniform depth |
| **Conditional blast-radius** (skip irrelevant dimensions) | ✅ **Unique** — zero-evidence dimensions skipped | ❌ Always full template | ❌ Always full template | ❌ Always full template | ❌ Always full template |
| **EARS acceptance criteria** | ✅ Mandated in template + rules | 🟡 Recommended | ❌ Prose | ❌ Prose | 🟡 Recommended |
| **Content-hashed spec blocks** (drift detection) | ✅ SHA-256 anchors in drift_anchors section | ❌ No drift detection | ❌ No drift detection | ❌ No drift detection | ❌ No drift detection |
| **Decomposition granularity guidance** | ✅ DGI* ≈ 0.85√S heuristic | ❌ Static decomposition | ❌ Static decomposition | ❌ Static decomposition | ❌ Static decomposition |
| **Orchestrator handoff** (compact ≤200t summary) | ✅ **Unique** — standardized handoff block | ❌ Files passed by reference | ❌ Files passed by reference | ❌ Files passed by reference | ❌ Files passed by reference |
| **Context budget per task** | ✅ Estimated tokens + hard cap; fails loudly | ❌ No budget | ❌ No budget | ❌ No budget | ❌ No budget |
| **Re-anchoring checkpoint** (anti-drift at 50%) | ✅ Goal re-read at midpoint | ❌ No drift prevention | ❌ No drift prevention | ❌ No drift prevention | ❌ No drift prevention |
| **Faithfulness check** (diff matches intent, not just tests) | ✅ Spec-compliance reviewer checks faithfulness | ❌ No intent verification | ❌ No intent verification | ❌ No intent verification | ❌ No intent verification |
| **File size enforcement** (split past ~350 lines) | ✅ Hard rule in plan/tasks/implement | ❌ No file limits | ❌ No file limits | ❌ No file limits | ❌ No file limits |
| **Validation gate** (before implementation) | ✅ `/prd-validate` blocks if tasks not ready | ❌ No validation gate | ❌ No validation gate | ❌ No validation gate | ❌ No validation gate |
| **Fail-fast input gates** on every command | ✅ **Unique** — every command stops with a clear recovery message if inputs are missing | ❌ Silent failure | ❌ Silent failure | ❌ Silent failure | ❌ Silent failure |
| **Task isolation** (fresh subagent ready) | ✅ **Unique** — every task is self-contained, 300-600 words | 🟡 Tasks reference spec/plan | 🟡 Tasks reference parent docs | 🟡 Tasks reference parent docs | 🟡 Tasks reference parent docs |
| **Task output log** | ✅ Deviation notes written to index.md; next task reads them | ❌ | ❌ | ❌ | ❌ |
| **Enhancement/compat mode** | ✅ **Unique** — frozen contracts, Layer 0, cutover | ❌ No enhancement mode | ❌ No enhancement mode | ❌ No enhancement mode | ❌ No enhancement mode |
| **Greenfield mode** (`--greenfield`) | ✅ Switches to ADR/tech-stack Q&A + spec-conformance validation | ❌ | ❌ | ❌ | ❌ |
| **Delta spec mode** (`--delta`) | ✅ ADDED/MODIFIED/REMOVED only — 60-80% smaller brownfield specs | ❌ Always full spec | ❌ Always full spec | ❌ Always full spec | ❌ Always full spec |
| **CoVe verification** (chain-of-verification) | ✅ Re-verifies every cited name before writing spec (~77% fewer hallucinated symbols) | ❌ No re-verification | ❌ No re-verification | ❌ No re-verification | ❌ No re-verification |
| **Shared rules (`_shared.md`)** | ✅ Single source of truth; update once, applies to all commands | ❌ Rules duplicated per command | ❌ | ❌ | ❌ |
| **Ponytail discipline** | ✅ Built into every command | ❌ | ❌ | ❌ | ❌ |
| **Semantic search (Semble) integration** | ✅ **Unique** — mandatory, token-efficient MCP | ❌ Regex/grep based | ❌ Regex/grep based | ❌ Regex/grep based | ❌ Regex/grep based |
| **TDD mode** | ✅ `--tdd` flag in planning | 🟡 Via extensions | ❌ | ❌ | ❌ |
| **Parallel layer execution** | ✅ `--parallel-layer N` | ❌ | ❌ | 🟡 | ✅ Worktrees |
| **Multi-agent support** | OpenCode, Claude Code, Codex CLI, Gemini CLI, Antigravity, generic | 30+ agents | 20+ agents | 8 agents | 8+ agents |
| **Review phase** | ✅ 6 specialist reviewers (security, perf, TS, quality, tests, spec) | ❌ No built-in review | ❌ No built-in review | ❌ No built-in review | ❌ No built-in review |
| **PR creation** | ✅ Auto-generates PR from artifacts | ✅ | ❌ | ❌ | ✅ |
| **Release management** | ✅ `/prd-release` with changelog + version + GitHub Release | ❌ | ❌ | ❌ | ❌ |
| **License** | MIT | MIT | MIT | MIT | MIT |

## When to Choose PRD Pipeline

**Choose PRD Pipeline if you work on brownfield codebases** where hallucinated names, broken contracts, and missing blast-radius analysis cause real damage. It is the only SDD tool that:

1. **Verifies every name against live code** before writing a single requirement — no "close enough" symbols, no plausible-looking field names
2. **Maps blast radius before specification** — forces you to know what API routes, DB tables, auth boundaries, UI surfaces, workflows, and tests are affected before writing PRD/spec
3. **Gates implementation behind validation** — tasks must pass `/prd-validate` before any code is written
4. **Handles enhancements properly** — frozen contracts, compatibility Layer 0, regression-first, cutover tasks
5. **Uses semantic search (Semble MCP) for token efficiency** — 100x less token waste finding code than regex/grep-based tools

**Choose another tool if:**
- You want an installer/CLI (`npm install`) rather than copying Markdown files
- You need 30+ agent integrations out of the box
- You prefer lighter process and are willing to trade safety for speed

> **Greenfield note:** PRD Pipeline now supports new projects via `--greenfield` flag on `/prd-discover` and `/prd-validate`. It skips live-code research and switches to external pattern research + spec-conformance validation. Full greenfield scaffold generation (boilerplate, CI setup) is outside scope — use a scaffolding tool alongside.

## Related Projects

| Project | Stars | Description |
|---|---|---|
| [GitHub Spec Kit](https://github.com/github/spec-kit) | 115K | The most popular SDD toolkit (Python, 30+ agents) |
| [OpenSpec](https://github.com/Fission-AI/OpenSpec) | 55K | Lightweight SDD framework (npm, 20+ agents) |
| [Spec-Driven Develop](https://github.com/zhu1090093659/spec_driven_develop) | 912 | Architecture-first SDD with GitHub Issues integration |
| [cc-sdd](https://github.com/gotalab/claude-code-spec) | ~1.8K | Claude Code SDD plugin with parallel execution |
| [SpecDD](https://github.com/specdd/specdd) | 19 | Agent-agnostic `.sdd` file framework |
| [SpecD](https://github.com/specd-sdd/SpecD) | 10 | Code-graph RAG powered SDD |
| [SpecAnchor](https://github.com/linziyanleo/spec-anchor) | 15 | Three-tier spec system with context construction |
| [Specforce Kit](https://github.com/jeancodogno/specforce-kit) | 4 | Go-based SDD orchestration layer |
| [SpecLoom](https://github.com/ruchit07/specloom) | 0 | Token-budgeted context bundles |
| [Doctrina](https://github.com/GCarin1/Doctrina) | 1 | AGENTS.md-native SDD with CLI |
| [Ponytail](https://github.com/DietrichGebert/ponytail) | — | Lazy senior developer rules (philosophy inspiration) |
| [Superpowers](https://github.com/obra/superpowers) | — | Agentic skills framework |
