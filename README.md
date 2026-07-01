
<p align="center">
  <img src="./assets/logo.png" alt="PRD Pipeline logo" width="120" />
</p>

<p align="center">
  <strong>Evidence-first PRD workflow for AI coding agents.</strong>
</p>

<p align="center">
  ⚠️ <strong>AI can make mistakes.</strong> All generated artifacts require human review. Use as an accelerator, not a replacement for careful verification.
</p>

<p align="center">
  <img alt="License" src="https://img.shields.io/badge/license-MIT-blue.svg">
  <img alt="Pipeline" src="https://img.shields.io/badge/pipeline-9\_commands-7C5CFF.svg">
  <img alt="Complexity Routing" src="https://img.shields.io/badge/complexity-routing-22C55E.svg">
  <img alt="MCP Ready" src="https://img.shields.io/badge/MCP-ready-22C55E.svg">
  <img alt="Validation Gated" src="https://img.shields.io/badge/validation-gated-F59E0B.svg">
  <img alt="Ponytail Discipline" src="https://img.shields.io/badge/ponytail-discipline-EC4899.svg">
  <img alt="Greenfield Ready" src="https://img.shields.io/badge/greenfield-ready-0EA5E9.svg">
  <img alt="CoVe Verified" src="https://img.shields.io/badge/CoVe-verified-8B5CF6.svg">
</p>

An evidence-first, PRD-driven software delivery workflow for local AI coding agents. Turns a feature idea into researched discovery, PRD, technical spec, dependency plan, isolated tasks, implementation, specialist review, and pull request — **every name verified against live code** before a single requirement is written.

```text
/prd-discover → /prd-write → /prd-plan → /prd-tasks → /prd-validate → /prd-implement → /prd-review → /prd-pr
```

## Quick Start

```bash
git clone https://github.com/rashee1997/prd-pipeline.git
# Copy into your agent's commands directory (see docs/installation.md)
cp prd-*.md _shared.md /path/to/your/agent/commands/

# Brownfield (existing codebase)
/prd-discover "your feature idea"

# Greenfield (new project from scratch)
/prd-discover "your app idea" --greenfield
```

## Key Differentiators

| PRD Pipeline is the only SDD tool that... | Why it matters |
|---|---|
| **Verifies every name against live code** before writing requirements | Eliminates hallucinated symbols and wrong field names on brownfield projects |
| **Chain-of-Verification (CoVe) checkpoint** before spec is written | Re-verifies every cited symbol/path/route against live code — reduces hallucinated names by ~77% |
| **Maps blast radius before specification** | Prevents silent contract breaks |
| **Gates implementation behind validation** | Catches unsafe tasks before any code is written |
| **Fail-fast input gates** on every command | Stops immediately with a clear recovery message if required inputs are missing |
| **Handles enhancements properly** (frozen contracts, Layer 0, cutover) | Designed for brownfield, not just greenfield |
| **`--greenfield` mode** for new projects | Switches discovery to tech-stack/ADR questions; replaces live-code validation with spec-conformance scoring |
| **`--delta` spec mode** for small brownfield changes | Outputs only ADDED/MODIFIED/REMOVED — 60-80% smaller specs for 1-3 file changes |
| **Makes semantic search (Semble MCP) mandatory** | 100x more token-efficient than regex/grep |
| **Shared rules in `_shared.md`** — single source of truth | Update base execution rules, MCP policy, and verification protocol once; applies to all 9 commands |
| **Zero runtime dependencies** — pure portable Markdown | Copy into any agent runtime and run |

## Intelligent Optimization

| Feature | What it does | Saves |
|---|---|---|
| **Complexity routing** | Detects trivial vs complex tasks; skips deep research for ≤3-file changes | 60-80% token waste on simple tasks |
| **Conditional blast-radius** | Only researches dimensions with evidence of impact | ~40% smaller specs |
| **`--delta` spec mode** | Brownfield spec outputs ADDED/MODIFIED/REMOVED only; unchanged sections omitted | 60-80% smaller specs for small changes |
| **CoVe verification checkpoint** | Re-verifies all cited names before writing spec/PRD | ~77% fewer hallucinated symbol names |
| **EARS acceptance criteria** | Machine-parseable WHEN/WHILE/IF/WHERE + SHALL notation | Clearer test targets |
| **Content-hashed spec blocks** | SHA-256 anchors enable CI drift detection | Catches stale specs |
| **Decomposition guidance** | DGI* ≈ 0.85√S heuristic prevents over-decomposition | Avoids 71% coordination overhead |
| **Context budget per task** | Estimated token cap; tasks fail loudly instead of silently truncating | Prevents hallucinated code |
| **Orchestrator handoff** | ≤200 token summary for next phase | 70% less context carry-over |
| **Re-anchoring checkpoint** | Re-reads goal at 50% of task steps | Prevents agent drift mid-task |
| **Faithfulness check** | Verifies diff matches acceptance criteria, not just passes tests | Catches "fake done" |
| **Task output log** | Each completed task writes a brief deviation note to index.md; next task reads it | Prevents stale assumptions in sequential runs |
| **Shared rules (`_shared.md`)** | Single file for base execution, semble-first, UNVERIFIED protocol | One update propagates to all 9 commands |

## Documentation

| File | Contents |
|---|---|
| [docs/installation.md](docs/installation.md) | Installation guides for OpenCode, Claude Code, Codex CLI, Antigravity, generic agents + required MCP tools |
| [docs/commands.md](docs/commands.md) | Full command reference, pipeline diagram, workflow, repo layout, output structure |
| [docs/principles.md](docs/principles.md) | Core principles (evidence-first, blast-radius, compatibility, ponytail discipline, when to use) |
| [docs/comparison.md](docs/comparison.md) | Comparison vs GitHub Spec Kit, OpenSpec, cc-sdd, and 10+ other SDD tools |

> **MCP tools:** The command files reference `mcp__serena`, `mcp__octocode`, `mcp__semble`, `mcp__context7`. If these are not installed or error, the AI automatically falls back to its own native tools for code research — the pipeline degrades gracefully.

## License

MIT — see [LICENSE](./LICENSE).

## Status

Experimental but production-oriented. Use carefully, inspect generated artifacts, adapt to your codebase conventions.
