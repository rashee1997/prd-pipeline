
<p align="center">
  <img src="./assets/logo.png" alt="PRD Pipeline logo" width="120" />
</p>

<p align="center">
  <strong>Evidence-first PRD workflow for AI coding agents.</strong>
</p>

<p align="center">
  <img alt="License" src="https://img.shields.io/badge/license-MIT-blue.svg">
  <img alt="Pipeline" src="https://img.shields.io/badge/pipeline-8\_commands-7C5CFF.svg">
  <img alt="MCP Ready" src="https://img.shields.io/badge/MCP-ready-22C55E.svg">
  <img alt="Validation Gated" src="https://img.shields.io/badge/validation-gated-F59E0B.svg">
  <img alt="Ponytail Discipline" src="https://img.shields.io/badge/ponytail-discipline-EC4899.svg">
</p>

An evidence-first, PRD-driven software delivery workflow for local AI coding agents. Turns a feature idea into researched discovery, PRD, technical spec, dependency plan, isolated tasks, implementation, specialist review, and pull request — **every name verified against live code** before a single requirement is written.

```text
/prd-discover → /prd-write → /prd-plan → /prd-tasks → /prd-validate → /prd-implement → /prd-review → /prd-pr
```

## Quick Start

```bash
git clone https://github.com/rashee1997/prd-pipeline.git
# Copy into your agent's commands directory (see docs/installation.md)
cp prd-*.md /path/to/your/agent/commands/

/prd-discover "your feature idea"
```

## Key Differentiators

| PRD Pipeline is the only SDD tool that... | Why it matters |
|---|---|
| **Verifies every name against live code** before writing requirements | Eliminates hallucinated symbols and wrong field names on brownfield projects |
| **Maps blast radius before specification** | Prevents silent contract breaks |
| **Gates implementation behind validation** | Catches unsafe tasks before any code is written |
| **Handles enhancements properly** (frozen contracts, Layer 0, cutover) | Designed for brownfield, not just greenfield |
| **Makes semantic search (Semble MCP) mandatory** | 100x more token-efficient than regex/grep |
| **Zero runtime dependencies** — pure portable Markdown | Copy into any agent runtime and run |

## Documentation

| File | Contents |
|---|---|
| [docs/installation.md](docs/installation.md) | Installation guides for OpenCode, Claude Code, Codex CLI, Antigravity, generic agents + required MCP tools |
| [docs/commands.md](docs/commands.md) | Full command reference, pipeline diagram, workflow, repo layout, output structure |
| [docs/principles.md](docs/principles.md) | Core principles (evidence-first, blast-radius, compatibility, ponytail discipline, when to use) |
| [docs/comparison.md](docs/comparison.md) | Comparison vs GitHub Spec Kit, OpenSpec, cc-sdd, and 10+ other SDD tools |

> **MCP tools:** The command files reference `mcp__serena`, `mcp__octocode`, `mcp__semble`, `mcp__context7`. If these are not installed or error, the AI automatically falls back to its own native tools for code research — the pipeline degrades gracefully.
>
> **⚠️ AI can make mistakes.** All generated artifacts require human review. Use the pipeline as an accelerator, not a replacement for careful human verification.

## License

MIT — see [LICENSE](./LICENSE).

## Status

Experimental but production-oriented. Use carefully, inspect generated artifacts, adapt to your codebase conventions.
