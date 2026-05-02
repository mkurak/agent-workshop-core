---
name: create-pr
description: "Move dirty changes to an auto-named branch, run save-learnings + AI review chain (generic baseline + team specialists), open PR. Optional --auto-merge enables GitHub auto-merge with polling + auto-fix loop. End-of-work returns the user to the target branch. Designed to satisfy team-repo-maintenance + branch-protection discipline."
argument-hint: "[--auto-merge] [--no-review] [--no-auto-fix] [--no-learning] [--timeout N]"
---

# /create-pr Skill

## Purpose

Take the working-tree changes (uncommitted, or recently committed to the default branch), derive an appropriate branch name + commit message + PR title from the diff, run `/save-learnings` so wisdom rides along in the same PR, run an AI review chain (generic baseline + any team-declared specialists), commit + push, open a PR. Optionally enable GitHub auto-merge with a bounded polling + auto-fix loop. Always return the user to the target branch at end-of-work.

This skill is the deterministic "ship a piece of work" flow — it consumes the disciplines defined by `team-repo-maintenance`, `branch-hygiene`, `learning-capture`, `docs-sync`, and `karpathy-guidelines`, so the user does not have to re-derive them every PR.

## Flags

| Flag | Default | Effect |
|---|---|---|
| `--auto-merge` | OFF | Enable GitHub auto-merge (`gh pr merge --auto --merge`); poll + auto-fix until merged or terminal failure |
| `--no-review` | OFF (review on) | Skip the entire review chain (generic + every team reviewer) |
| `--no-auto-fix` | OFF (fix on) | During the polling loop, do not attempt to fix CI/merge failures; surface to the user instead |
| `--no-learning` | OFF (learning on) | Skip `/save-learnings` + doc-impact pipeline |
| `--timeout {min}` | 10 | Polling timeout in minutes; 1-minute interval, applies to both `--auto-merge` and manual-merge wait |

## Flow

The flow runs sequentially. Each step has a clear precondition and postcondition; if a precondition fails, surface the issue and stop instead of proceeding.

### Step 1 — Pre-checks

- Current directory is inside a git repo (`git rev-parse --git-dir`).
- The working tree has changes OR the current branch has unpushed commits ahead of its upstream. If neither, exit cleanly: "Nothing to do — working tree clean and branch up-to-date."
- Determine the repo's default branch: `gh repo view --json defaultBranchRef -q .defaultBranchRef.name` (usually `main`, sometimes `master`). Treat this as the canonical destination.

### Step 2 — Determine target branch

The "target branch" is the branch the PR will merge into AND the branch the user returns to at end-of-work.

**Case A: User is on the default branch (`main`/`master`).**
- Target = default branch.
- If the default branch has local commits ahead of `origin/{default}` (rare — branch protection should prevent direct commits), proceed with reset + cherry-pick: create the feature branch carrying those commits, then reset the local default branch to `origin/{default}`. Do not ask — just handle silently. This is an anomaly recovery, not a routine path.

**Case B: User is on a non-default branch.**
- Use `AskUserQuestion` to ask the user where the PR should go. Provide three choices:
  1. **Upper branch** — auto-detect via `git rev-parse --abbrev-ref @{upstream}` if available, otherwise `git merge-base` against known branches. If detected, show the name in the option label.
  2. **Default branch** — `main` (or whatever the default is).
  3. **Other** — free-text option allowing the user to type a custom target branch name.
- Selection becomes the target branch.

### Step 3 — Generate branch name and commit message

Analyze the staged + unstaged + untracked changes:
- `git diff --stat HEAD` for size signal
- `git diff --name-only HEAD` for file list
- `git status -s` for untracked files
- `git log {target}..HEAD --oneline` if the user has local commits

Determine:
- **Type** (one of `feat`, `fix`, `docs`, `chore`, `refactor`, `test`, `perf`, `style`):
  - New file under `skills/`, `agents/`, `rules/` → `feat`
  - Bug-fix language in commit messages or diff → `fix`
  - Only `*.md` / `docs/` / `README*` changes → `docs`
  - Routine sync, version bump, generated files → `chore`
  - No behavior change, only structural moves → `refactor`
  - Only test files → `test`
- **Scope** (one of: skill name, rule name, agent name, CLI command, repo area). Pick the most specific scope that covers the change.
- **Slug** — kebab-case, ≤ 50 characters, ASCII only. Derive from the most prominent change.

Outputs:
- **Branch name:** `{type}/{slug}` (e.g., `feat/create-pr-skill`, `fix/winget-403`, `docs/translate-trk-en`).
- **Commit subject:** `{type}({scope}): {one-line summary}` — under 70 chars.
- **Commit body:** 2-4 bullet points describing the change. Last line is a "Discovered via" context if invoking from a team-repo context (per [team-repo-maintenance](../../rules/team-repo-maintenance.md) §3).

Do not ask the user to confirm names — generate, proceed.

### Step 4 — Save learnings (unless `--no-learning`)

Invoke `/save-learnings` in manual mode (no `--from-markers` flag — analyzes the live conversation). This:
- Writes wiki / journal / agent children / skill learnings updates (project-local).
- Updates team-repo files for maintainers with push access; users without push see a graceful permission-denied and stay local.
- Prepares `doc-impact` drafts for any `<!-- learning -->` markers with `doc-impact: readme/docs/breaking`.

For each prepared draft, present it inline to the user for accept/reject/edit:

```
📝 Doc draft for README.md:
<diff>
Accept? (y/n/edit)
```

Accepted drafts are staged. Rejected drafts are discarded.

If `/save-learnings` is not installed (defensive), skip with a one-line notice: "save-learnings not available; continuing without learning capture." Do not fail the skill.

### Step 5 — Review chain (unless `--no-review`)

Two layers, executed sequentially. **Generic always runs first; team reviewers append.**

#### Layer 5a — Generic reviewer (always)

Invoke `Task` with `subagent_type: general-purpose` and the following review prompt. The subagent has fresh context, so its review is not biased by the same model that wrote the diff.

```
Review the staged diff for this PR. Apply the four Karpathy guidelines:

1. **Think Before Coding** — Are the assumptions explicit? Is anything ambiguous that should have been clarified?
2. **Simplicity First** — Any over-engineering? Any abstractions for single-use code? Any features beyond what was asked?
3. **Surgical Changes** — Any drive-by edits unrelated to the goal? Any orphaned imports / variables / functions left behind?
4. **Goal-Driven Execution** — Does the change verify against its goal? Are there clear success criteria?

Also check general code quality:
- **Naming clarity** — variables, functions, modules
- **Scope creep** — does this PR do one thing, or multiple?
- **Security smells** — logging secrets, eval / exec on untrusted input, command injection, SQL injection, XSS, hardcoded credentials
- **Dead code** — commented-out blocks, unused params, debug print statements
- **Test coverage** — if tests exist in this repo, does this change need a test?

Diff context:
{git diff --staged}

Report in this format:
- 🔴 **Issues** (must fix before merge)
- 🟡 **Concerns** (should consider)
- 🟢 **Looks good** (positive observations, optional)

Keep it terse. Skip nits unless they're security or correctness related.
```

The generic reviewer's output is captured.

#### Layer 5b — Team reviewers (per installed team)

For each installed team in `~/.claude/repos/agentteamland/{team}/`:
- Read `team.json`. Look for `capabilities.review`.
- If declared and the value is a string referencing an entry in the team's `agents` array, invoke `Task` with `subagent_type` matching the agent name (e.g., `subagent_type: "code-reviewer"` if the team declares `capabilities.review: "code-reviewer"`).
- The team agent receives the same diff and produces a domain-specific review.
- Each team reviewer's output is appended to the report.

If a team does not declare `capabilities.review`, skip silently — there is no fallback per team. The generic reviewer (5a) is the platform-wide baseline.

#### Present the consolidated report

Show the user the generic report + each team report. Ask: continue / abort / edit (open editor on the diff to amend).

### Step 6 — Commit + push

Stage everything (the original diff + any learning artifacts + any accepted doc drafts):

```bash
# If we're not yet on the feature branch:
git checkout -b {branch-name}

# Stage and commit:
git add -A
git commit -m "{commit-subject}

{commit-body}

{discovered-via-line if applicable}"

git push -u origin {branch-name}
```

If the source branch had local commits (Case A anomaly recovery), the cherry-pick + reset happens here too.

### Step 7 — Open PR

```bash
gh pr create \
  --base {target-branch} \
  --title "{commit-subject}" \
  --body "$(cat <<'EOF'
## Summary
<2-4 bullets describing the change>

## Discovered via
<context: which session / project / observation revealed the need>

## Version bump
<X.Y.Z → X.Y.Z+1 if a team.json version was bumped, otherwise: "N/A">

## Test plan
- [ ] <verification step>
- [ ] <regression check>
EOF
)"
```

Do not pass `--assignee` or `--reviewer` — see [team-repo-maintenance](../../rules/team-repo-maintenance.md) §4.

Capture the PR number and URL.

### Step 8 — `--auto-merge` polling (only if flag set)

If `--auto-merge` was not passed, skip directly to Step 10 (manual-merge polling).

Enable auto-merge:

```bash
gh pr merge {N} --auto --merge
```

This is the **only allowed merge invocation**. It does not merge immediately — GitHub waits for required checks to pass and then merges automatically. Branch protection's check gate is preserved. See [team-repo-maintenance](../../rules/team-repo-maintenance.md) for the exception note that documents this.

### Step 9 — Polling + auto-fix loop (if `--auto-merge`)

Poll the PR state at 1-minute intervals, up to `{timeout}` attempts (default 10).

```bash
fix_attempts=0
for i in $(seq 1 {timeout}); do
  state=$(gh pr view {N} --json state,mergeStateStatus -q '"\(.state) \(.mergeStateStatus)"')

  case "$state" in
    "MERGED "*)
      # Success — proceed to end-of-work
      break
      ;;
    "CLOSED "*)
      # User closed without merge — exit cleanly, no end-of-work
      report "PR closed without merge by user."
      exit 0
      ;;
    *" CLEAN"|*" HAS_HOOKS")
      # Healthy state, just waiting for checks — continue polling
      sleep 60
      ;;
    *" BLOCKED"|*" UNSTABLE"|*" DIRTY"|*" BEHIND")
      # CI failure or merge conflict — examine and decide
      handle_failure
      ;;
    *)
      sleep 60
      ;;
  esac
done
```

#### `handle_failure` logic

Skip to "report and exit" if `--no-auto-fix` was passed.

Get the failure context:
- For merge conflicts (`DIRTY`, `BEHIND`): `git fetch origin` + inspect conflict markers after attempted merge
- For CI failures (`BLOCKED`, `UNSTABLE`): `gh run list --branch {branch-name} --limit 1 --json databaseId,conclusion` + `gh run view {id} --log-failed`

Classify the failure:

**In-scope (auto-fix attempted):**
- **Merge conflict** — fetch latest target, attempt 3-way merge, resolve by accepting both sides where mechanically obvious, otherwise abort
- **Lint / format failure** — run the project's formatter (detect `package.json scripts.lint`, `.prettierrc`, `gofmt`, `cargo fmt`, etc.); commit the diff
- **Trivial type error / missing import** — look for clearly-stated suggestions in compiler output, apply them; commit the diff

**Out-of-scope (notify and stop):**
- Real test failures (assertion errors, regressions in existing test suites)
- Non-trivial build errors (missing dependency, refactor needed)
- Infrastructure / CI config issues (GitHub Actions itself failing, secrets missing)
- Missing required reviews (human reviewers blocking)

If in-scope and `fix_attempts < 3`:
- Apply fix, `git commit -m "fix({scope}): auto-fix {category} during /create-pr polling"`, `git push`
- `fix_attempts++`
- Resume polling

If out-of-scope or `fix_attempts >= 3`:
- Capture the failure summary + actionable hint for the user
- Skip end-of-work
- Report URL + failure to user, exit

### Step 10 — Manual-merge polling (only if `--auto-merge` was NOT set)

If `--auto-merge` flag was not used, the skill still polls for merge — the user might merge manually within `{timeout}` minutes.

```bash
for i in $(seq 1 {timeout}); do
  state=$(gh pr view {N} --json state -q '.state')
  case "$state" in
    "MERGED") break ;;
    "CLOSED") report "PR closed without merge."; exit 0 ;;
    *) sleep 60 ;;
  esac
done

if [[ "$state" != "MERGED" ]]; then
  report "Polling timed out after {timeout} minutes — PR is still open. End-of-work skipped."
  report "PR URL: {url}"
  exit 0
fi
```

### Step 11 — End-of-work (universal)

Reached only when the PR was successfully merged (auto-merge OR manual-merge within timeout).

```bash
git checkout {target-branch}
git pull origin {target-branch}
```

This is the universal "return-to-clean-state" step that applies regardless of which path was taken in Step 2 or Step 8. The user ends the skill on the target branch, with the merged change incorporated, ready for the next task.

If pulling fails (network issue, merge conflict in target branch — rare), surface the error and leave the user where they are.

### Step 12 — Final report

A consolidated, single-block report:

```
✅ /create-pr complete
   Branch:      feat/create-pr-skill
   PR:          https://github.com/.../pull/N
   Review:      generic + 1 team reviewer (software-project-team)
                3 issues, 1 concern, all addressed
   Learnings:   /save-learnings ran — 2 wiki pages updated, 1 README draft accepted
   Auto-merge:  enabled, merged after 4 min (1 auto-fix: prettier formatting)
   End-of-work: returned to main, pulled latest
```

Adjust the report fields to match what actually happened. If something was skipped (`--no-X` flag), say so explicitly.

## Important constraints

1. **Never merge directly.** This skill uses `gh pr merge --auto --merge` (auto-merge enable) only when `--auto-merge` flag is passed. Direct merge (`gh pr merge --merge` or `--squash` or `--rebase`) is **always forbidden** — see [team-repo-maintenance](../../rules/team-repo-maintenance.md) "PR merge discipline." The user has explicitly authorized auto-merge by typing the flag; this is the exception note documented in the rule.

2. **Discovered via context.** When invoked from a shared/team repo, follow [team-repo-maintenance](../../rules/team-repo-maintenance.md) discipline: include "Discovered via" in PR body, bump version, conventional commit. The skill should detect "is this a shared repo?" by checking if the current directory is under `~/.claude/repos/agentteamland/` or matches a known shared repo pattern.

3. **Idempotent save-learnings.** Even if `/save-learnings` was run earlier in the same session, running it again here is safe — it appends, deduplicates, and processes only fresh content.

4. **Schema validation.** If the staged diff touches a `team.json`, run the validator before push: `~/.claude/repos/agentteamland/core/scripts/validate-team-json.sh path/to/team.json`. Per [branch-hygiene](../../rules/branch-hygiene.md), this is non-negotiable.

5. **Branch hygiene before start.** Before starting the flow, verify the local default branch is current with origin. If not, fast-forward (`git fetch && git pull --ff-only`) before deriving the new branch — otherwise you build on stale base. Per [branch-hygiene](../../rules/branch-hygiene.md).

6. **No silent partial failures.** If any step fails, the skill stops and reports — it does not silently continue. The user knows where they are after the skill ends.

## Related rules and skills

- [team-repo-maintenance](../../rules/team-repo-maintenance.md) — governance for shared repos (discipline this skill enforces)
- [branch-hygiene](../../rules/branch-hygiene.md) — drift-prevention checkpoints
- [learning-capture](../../rules/learning-capture.md) — inline marker protocol (consumed by `/save-learnings`)
- [docs-sync](../../rules/docs-sync.md) — proactive doc updates (consumed via doc-impact drafts)
- [karpathy-guidelines](../../rules/karpathy-guidelines.md) — review prompt's foundation
- [/save-learnings](../save-learnings/skill.md) — invoked at Step 4

## Future evolution (v2)

- **Domain-aware review routing.** Each team agent declares `domains: ["*.tsx", ...]` glob; skill matches the diff's file types and invokes only relevant agents instead of all team reviewers. Reduces review time for narrow PRs.
- **Parallel team review.** Run team reviewers concurrently instead of sequentially.
- **Auto-fix scope expansion.** Extend in-scope to test failures where the test was added in the same diff (Claude wrote the test wrong).

## Accumulated Learnings

(Auto-rebuilt by /save-learnings from learnings/*.md frontmatter. Do not edit by hand. Currently empty — populates as the skill is used and edge-case learnings accumulate.)
