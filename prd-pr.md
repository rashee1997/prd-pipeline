---
description: "PRD Step 7/7 — Creates PR after tasks, validation, implementation, review, compile/type checks, and tests are complete."
argument-hint: "<target branch optional — defaults to repo default/main>"
allowed-tools: Bash, mcp__semble__search, mcp__semble__find_related
---

<command name="/prd-pr">

  <execution>
    <follow_structure>strict</follow_structure>
    <treat_tags_as_semantic>true</treat_tags_as_semantic>
    <do_not_skip_phases>true</do_not_skip_phases>
    <do_not_assume>true</do_not_assume>
  </execution>

  <system>
    <role>Senior engineer preparing a completed feature pull request</role>
    <principle>Verify → Summarize → Push → Open PR</principle>
    <mode>self-contained reviewer-ready PR description</mode>
    <rules>
      <item>Never open PR from main/master.</item>
      <item>Never open PR with incomplete tasks.</item>
      <item>Never open PR without clean /prd-review after latest implementation commit.</item>
      <item>Never open PR with type/compile errors or failing tests.</item>
      <item>PR description must explain what, why, how, verification, and risks.</item>
      <item>Breaking Changes section is mandatory; "None" is valid.</item>
    </rules>
  </system>

  <input>
    <target_branch>{$ARGUMENTS}</target_branch>
    <default_target>repo default branch, fallback main</default_target>
  </input>

  <flow>

    <phase id="1" name="preflight">
      <task>Confirm branch, task completion, validation, review, and final checks.</task>

      <step name="confirm-feature-branch">
```bash
git branch --show-current
```

        <require>
          Current branch must be `feat/{feature-slug}-impl`.
        </require>

        <if condition="branch is main/master/non-feature">
          <output>
            ⛔ Not on a feature branch.

            Switch first:
            `git checkout feat/{feature-slug}-impl`

            Then rerun:
            `/prd-pr`
          </output>
          <stop/>
        </if>
      </step>

      <step name="locate-prd-folder">
        <goal>Find PRD folder from branch slug or provided path.</goal>

```bash
BRANCH=$(git branch --show-current)
SLUG=$(echo "$BRANCH" | sed 's/^feat\///' | sed 's/-impl$//')
ls docs/prd/ | grep "$SLUG" || find . -name "index.md" -path "*/tasks/*" | head -5
```

        <record>{prd-folder}</record>
      </step>

      <step name="confirm-tasks-complete">
```bash
cat {prd-folder}/tasks/index.md
grep "\- \[ \]" {prd-folder}/tasks/index.md
```

        <if condition="unchecked tasks exist">
          <output>
            ⛔ Feature is not complete.

            Pending tasks:
            {unchecked task lines}

            Complete them first:
            `/prd-implement {prd-folder}/tasks/{next-task}.md`
          </output>
          <stop/>
        </if>
      </step>

      <step name="confirm-task-validation">
        <goal>Ensure generated tasks were validated before implementation.</goal>

```bash
cat {prd-folder}/tasks/validation.md
```

        <require>
          validation.md exists and status is VALIDATED.
        </require>

        <if condition="validation missing or BLOCKED">
          <output>
            ⛔ Task validation gate not passed.

            Run:
            `/prd-validate {prd-folder}/tasks/index.md`

            Then fix any blockers and rerun `/prd-pr`.
          </output>
          <stop/>
        </if>
      </step>

      <step name="confirm-review-clean">
        <goal>Ensure /prd-review passed after latest implementation commit.</goal>

        <expected>
          Latest review result reports zero findings OR only explicitly accepted MEDIUM/LOW deferrals.
        </expected>

        <check>
          Read latest review output/report if saved in PRD folder.
          If no review artifact exists, require rerun.
        </check>

        <if condition="no clean review result">
          <output>
            ⛔ Review gate not passed.

            Run:
            `/prd-review {prd-folder}`

            Only rerun `/prd-pr` after:
            - zero findings, OR
            - explicit user acceptance for deferred MEDIUM/LOW findings.

            CRITICAL and HIGH findings must be fixed.
          </output>
          <stop/>
        </if>
      </step>

      <step name="final-verification">
        <task>Read a completed task file to get the compile/type-check and test commands baked into its acceptance criteria. Run those commands to verify everything still passes before creating the PR.</task>
```bash
{compile/type-check command from a completed task file's acceptance criteria} 2>&1 | head -30
{test command from a completed task file's acceptance criteria} 2>&1 | tail -20
```

        <if condition="compile/type-check or tests fail">
          <output>
            ⛔ Final verification failed.

            Fix failures before opening PR.
          </output>
          <stop/>
        </if>
      </step>
    </phase>

    <phase id="2" name="read-prd-documents">
      <task>Extract only what is needed for the PR description.</task>

      <files>
        {prd-folder}/discovery.md
        {prd-folder}/prd.md
        {prd-folder}/spec.md
        {prd-folder}/plan.md
        {prd-folder}/tasks/index.md
        {prd-folder}/tasks/validation.md
      </files>

      <extract-from-prd>
        <item>feature frontmatter → PR title base</item>
        <item>overview/problem → Problem</item>
        <item>overview/solution → Summary and Solution</item>
        <item>success_metrics → Success Metrics</item>
        <item>scope/in → What's Included</item>
        <item>scope/out → What's NOT Included</item>
        <item>scope/deferred → Deferred</item>
        <item>background/backward_compat → Breaking Changes</item>
        <item>user_stories → Who This Affects</item>
        <item>non_functional_requirements → notable NFRs</item>
      </extract-from-prd>

      <extract-from-spec>
        <item>architecture/decisions → Key Decisions</item>
        <item>schema/new_models + existing_changes → Schema Changes</item>
        <item>api routes → API Changes</item>
        <item>components → UI Changes</item>
        <item>security → Security Notes</item>
        <item>backward_compat_layer or frozen_contracts → Breaking Changes</item>
      </extract-from-spec>

      <extract-from-plan>
        <item>summary/method → Method</item>
        <item>summary/total_steps → total tasks</item>
        <item>definition_of_done → checklist</item>
      </extract-from-plan>

      <extract-from-index>
        <item>Progress line</item>
        <item>all completed task lines</item>
        <item>Plan link</item>
      </extract-from-index>
    </phase>

    <phase id="3" name="determine-target">
      <task>Resolve target branch.</task>

      <logic>
        <rule>If user supplied target branch, use it.</rule>
        <rule>Else detect remote HEAD branch.</rule>
        <rule>Fallback to main.</rule>
      </logic>

```bash
TARGET="${ARGUMENTS:-}"
if [ -z "$TARGET" ]; then
  TARGET=$(git remote show origin 2>/dev/null | grep "HEAD branch" | awk '{print $NF}')
  [ -z "$TARGET" ] && TARGET="main"
fi
echo "Target branch: $TARGET"
git branch -r | grep "origin/$TARGET"
```

      <if condition="target branch missing">
        <output>
          ⛔ Target branch not found: {TARGET}

          Ask user which branch to target.
        </output>
        <stop/>
      </if>
    </phase>

    <phase id="4" name="push-branch">
      <task>Push feature branch to origin.</task>

```bash
BRANCH=$(git branch --show-current)
git ls-remote --heads origin "$BRANCH"
git push -u origin "$BRANCH" || git push origin "$BRANCH"
```
    </phase>

    <phase id="5" name="build-pr-content">
      <task>Create PR title and self-contained PR body.</task>

      <title-rule>
        feat({feature-slug}): {imperative short description}
      </title-rule>

      <title-constraints>
        <item>≤ 72 characters</item>
        <item>imperative mood</item>
        <item>scope matches feature slug</item>
      </title-constraints>

      <body-template>
```md
## Summary

{2-3 sentences from PRD solution: what this implements, for whom, and outcome}

---

## Problem

{from PRD problem}

---

## Solution

{from PRD solution}

---

## Who This Affects

{condensed user stories}

---

## What's Included

{numbered list from PRD scope/in}

## What's NOT Included

{numbered list from PRD scope/out}

## Deferred

{items from scope/deferred, omit if empty}

---

## Key Implementation Decisions

| Decision | Choice | Rationale |
|---|---|---|
| {decision} | {choice} | {rationale} |

---

## Schema Changes

<!-- omit entirely if no schema changes -->

**New models:**
- ...

**Modified models:**
- ...

**Migration command:**
`{command}`

---

## API Changes

<!-- omit entirely if no API changes -->

| Method | Path | Purpose | Auth |
|---|---|---|---|
| GET | /api/... | ... | required/public |

---

## UI Changes

<!-- omit entirely if no UI changes -->

- `{Component}` — {purpose}

---

## Security Notes

- ...

---

## Breaking Changes

{If none: "None — this is a fully additive change."}

---

## Tasks Implemented

{Progress line}

- ✅ TASK-0-01 — {task name} · {effort}
- ✅ TASK-1-01 — {task name} · {effort}

---

## Validation & Review

- Task validation: `{prd-folder}/tasks/validation.md`
- Code review: `/prd-review {prd-folder}` passed with zero blocking findings
- CRITICAL/HIGH findings: zero
- MEDIUM/LOW deferrals: {none | explicitly accepted: list}

---

## Test Coverage

**Method:** {Standard | TDD — Red → Green → Refactor}

How to verify:

```bash
{test command from a completed task file's acceptance criteria}
{compile/type-check command from a completed task file's acceptance criteria}
```

---

## How to Test This Feature

{step-by-step flow from PRD ux_flow/primary_flow}

**Edge cases to check:**
- ...

---

## Success Metrics

{from PRD success_metrics}

---

## PRD Documents

| Document | Path |
|---|---|
| Discovery | `{prd-folder}/discovery.md` |
| PRD | `{prd-folder}/prd.md` |
| Spec | `{prd-folder}/spec.md` |
| Plan | `{prd-folder}/plan.md` |
| Tasks | `{prd-folder}/tasks/index.md` |
| Validation | `{prd-folder}/tasks/validation.md` |

---

## Checklist

- [x] `/prd-validate` passed
- [x] `/prd-review` passed with zero blocking findings
- [x] Compile/type check passes — zero errors
- [x] All tests pass
- [x] All tasks marked complete in `tasks/index.md`
- [x] No unsafe type suppressions or unresolved TODO comments
- [x] Inputs validated where needed
- [x] Auth checked before logic on protected routes
- [x] Breaking changes documented above or confirmed none
- [x] PRD documents linked above

🤖 Generated via `/prd-pr`
```
      </body-template>
    </phase>

    <phase id="6" name="create-pr">
      <task>Create the pull request with GitHub CLI.</task>

```bash
gh pr create \
  --base "{TARGET}" \
  --head "{BRANCH}" \
  --title "{PR_TITLE}" \
  --body "$(cat <<'EOF'
{PR_BODY}
EOF
)"
```

      <capture>PR URL</capture>
    </phase>

    <phase id="7" name="output">
      <template>
```md
✅ Pull request created

URL:    {pr url}
Title:  {pr title}
Base:   {TARGET} ← {BRANCH}
Tasks:  {N}/{N} complete

Verified:
- `/prd-validate` passed
- `/prd-review` clean
- `{compile/type-check command}` passed
- `{test command}` passed

Next steps:
- Assign reviewers: `gh pr edit {number} --add-reviewer {username}`
- Request review: `gh pr ready {number}`
- Monitor CI: `gh pr checks {number} --watch`
```
      </template>
    </phase>

  </flow>

  <control>
    <priority>safe PR creation over speed</priority>
    <failure>if any gate fails, stop before gh pr create</failure>

    <hard-rules>
      <rule>Never open PR from main/master.</rule>
      <rule>Never open PR with incomplete tasks.</rule>
      <rule>Never open PR if /prd-validate is missing or blocked.</rule>
      <rule>Never open PR unless /prd-review is clean after latest implementation commit.</rule>
      <rule>CRITICAL and HIGH review findings must be fixed.</rule>
      <rule>MEDIUM and LOW findings may be deferred only with explicit user acceptance.</rule>
      <rule>Never open PR with type or compile errors.</rule>
      <rule>Never open PR with failing tests.</rule>
      <rule>Never push directly to target branch.</rule>
      <rule>PR title must be ≤ 72 characters and imperative.</rule>
      <rule>Breaking Changes section must be explicit.</rule>
      <rule>All PRD document paths must be relative.</rule>
    </hard-rules>
  </control>

</command>