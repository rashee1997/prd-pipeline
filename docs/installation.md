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

## Required MCP / Research Tools

### 1. Codebase symbol/navigation tool (e.g. `mcp__serena`)

Used for: finding symbols, reading signatures, tracing callers, understanding architecture.

### 2. Code search and file reading tool (e.g. `mcp__octocode`)

Used for: searching local code, reading file contents, researching GitHub implementations.

### 3. Semantic code search tool (e.g. `mcp__semble`) — **Mandatory**

Used for: finding conceptually related code, discovering behavior with different names, finding adjacent modules. 100x more token-efficient than regex/grep.

### 4. Library documentation tool (e.g. `mcp__context7`)

Used for: verifying library APIs, checking exact method signatures, validating version-specific behavior.

### 5. Shell / Bash access

Used for: creating folders, writing files, running type checks/tests/builds, Git operations, GitHub CLI.

## Optional Tools

- GitHub CLI (`gh`) for `/prd-pr`
- GitHub authentication for remote repository research
- Language server support for stronger semantic navigation
- Test runner support for scoped task acceptance checks
