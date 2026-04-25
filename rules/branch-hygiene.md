# Branch + working-tree hygiene

## Who runs this

**You (the agent).** This rule encodes three discipline checkpoints that prevent the recurring "drift" problem — local checkouts on stale merged branches, untrusted assumptions about working-tree state, validation skipped before push. Each was caught in production after the fact more than once. Each now has a fixed checkpoint here.

## When this fires

Three temporal triggers, each non-negotiable:

1. **Before starting any task in a shared repo** — verify branch + clean state
2. **Right after the user signals a PR merge** — switch to main + pull immediately
3. **Right before pushing a commit that touches a constrained file** — run the local validator

If any of these gets skipped, drift accumulates silently and the user has to surface it (which has happened, repeatedly — that's why this rule exists).

## 1. Pre-task verification

Before editing any file inside one of these locations:

- `~/.claude/repos/agentteamland/{team}/` (cached install paths used by `atl`)
- `<workspace>/repos/{team}/` (peer clones for maintainer work)
- `<workspace>/` itself

…verify three things in this order:

```bash
git branch --show-current        # → should be main (or master for forks)
git status --porcelain           # → should be empty
git rev-list --count HEAD..@{u}  # → should be 0 (or run git pull first)
```

If branch ≠ main: STOP. The branch is either stale-merged (PR merged, branch ought to be cleaned up) OR active work-in-progress that another session left behind. Surface to the user before editing.

If working tree is dirty: STOP. The uncommitted changes might be lost work from a prior session. Surface; ask the user what to do.

If behind on main: `git pull --ff-only` first, then proceed.

**Why so strict:** the alternative is what bit us on 2026-04-25 — six local repos sat on merged feature branches, no work was lost but `atl update`'s pull operated on the wrong branches and the user's "what's the latest?" question had wrong answers for each one. Cheap pre-task check beats expensive after-the-fact reconciliation.

## 2. Post-merge cleanup — immediate, automatic

The user signals a PR merge in many ways. The signal can be **direct**:

- `"Onayladım"`, `"merged"`, `"approved + merged"`, `"merge ettim"`, `"closed and merged"`

…or **indirect** — and the indirect cases are the ones that historically failed to trigger the reflex:

- `"PR kalmadı"` / `"onaylanacak bir şey yok"` — implies all PRs are merged
- `"hepsi merged"` / `"tüm PR'lar bitti"`
- A screenshot of an empty GitHub PR inbox
- A screenshot of branch state showing local commits ahead-of-main with "Create PR" button (= local is behind merged remote main)
- The user asking "neden burada hâlâ X branch görünüyor" / "Create PR butonu görünüyor" — the very fact they're asking means drift is visible to them

**Default rule when uncertain**: if the user mentions PR state in any form, run `/repo-status` (workspace) or the manual `gh pr list` checks (anywhere) BEFORE assuming. Uncertainty is a signal to verify, not to assume the previous state still holds.

Once the merge signal is detected (direct OR indirect), your **next action** in each affected repo is:

```bash
cd <affected-repo>
git checkout main
git pull --ff-only origin main
```

Do NOT wait for the next task. Do NOT defer cleanup "until later". Do NOT assume `atl update`'s background hook will catch it (it doesn't — it pulls whatever branch you're on, which after merge is typically the now-orphaned feature branch).

If multiple repos were affected by the merge train (e.g., a session shipped 3 PRs across 3 repos), do this for each one. Surface the result back to the user: "Switched 3 repos to main — all clean."

**What you do NOT do automatically:**

- Don't `git branch -d` the merged feature branch. Local branches with no upstream are harmless and the user owns the decision to prune.
- Don't switch branches when uncommitted changes are present (loss-risk; surface and ask first).
- Don't run pull on detached HEAD or in-progress rebase/merge states (surface; ask).

## 3. Pre-push validation — for files with machine-checkable constraints

Some files are validated against schemas, length limits, or other rules by the CI on push. Before pushing a commit that touches one of these files, run the local validator. Skipping this and discovering the failure on CI costs an extra round-trip per failure.

Known cases as of writing:

| Concern | Local validator | Why |
|---|---|---|
| `team.json` schema (any agentteamland repo) | `~/.claude/repos/agentteamland/core/scripts/validate-team-json.sh <path>` | Schema-validated by CI — has bitten 3× in production for description maxLength=200 |
| Personal-machine paths + user-private strings (any file in any agentteamland repo) | `~/.claude/repos/agentteamland/core/scripts/scan-personal-paths.sh` | Catches `/Users/<name>/`, `/home/<name>/`, `C:\Users\<name>\` paths; reads user-private strings (project names etc.) from `~/.claude/scan-personal-strings.conf` (NOT checked into any repo). Caught at least one absolute-path leak in production (workspace#4 first commit, fixed pre-merge after user spotted it in PR review) |

When you add a new file with machine-checkable constraints to any agentteamland repo, **also add a local validator script** to `core/scripts/` and update this table. Otherwise the constraint will get forgotten the same way `team.json`'s did.

The rule is "**validate-once-trust-never**" — every push touching a constrained file gets re-validated locally, no exceptions, no "this is just a small change."

### Recommended pre-push routine

Before any push to an agentteamland repo, run both validators in sequence:

```bash
# From inside the repo being pushed
~/.claude/repos/agentteamland/core/scripts/scan-personal-paths.sh
~/.claude/repos/agentteamland/core/scripts/validate-team-json.sh team.json   # if team.json was edited
```

Both are idempotent and fast (sub-second on typical staging areas). The cost of running them is trivial; the cost of skipping is a CI failure or — worse — a leaked path that lands in a public commit.

### User-private string config (the file that doesn't go in a repo)

`scan-personal-paths.sh` reads from `~/.claude/scan-personal-strings.conf` if it exists. The format is one pattern per line:

```
# Personal project names (kept out of public commits)
my-private-project
internal-codename

# Personal hostnames or scratch identifiers (regex prefix for ERE):
regex:my-scratch-\w+
```

This config file lives in the maintainer's home — **never** in a public repo, including `core` itself. The script in `core/scripts/` only carries universal OS-path patterns; the user-specific strings are read from this private config so the very mechanism for preventing leaks doesn't itself become a leak.

## Tool support — `/repo-status` skill (workspace-scoped)

The workspace's `.claude/skills/repo-status/skill.md` provides a one-command read-only report that surfaces all three checkpoints:

- Current branches across workspace + every peer repo + every cached install path
- Open PRs across the org
- Stale-merged-branch detection with one-line recovery commands

Use it:

- At session start in workspace, before deciding what to work on
- After a merge train of multiple PRs
- Anytime the user asks "ortam temiz mi" / "tüm repolar main'de mi" / "açık PR var mı"
- Anytime YOU (the agent) feel uncertain about state — uncertainty is a signal to verify, not to assume

Outside workspace (e.g., in a project that imports a team), the agent uses the manual git/gh checks from sections 1 and 2 directly. The /repo-status skill specifically covers the workspace's multi-repo topology.

## Anti-patterns

- ❌ "Probably on main" — proceeding without `git branch --show-current` confirmation
- ❌ Running `atl update` and assuming it caught all stale branches (it doesn't)
- ❌ Skipping the local validator because "this change is too small to need it"
- ❌ Auto-deleting local branches without user signal — harmless to leave; risky to delete
- ❌ Running `git checkout main` when working tree is dirty — silent data loss path

## History

This rule was created on 2026-04-25 after a session shipped three PRs (design-system-team@0.5.0, workspace state snapshot, core@1.4.0), all of which merged cleanly, but then six local checkouts (workspace + 5 peer/cached repos) remained on the now-stale feature branches. The user surfaced the drift with the question "tüm repolar main'de mi"; the agent had to clean up post-hoc. The trust gap was real: the agent had been claiming a state ("we're in good shape") without verifying it. This rule + the `/repo-status` skill close that gap by making verification the default, not the exception.

Companion rules:

- [team-repo-maintenance.md](team-repo-maintenance.md) — covers the commit/PR flow; this rule covers what happens BEFORE work starts and AFTER merge lands
- [version-check.md](version-check.md) — auto-update hook for cached repos; doesn't switch branches, only pulls
- [learning-capture.md](learning-capture.md) — the inline `<!-- learning -->` marker protocol that runs at session-end

If a maintainer reads this years from now wondering "is this still needed?" — the answer is yes if any of the three failure modes (stale-branch checkout, untrusted state assumption, skipped pre-push validation) is still possible. Drop it only when all three have been eliminated structurally (e.g., a SessionStart hook that hard-fails any repo not on main, a CI-gated branch-protection extension, etc.).
