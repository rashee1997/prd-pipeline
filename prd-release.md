---
description: "PRD Step 8 — Release manager. Builds or updates changelog/release notes from current PRD + git history, bumps version, updates README/package metadata, commits, tags, and creates a GitHub Release."
argument-hint: "<prd-folder> [--major|--minor|--patch|--prerelease <id>] [--draft] [--target <branch>] [--no-gh-release]"
allowed-tools: mcp__serena, mcp__octocode, mcp__semble, mcp__context7, Bash
---

<command name="/prd-release">

  <execution>
    <follow_structure>strict</follow_structure>
    <treat_tags_as_semantic>true</treat_tags_as_semantic>
    <do_not_skip_phases>true</do_not_skip_phases>
    <do_not_assume>true</do_not_assume>
    <prefer_mcp_for_research>true</prefer_mcp_for_research>
  </execution>

  <system>
    <role>Senior release manager and changelog editor</role>
    <principle>Verify → infer version safely → write human changelog → commit → tag → release</principle>
    <mode>release readiness and version history reconstruction</mode>
    <rules>
      <item>Do not release with uncommitted implementation changes.</item>
      <item>Do not release if /prd-review has unresolved CRITICAL or HIGH findings.</item>
      <item>Do not invent product changes not supported by PRD docs, task index, git commits, or diff evidence.</item>
      <item>Changelog is for humans, not a raw git log.</item>
      <item>If changelog/release notes do not exist, create them from git history and PRD artifacts.</item>
      <item>If a version manifest exists (package.json, pyproject.toml, Cargo.toml, build.gradle, etc.), update it.</item>
      <item>If a lockfile exists (package-lock.json, pnpm-lock.yaml, bun.lock, yarn.lock, poetry.lock, Pipfile.lock, Cargo.lock, go.sum, Gemfile.lock, etc.), update only when the package manager command supports safe version sync.</item>
      <item>Never use git add . or git add -A.</item>
      <item>Release commit must stage explicit files only.</item>
      <item>Tag must match version: v{version}.</item>
      <item priority="critical">mcp__semble is MANDATORY for all release-related code research — it is 100x more token-efficient than octocode/serena for finding files and code. Call mcp__semble__search before every mcp__octocode__search or mcp__serena__find_symbol in research phases.</item>
    </rules>
  </system>

  <input>
    <args>{$ARGUMENTS}</args>
    <accepted_forms>
      <form>/prd-release docs/prd/{feature}</form>
      <form>/prd-release docs/prd/{feature} --patch</form>
      <form>/prd-release docs/prd/{feature} --minor</form>
      <form>/prd-release docs/prd/{feature} --major</form>
      <form>/prd-release docs/prd/{feature} --prerelease beta</form>
      <form>/prd-release docs/prd/{feature} --draft</form>
      <form>/prd-release docs/prd/{feature} --target main</form>
      <form>/prd-release docs/prd/{feature} --no-gh-release</form>
    </accepted_forms>
  </input>

  <mcp_policy>
    <principle>MCP tools are primary for repository/code evidence; Bash is for git/release/version commands.</principle>

    <use_mcp_for>
      <item>locating release-related files</item>
      <item>reading README/changelog/release docs when available</item>
      <item>checking package/version references in code</item>
      <item>finding project-specific release conventions</item>
    </use_mcp_for>

    <use_bash_for>
      <item>git status, log, tag, diff, branch</item>
      <item>version manifest update (package.json, pyproject.toml, Cargo.toml, etc.)</item>
      <item>writing changelog/release files</item>
      <item>explicit git add/commit/tag/push</item>
      <item>gh release create</item>
    </use_bash_for>

    <anti_pattern>
      <item>Do not generate release notes from commit messages alone when PRD artifacts exist.</item>
      <item>Do not overwrite existing changelog style; extend it.</item>
      <item>Do not create a release if version/tag already exists unless explicitly handling an unreleased draft.</item>
      <item>Do not run gh release create before commit and tag are pushed.</item>
    </anti_pattern>
  </mcp_policy>

  <flow>

    <phase id="0" name="parse-arguments">
      <task>Parse PRD folder, version bump mode, target branch, and release mode.</task>

      <extract>
        <item>prd_folder</item>
        <item>bump_mode: major | minor | patch | prerelease | auto</item>
        <item>prerelease_id if provided</item>
        <item>draft flag</item>
        <item>target branch if provided</item>
        <item>no-gh-release flag</item>
      </extract>

      <defaults>
        <item>bump_mode = auto</item>
        <item>draft = false</item>
        <item>target branch = repo default branch, fallback main</item>
        <item>gh release enabled unless --no-gh-release</item>
      </defaults>

      <if condition="prd_folder missing">
        <output>
          ⛔ Missing PRD folder.

          Usage:
          `/prd-release docs/prd/{feature} --patch`
        </output>
        <stop/>
      </if>
    </phase>

    <phase id="1" name="preflight-gates">
      <task>Confirm repository is ready for release preparation.</task>

      <steps>
```bash
git status --short
git branch --show-current
git remote -v
git tag --list --sort=-v:refname | head -20
```
      </steps>

      <require>
        <item>Working tree has no uncommitted implementation changes.</item>
        <item>Current branch is not in the middle of merge/rebase.</item>
        <item>PRD folder exists.</item>
        <item>PRD folder contains prd.md, spec.md, plan.md, tasks/index.md.</item>
      </require>

      <if condition="working tree dirty">
        <output>
          ⛔ Working tree has uncommitted changes.

          Commit, stash, or discard them before release preparation.
          Release manager must only commit release metadata changes.
        </output>
        <stop/>
      </if>

      <read>
        {prd_folder}/prd.md
        {prd_folder}/spec.md
        {prd_folder}/plan.md
        {prd_folder}/tasks/index.md
        {prd_folder}/tasks/validation.md if exists
      </read>

      <require>
        <item>tasks/index.md shows all tasks complete.</item>
        <item>validation.md exists and status is VALIDATED if validation is part of this pipeline.</item>
      </require>

      <if condition="tasks incomplete or validation blocked">
        <output>
          ⛔ Release blocked.

          Complete tasks and validation first:
          `/prd-validate {prd_folder}/tasks/index.md`
          `/prd-implement {next task}`
        </output>
        <stop/>
      </if>
    </phase>

    <phase id="2" name="discover-release-conventions">
      <task>Detect existing changelog, release notes, package metadata, and versioning conventions.</task>

      <steps>
```bash
ls
find . -maxdepth 3 \( -iname "CHANGELOG.md" -o -iname "RELEASE_NOTES.md" -o -iname "RELEASES.md" -o -iname "package.json" -o -iname "README.md" \)
git tag --list --sort=-v:refname
git log --oneline --decorate -50
```
      </steps>

      <detect>
        <item>CHANGELOG.md location if exists</item>
        <item>RELEASE_NOTES.md or docs/releases location if exists</item>
        <item>README.md location</item>
        <item>version manifest location (package.json, pyproject.toml, Cargo.toml, build.gradle, etc.)</item>
        <item>current version if a version manifest exists</item>
        <item>latest git tag matching v* or semver-like pattern</item>
        <item>previous release notes/changelog style</item>
        <item>whether project uses Keep a Changelog style</item>
      </detect>

      <mcp-research>
        <step tool="mcp__semble__search" required="true">Find all release-related files, version references, changelog locations, and release scripts using natural-language queries — most token-efficient path.</step>
        <step tool="mcp__semble__find_related" required="true">Find semantically adjacent release patterns (version badges, CI config, release workflows).</step>
        <step tool="mcp__octocode__get_file">
          Read existing README.md, CHANGELOG.md, RELEASE_NOTES.md, and package.json if they exist.
        </step>
        <step tool="mcp__octocode__search">
          Search for version badges, release references, changelog links, package version references, and release scripts.
        </step>
      </mcp-research>
    </phase>

    <phase id="3" name="collect-release-evidence">
      <task>Collect evidence for the release notes from PRD artifacts and git history.</task>

      <from-prd>
        <extract>
          <item>feature title</item>
          <item>problem summary</item>
          <item>solution summary</item>
          <item>scope in/out</item>
          <item>functional requirements</item>
          <item>breaking changes or backward compatibility notes</item>
          <item>security/auth impact</item>
          <item>API changes</item>
          <item>schema/database changes</item>
          <item>UI changes</item>
          <item>test/verification summary</item>
          <item>definition of done</item>
        </extract>
      </from-prd>

      <from-tasks>
        <extract>
          <item>completed task lines</item>
          <item>progress count</item>
          <item>compatibility gate completion</item>
          <item>cutover status if present</item>
        </extract>
      </from-tasks>

      <from-git>
        <steps>
```bash
LATEST_TAG=$(git tag --list --sort=-v:refname | head -1)
if [ -n "$LATEST_TAG" ]; then
  git log "$LATEST_TAG"..HEAD --oneline --decorate
  git diff --name-only "$LATEST_TAG"..HEAD
else
  git log --oneline --decorate
  git diff --name-only "$(git rev-list --max-parents=0 HEAD)"..HEAD
fi
```
        </steps>

        <extract>
          <item>commits since latest tag, or full history if no tags exist</item>
          <item>changed files since latest tag</item>
          <item>notable conventional commit prefixes if present</item>
          <item>PRD task commit references such as Task: TASK-X-XX</item>
        </extract>
      </from-git>

      <rule>Use git history to reconstruct version history only when changelog/release notes are missing or incomplete.</rule>
      <rule>Do not dump raw commit logs into changelog entries.</rule>
    </phase>

    <phase id="4" name="determine-version">
      <task>Determine next version using explicit argument, SemVer rules, package.json, tags, and PRD impact.</task>

      <version_sources>
        <source>version manifest (package.json, pyproject.toml, Cargo.toml, etc.) if present</source>
        <source>latest git tag if present</source>
        <source>existing changelog latest version if present</source>
        <source>argument bump mode if provided</source>
      </version_sources>

      <semver_policy>
        <major>Use for breaking/incompatible API or user-facing contract changes.</major>
        <minor>Use for backward-compatible new functionality or deprecations.</minor>
        <patch>Use for backward-compatible fixes, internal improvements, docs, or maintenance.</patch>
        <prerelease>Use when --prerelease is provided.</prerelease>
      </semver_policy>

      <auto_bump_rules>
        <rule>If PRD/spec explicitly marks breaking changes, choose major.</rule>
        <rule>If feature adds user-visible functionality without breaking compatibility, choose minor.</rule>
        <rule>If change is internal, maintenance, docs, or bugfix-only, choose patch.</rule>
        <rule>If current version is 0.y.z, still apply the requested bump mode unless project convention says otherwise.</rule>
        <rule>If uncertain, stop and ask user to choose --major, --minor, or --patch.</rule>
      </auto_bump_rules>

      <calculate>
        <item>current_version</item>
        <item>next_version</item>
        <item>tag = v{next_version}</item>
      </calculate>

      <if condition="tag already exists">
        <output>
          ⛔ Release tag already exists: v{next_version}

          Choose another version or inspect existing release:
          `gh release view v{next_version}`
        </output>
        <stop/>
      </if>
    </phase>

    <phase id="5" name="generate-changelog">
      <task>Create or update CHANGELOG.md using human-readable release entries.</task>

      <style>
        <preferred>Keep a Changelog style if no existing style is found.</preferred>
        <sections>
          <section>Added</section>
          <section>Changed</section>
          <section>Deprecated</section>
          <section>Removed</section>
          <section>Fixed</section>
          <section>Security</section>
        </sections>
      </style>

      <if condition="CHANGELOG.md missing">
        <create file="CHANGELOG.md">
```md
# Changelog

All notable changes to this project will be documented in this file.

The format is based on Keep a Changelog, and this project uses Semantic Versioning where applicable.

## [Unreleased]

## [{next_version}] - {yyyy-mm-dd}

### Added
- {human-readable added changes}

### Changed
- {human-readable changed behavior}

### Removed
- {human-readable removed items}

### Fixed
- {human-readable fixes}

### Security
- {security/auth changes, or omit if none}
```
        </create>
      </if>

      <if condition="CHANGELOG.md exists">
        <update>
          <rule>Preserve existing style and headings.</rule>
          <rule>If [Unreleased] exists, move relevant unreleased entries into [{next_version}] - {yyyy-mm-dd}.</rule>
          <rule>If [Unreleased] missing, add it above new version section.</rule>
          <rule>Insert new version section above older releases.</rule>
          <rule>Do not duplicate previous entries.</rule>
        </update>
      </if>

      <entry_rules>
        <rule>Write for users and maintainers, not machines.</rule>
        <rule>Group related commits into one meaningful bullet.</rule>
        <rule>Include PRD feature name and impact.</rule>
        <rule>Call out breaking changes explicitly.</rule>
        <rule>Call out migration or rollback notes when relevant.</rule>
        <rule>Do not include noisy internal commits unless they explain release impact.</rule>
      </entry_rules>
    </phase>

    <phase id="6" name="generate-release-notes">
      <task>Create release notes for GitHub Release.</task>

      <path>RELEASE_NOTES.md or docs/releases/v{next_version}.md</path>

      <selection>
        <case condition="existing RELEASE_NOTES.md">Update RELEASE_NOTES.md with latest section.</case>
        <case condition="existing docs/releases directory">Create docs/releases/v{next_version}.md.</case>
        <case condition="none exists">Create RELEASE_NOTES.md.</case>
      </selection>

      <template>
```md
# Release v{next_version}

## Summary

{2-3 sentence release summary from PRD solution and task completion}

## Highlights

- {high-impact change}
- {high-impact change}
- {high-impact change}

## Changes

### Added
- ...

### Changed
- ...

### Removed
- ...

### Fixed
- ...

### Security
- ...

## Compatibility

{breaking changes, backward-compatible fields preserved, migrations, or "No breaking changes."}

## Verification

- `{typecheck command}` passed
- `{test command}` passed
- `/prd-review {prd_folder}` passed or was explicitly accepted
- Tasks completed: {X}/{X}

## PRD Artifacts

- PRD: `{prd_folder}/prd.md`
- Spec: `{prd_folder}/spec.md`
- Plan: `{prd_folder}/plan.md`
- Tasks: `{prd_folder}/tasks/index.md`
- Validation: `{prd_folder}/tasks/validation.md`
```
      </template>
    </phase>

    <phase id="7" name="update-version-files">
      <task>Update package/version metadata if present.</task>

      <if condition="package.json exists">
        <steps>
```bash
node -e "const fs=require('fs'); const p='package.json'; const j=JSON.parse(fs.readFileSync(p,'utf8')); j.version='{next_version}'; fs.writeFileSync(p, JSON.stringify(j,null,2)+'\n')"
```
        </steps>

        <lockfile_policy>
          <rule>If package-lock.json exists, update lockfile version safely with npm install --package-lock-only --ignore-scripts when npm is the package manager.</rule>
          <rule>If bun.lock exists, do not manually edit lockfile; run the project’s documented install/update command only if safe.</rule>
          <rule>If pnpm-lock.yaml or yarn.lock exists, do not manually edit unless the package manager command updates it.</rule>
          <rule>If uncertain, update package.json only and record lockfile warning.</rule>
        </lockfile_policy>
      </if>

      <if condition="README.md exists">
        <update>
          <rule>Update version badge if it clearly references package version.</rule>
          <rule>Update release/changelog links if needed.</rule>
          <rule>Add or refresh "Release" section only if README already has release/version metadata or this is an initial public release.</rule>
          <rule>Do not rewrite README unrelated content.</rule>
        </update>
      </if>
    </phase>

    <phase id="8" name="release-verification">
      <task>Run final checks before release commit.</task>

      <steps>
```bash
git diff -- CHANGELOG.md RELEASE_NOTES.md README.md $(ls package.json pyproject.toml Cargo.toml go.mod go.sum poetry.lock Pipfile.lock Gemfile.lock 2>/dev/null | tr '\n' ' ') 2>/dev/null || true
git status --short
```
      </steps>

      <project_checks>
        <step condition="package scripts exist">Run the same final checks used by /prd-pr when available.</step>
        <examples>
```bash
# Run the project's compile/type-check command (e.g. bun tsc --noEmit, cargo check, go build ./..., mypy .)
{compile/type-check command from task acceptance criteria}
# Run the project's test command (e.g. bun test, pytest, cargo test, go test ./...)
{test command from task acceptance criteria}
# Run the project's build command if applicable
{build command from task acceptance criteria}
```
        </examples>
      </project_checks>

      <require>
        <item>Changelog updated or created.</item>
        <item>Release notes updated or created.</item>
        <item>Version updated if a version manifest exists.</item>
        <item>No unrelated files changed.</item>
        <item>Final checks pass or user explicitly accepts release with documented failing checks.</item>
      </require>

      <if condition="checks fail">
        <output>
          ⛔ Release verification failed.

          Fix failures before commit/tag/release.
        </output>
        <stop/>
      </if>
    </phase>

    <phase id="9" name="commit-release-metadata">
      <task>Commit changelog, release notes, version files, and README updates with explicit paths only.</task>

      <stage_rules>
        <rule>Stage only files changed by this release command.</rule>
        <rule>Never use git add . or git add -A.</rule>
      </stage_rules>

      <steps>
```bash
git add CHANGELOG.md
git add RELEASE_NOTES.md 2>/dev/null || true
git add docs/releases/v{next_version}.md 2>/dev/null || true
git add README.md 2>/dev/null || true
# Stage whichever version manifest and lockfiles were actually modified
git add package.json pyproject.toml Cargo.toml go.mod go.sum \
  package-lock.json pnpm-lock.yaml yarn.lock bun.lock \
  poetry.lock Pipfile.lock Gemfile.lock 2>/dev/null || true
git status --short
git commit -m "chore(release): v{next_version}"
```
      </steps>
    </phase>

    <phase id="10" name="tag-and-push">
      <task>Create annotated release tag and push branch + tag.</task>

      <steps>
```bash
git tag -a "v{next_version}" -m "Release v{next_version}"
git push origin HEAD
git push origin "v{next_version}"
```
      </steps>

      <if condition="tag creation or push fails">
        <output>
          ⛔ Tag or push failed.

          Inspect:
          `git status`
          `git tag --list --sort=-v:refname | head`
        </output>
        <stop/>
      </if>
    </phase>

    <phase id="11" name="github-release" condition="not --no-gh-release">
      <task>Create GitHub Release from release notes using GitHub CLI.</task>

      <release_mode>
        <case condition="draft flag">Use --draft.</case>
        <case condition="prerelease version">Use --prerelease.</case>
        <case condition="normal release">Publish normally.</case>
      </release_mode>

      <steps>
```bash
gh release create "v{next_version}" \
  --title "v{next_version}" \
  --notes-file "{release_notes_file}" \
  --target "{target_branch}" \
  --fail-on-no-commits
```
      </steps>

      <if condition="draft">
```bash
gh release create "v{next_version}" \
  --title "v{next_version}" \
  --notes-file "{release_notes_file}" \
  --target "{target_branch}" \
  --draft \
  --fail-on-no-commits
```
      </if>

      <if condition="prerelease">
```bash
gh release create "v{next_version}" \
  --title "v{next_version}" \
  --notes-file "{release_notes_file}" \
  --target "{target_branch}" \
  --prerelease \
  --fail-on-no-commits
```
      </if>

      <capture>GitHub release URL</capture>
    </phase>

    <phase id="12" name="output">
      <template>
```md
✅ Release prepared

Version:
v{next_version}

Files updated:
- CHANGELOG.md
- {release_notes_file}
- {version manifest file, e.g. package.json / pyproject.toml / Cargo.toml} {if changed}
- README.md {if changed}

Git:
- Release commit: chore(release): v{next_version}
- Tag: v{next_version}
- Pushed: ✅

GitHub Release:
{release_url or "Skipped via --no-gh-release"}

Verification:
- Tasks complete: ✅
- Validation passed: ✅
- Changelog updated: ✅
- Release notes updated: ✅
- Version metadata updated: {✅|n/a}
- Final checks: ✅

Next:
- Review release page
- Announce release if needed
```
      </template>
    </phase>

  </flow>

  <control>
    <priority>release correctness and traceability over speed</priority>
    <failure>if release evidence is insufficient, stop and request explicit version/release decision</failure>

    <version_rules>
      <rule>Use explicit bump argument when provided.</rule>
      <rule>Use SemVer auto-bump only when PRD/spec impact is clear.</rule>
      <rule>Do not guess major/minor/patch if impact is ambiguous.</rule>
      <rule>Do not create duplicate tag.</rule>
    </version_rules>

    <changelog_rules>
      <rule>Changelog entries are human-readable summaries, not raw commits.</rule>
      <rule>Create CHANGELOG.md if missing.</rule>
      <rule>Preserve existing changelog style if present.</rule>
      <rule>Use Keep a Changelog categories when no style exists.</rule>
      <rule>Include compatibility and security notes when relevant.</rule>
    </changelog_rules>

    <release_rules>
      <rule>Never release with dirty working tree before metadata generation.</rule>
      <rule>Never release with unrelated files staged.</rule>
      <rule>Never use git add . or git add -A.</rule>
      <rule>Do not run gh release create before pushing tag.</rule>
      <rule>Use --notes-file for curated release notes.</rule>
      <rule>Use --fail-on-no-commits to avoid duplicate releases when applicable.</rule>
      <rule>mcp__semble is mandatory for all release code research — call mcp__semble__search before mcp__octocode__search or mcp__serena__find_symbol. Most token-efficient path.</rule>
    </release_rules>
  </control>

  <final>
    ## ✅ Release Manager Complete

    Release artifacts generated, committed, tagged, and published if GitHub Release was enabled.
  </final>

</command>