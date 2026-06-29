# Installation

PRD Pipeline is a collection of portable Markdown command files. No runtime dependencies, no npm/pip install — just copy the files into your agent's command directory.

```bash
git clone https://github.com/rashee1997/prd-pipeline.git
```

## By Agent Runtime

### OpenCode

```bash
# Project-level
cd /path/to/your/project
mkdir -p .opencode/commands
cp /path/to/prd-pipeline/prd-*.md .opencode/commands/

# Or global
mkdir -p ~/.config/opencode/commands
cp /path/to/prd-pipeline/prd-*.md ~/.config/opencode/commands/
```

Run: `/prd-discover "your feature idea"`

### Claude Code

```bash
# Project-level
cd /path/to/your/project
mkdir -p .claude/commands
cp /path/to/prd-pipeline/prd-*.md .claude/commands/

# Or personal (global)
mkdir -p ~/.claude/commands
cp /path/to/prd-pipeline/prd-*.md ~/.claude/commands/
```

Run: `/prd-discover "your feature idea"`

### Codex CLI

```bash
# Legacy custom prompts (deprecated by OpenAI)
mkdir -p ~/.codex/prompts
cp /path/to/prd-pipeline/prd-*.md ~/.codex/prompts/
```

Note: Codex recommends using skills instead of custom prompts. Adapt command files into Codex skills manually.

### Antigravity CLI

Suggested plugin layout:

```text
~/.gemini/antigravity-cli/plugins/prd-pipeline/
├── plugin.json
└── skills/
    ├── prd-discover.md
    ├── prd-write.md
    ├── prd-plan.md
    ├── prd-tasks.md
    ├── prd-validate.md
    ├── prd-implement.md
    ├── prd-review.md
    └── prd-pr.md
```

Example `plugin.json`:

```json
{
  "$schema": "https://antigravity.google/schemas/v1/plugin.json",
  "name": "prd-pipeline",
  "description": "Evidence-first PRD pipeline commands for agentic software development."
}
```

```bash
git clone https://github.com/rashee1997/prd-pipeline.git
mkdir -p ~/.gemini/antigravity-cli/plugins/prd-pipeline/skills
cp /path/to/prd-pipeline/prd-*.md ~/.gemini/antigravity-cli/plugins/prd-pipeline/skills/
cat > ~/.gemini/antigravity-cli/plugins/prd-pipeline/plugin.json <<'EOF'
{
  "$schema": "https://antigravity.google/schemas/v1/plugin.json",
  "name": "prd-pipeline",
  "description": "Evidence-first PRD pipeline commands for agentic software development."
}
EOF
```

Then restart or reload Antigravity CLI.

### Generic Markdown Command Agents

1. Clone the repository: `git clone https://github.com/rashee1997/prd-pipeline.git`
2. Create the runtime's command/skill directory.
3. Copy each `prd-*.md` file into that directory.
4. Confirm filenames map to slash commands.
5. Update tool names in frontmatter if your MCP names differ.
6. Run `/prd-discover "feature idea"`.

Minimum required files:

```
prd-discover.md
prd-write.md
prd-plan.md
prd-tasks.md
prd-validate.md
prd-implement.md
prd-review.md
prd-pr.md
prd-release.md
```

## Tool Dependencies

The command files reference specific MCP tool families (`mcp__serena`, `mcp__octocode`, `mcp__semble`, `mcp__context7`). These are **optional** — if a tool is unavailable in your environment, the agent must use its own equivalent built-in capabilities (native file search, symbol lookup, code reading, web search, etc.).

| Tool family | Purpose | Fallback if unavailable |
|---|---|---|
| `mcp__serena` | Symbol/navigation — find symbols, read signatures, trace callers | Agent's native symbol lookup or code-reading tools |
| `mcp__octocode` | Code search + file reading — search local code, read files, GitHub research | Agent's native `grep`/`read`/`search` capabilities |
| `mcp__semble` (recommended) | Semantic code search — find code by concept/natural language. 100x more token-efficient than regex/grep | Agent's native file search + grep; less efficient but functional |
| `mcp__context7` | Library documentation — verify API signatures, version-specific behavior | Agent's web search or training knowledge |
| Bash / Shell | Creating folders, writing files, running type checks/tests/builds, Git operations, GitHub CLI | — |

**Important:** If a referenced MCP tool is not installed, the agent should not fail — it must use its own native capabilities instead. The pipeline is designed to degrade gracefully. The `allowed-tools` frontmatter in each command file lists the preferred tools, but agents with different tool names can adapt.

### Recommended (for best experience)

- **mcp__semble** — semantic code search. Reduces token consumption ~100x vs regex/grep for code discovery. Install: `claude mcp add semble -s user -- uvx --from "semble[mcp]" semble`
- **GitHub CLI (`gh`)** — required for `/prd-pr` and `/prd-release` GitHub Release creation
- **GitHub authentication** — for remote repository research via OctoCode
- **Language server** — for stronger semantic navigation
- **Test runner** — for scoped task acceptance checks
