---
name: pr-create
version: 1.0.0
description: |
  Create a GitHub PR with Linear issue linking, structured title,
  product-focused body, and automated screenshot proof.
  Reads .context/decisions.md for context recovery.
allowed-tools:
  # Core file operations
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  # User interaction
  - AskUserQuestion
  - mcp__conductor__AskUserQuestion
  # Linear integration
  - mcp__claude_ai_Linear__list_issues
  - mcp__claude_ai_Linear__get_issue
  # Future: browse skill for screenshots (v2)
  # Future: S3 upload for screenshot hosting (v2)
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->

## Preamble (run first)

```bash
_UPD=$(~/.claude/skills/gstack/bin/gstack-update-check 2>/dev/null || .claude/skills/gstack/bin/gstack-update-check 2>/dev/null || true)
[ -n "$_UPD" ] && echo "$_UPD" || true
mkdir -p ~/.gstack/sessions
touch ~/.gstack/sessions/"$PPID"
_SESSIONS=$(find ~/.gstack/sessions -mmin -120 -type f 2>/dev/null | wc -l | tr -d ' ')
find ~/.gstack/sessions -mmin +120 -type f -delete 2>/dev/null || true
_CONTRIB=$(~/.claude/skills/gstack/bin/gstack-config get gstack_contributor 2>/dev/null || true)
_BRANCH=$(git branch --show-current 2>/dev/null || echo "unknown")
echo "BRANCH: $_BRANCH"
```

If output shows `UPGRADE_AVAILABLE <old> <new>`: read `~/.claude/skills/gstack/gstack-upgrade/SKILL.md` and follow the "Inline upgrade flow" (auto-upgrade if configured, otherwise AskUserQuestion with 4 options, write snooze state if declined). If `JUST_UPGRADED <from> <to>`: tell user "Running gstack v{to} (just updated!)" and continue.

## AskUserQuestion Format

**ALWAYS follow this structure for every AskUserQuestion call:**
1. **Re-ground:** State the project, the current branch (use the `_BRANCH` value printed by the preamble — NOT any branch from conversation history or gitStatus), and the current plan/task. (1-2 sentences)
2. **Simplify:** Explain the problem in plain English a smart 16-year-old could follow. No raw function names, no internal jargon, no implementation details. Use concrete examples and analogies. Say what it DOES, not what it's called.
3. **Recommend:** `RECOMMENDATION: Choose [X] because [one-line reason]`
4. **Options:** Lettered options: `A) ... B) ... C) ...`

Assume the user hasn't looked at this window in 20 minutes and doesn't have the code open. If you'd need to read the source to understand your own explanation, it's too complex.

Per-skill instructions may add additional formatting rules on top of this baseline.

## Contributor Mode

If `_CONTRIB` is `true`: you are in **contributor mode**. You're a gstack user who also helps make it better.

**At the end of each major workflow step** (not after every single command), reflect on the gstack tooling you used. Rate your experience 0 to 10. If it wasn't a 10, think about why. If there is an obvious, actionable bug OR an insightful, interesting thing that could have been done better by gstack code or skill markdown — file a field report. Maybe our contributor will help make us better!

**Calibration — this is the bar:** For example, `$B js "await fetch(...)"` used to fail with `SyntaxError: await is only valid in async functions` because gstack didn't wrap expressions in async context. Small, but the input was reasonable and gstack should have handled it — that's the kind of thing worth filing. Things less consequential than this, ignore.

**NOT worth filing:** user's app bugs, network errors to user's URL, auth failures on user's site, user's own JS logic bugs.

**To file:** write `~/.gstack/contributor-logs/{slug}.md` with **all sections below** (do not truncate — include every section through the Date/Version footer):

```
# {Title}

Hey gstack team — ran into this while using /{skill-name}:

**What I was trying to do:** {what the user/agent was attempting}
**What happened instead:** {what actually happened}
**My rating:** {0-10} — {one sentence on why it wasn't a 10}

## Steps to reproduce
1. {step}

## Raw output
```
{paste the actual error or unexpected output here}
```

## What would make this a 10
{one sentence: what gstack should have done differently}

**Date:** {YYYY-MM-DD} | **Version:** {gstack version} | **Skill:** /{skill}
```

Slug: lowercase, hyphens, max 60 chars (e.g. `browse-js-no-await`). Skip if file already exists. Max 3 reports per session. File inline and continue — don't stop the workflow. Tell user: "Filed gstack field report: {title}"

## Step 0: Detect base branch

Determine which branch this PR targets. Use the result as "the base branch" in all subsequent steps.

1. Check if a PR already exists for this branch:
   `gh pr view --json baseRefName -q .baseRefName`
   If this succeeds, use the printed branch name as the base branch.

2. If no PR exists (command fails), detect the repo's default branch:
   `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`

3. If both commands fail, fall back to `main`.

Print the detected base branch name. In every subsequent `git diff`, `git log`,
`git fetch`, `git merge`, and `gh pr create` command, substitute the detected
branch name wherever the instructions say "the base branch."

---

# PR Create: Product-Focused Pull Request

You are running the `/pr-create` workflow. Create a GitHub PR with a Linear issue link, structured title, and product-focused body.

---

## Step 1: Gather Context

Run these commands to understand what is being shipped:

```bash
git branch --show-current
git log <base>..HEAD --oneline
git diff <base>...HEAD --stat
```

Then read:
1. `.context/decisions.md` — if it exists, this is the primary source for Problem, Solution, Risks, and Assumptions sections.
2. Project `CLAUDE.md` files — for any Linear ticket references or project context.
3. **Test plan artifact** — check if `/plan-eng-review` wrote a test plan:
```bash
SLUG=$(git remote get-url origin 2>/dev/null | sed 's|.*[:/]\([^/]*/[^/]*\)\.git$|\1|;s|.*[:/]\([^/]*/[^/]*\)$|\1|' | tr '/' '-')
BRANCH=$(git rev-parse --abbrev-ref HEAD)
ls -t ~/.gstack/projects/$SLUG/*-$BRANCH-test-plan-*.md 2>/dev/null | head -1
```
If a test plan file exists, read it. Use its "Affected Pages/Routes", "Key Interactions to Verify", "Edge Cases", and "Critical Paths" sections to populate the "How to Verify" section of the PR body.

---

## Step 2: Linear Issue Discovery

Find the related Linear issue. Follow this priority order — stop at first match:

1. **Check `.context/decisions.md`** for a `TEN-\d+` pattern. If found, use that ticket.
2. **Check CLAUDE.md files** for `TEN-\d+` references.
3. **Search Linear API** via `mcp__claude_ai_Linear__list_issues`:
   - Search by keywords extracted from the branch name (split on `/`, `-`, `_`) and the first commit message.
   - Also filter by assignee "me" and states "started" or "in progress".
4. **Confirm with user:**
   - If 0 matches or >3 matches: use `mcp__conductor__AskUserQuestion` with any matches found + "Enter ticket manually" + "No ticket (N/A)" as options.
   - If 1-3 matches: use `mcp__conductor__AskUserQuestion` presenting the matches with a recommendation of the best match based on title similarity to the branch name.
5. Once identified, call `mcp__claude_ai_Linear__get_issue` to get the full issue details (title, description, URL).

---

## Step 3: PR Title

Format: `TEN-123: <type>(<scope>): <summary>`

Rules:
- **type**: Infer from the diff. `feat` for new features, `fix` for bug fixes, `refactor` for structural changes, `test` for test-only, `docs` for documentation, `chore` for config/build/deps.
- **scope**: Infer from changed files. Map to domain areas: `screening`, `auth`, `payments`, `employer-portal`, `contractor-portal`, `api`, `infra`, etc.
- **summary**: Imperative mood, ≤50 chars (excluding the `TEN-###: ` prefix).
- If no Linear ticket: omit the `TEN-123: ` prefix entirely.

---

## Step 4: Screenshot Proof (v2 — placeholder)

This step will be implemented in a future version using the browse skill + S3 upload.
Skip this step. Do not block PR creation.

---

## Step 5: PR Body

**If `.context/decisions.md` does NOT exist:** Before generating the body, use `mcp__conductor__AskUserQuestion` to gather the information needed for sections that require deeper context. Ask in batches:
- Batch 1: "What problem does this solve and why are we doing this?" + "Were there alternative approaches considered?"
- Batch 2: "What are the risks?" + "What assumptions were made?"

Do NOT silently infer these sections — prompt the user explicitly.

**If `.context/decisions.md` exists:** populate all sections directly from it. Use the Linear issue description to enrich the Problem/Why sections.

Generate the PR body using this exact template:

````markdown
## Problem
**Linear:** [TEN-123: title](url)

{What problem are we solving? 3 lines max. Non-technical. Product/design language only.}

### Why are we doing this?
{Thorough explanation of what we want to achieve with this PR. Not just "what's broken" but the thought process behind wanting to implement this change. What is the desired end state? What user/business outcome are we targeting? What triggered this work — a user complaint, a metric, a strategic decision? This section should give a reviewer full context on the motivation without needing to read the code.}

## Impact
- **Users:** {what changed from their perspective, or "No change"}
- **Admins/internal:** {what changed from admin perspective, or "No change"}
- **System:** {background services, data model, infra, or "No change"}

## Solution Analysis

### Alternatives Considered

**Option A: {name}**
{Description of the approach — what it involves, how it would work, what it touches.}
- **Pros:** {bullet list}
- **Cons:** {bullet list}

**Option B: {name}**
{Description of the approach — what it involves, how it would work, what it touches.}
- **Pros:** {bullet list}
- **Cons:** {bullet list}

_(If no `.context/decisions.md` and user confirms no alternatives: "Straightforward change — no alternatives analysis needed.")_

### Decision
{Which option was chosen and why. Address the cons of the chosen solution — how are they mitigated? Address the pros of the rejected solutions — why were they not enough to justify choosing them? This should read as a thorough reasoning, not just a label.}

## How to Verify

### Prerequisites
{What state the system needs to be in. What data must exist. What services must be running. What user accounts are needed. Be specific — if a database record needs specific field values, say which ones.}

### Data/State Setup
{Runnable queries or commands that set up the required state for testing. The reviewer should be able to copy-paste these directly.}

```bash
# Example: create test data
docker exec tenx-mongodb mongosh --quiet --eval '...'
```

<details>
<summary>Step-by-step verification</summary>

{If a test plan artifact was found in Step 1, use its Affected Pages/Routes and Key Interactions sections to generate these steps. Include the edge cases from the test plan as additional verification steps.}

1. Go to `{url}`
2. {action}
3. **Expect:** {result}

</details>

### Critical Paths
{If a test plan artifact was found in Step 1, list the critical end-to-end flows from it here. Otherwise infer from the diff.}

- {end-to-end flow that must work}

## Proof
_Screenshots will be added in a future version (browse skill + S3 upload)._

## Risks
{For each risk: describe the risk, its likelihood, its blast radius, and how it is mitigated. Be thorough — a reviewer should understand exactly what could go wrong and what safeguards exist.}

- **{Risk name}:** {What could happen. How likely is it. What is the blast radius — who/what is affected. How is it mitigated or monitored.}

## Assumptions
{List every assumption made during implementation. For each: state the assumption, why it was made, and what breaks if it turns out to be wrong.}

- **{Assumption}:** {Why we assumed this. What breaks if wrong.}

---
🤖 Generated with [Claude Code](https://claude.com/claude-code)
````

---

## Step 6: Create or Update PR

1. Check if a PR already exists: `gh pr view --json url -q .url 2>/dev/null`
2. If a PR exists: update it with `gh pr edit --title "<title>" --body "$(cat <<'EOF' ... EOF)"`
3. If no PR exists: create with `gh pr create --base <base> --title "<title>" --body "$(cat <<'EOF' ... EOF)"`
4. **Output the PR URL** — this should be the final output the user sees.

---

## Important Rules

- **Never skip Linear discovery.** Always attempt to find the ticket.
- **Never silently infer Problem/Why/Risks/Assumptions.** If `.context/decisions.md` is missing, ask the user.
- **PR body is product-focused.** Do not describe code changes — describe what changed for users, admins, and the system.
- **Keep the Problem section to 3 lines max.** The "Why" section is where depth goes.
- **Verification must be copy-pasteable.** Include actual commands, URLs, and expected results.
