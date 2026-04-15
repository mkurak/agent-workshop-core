# Automatic Version Check Rule

## When This Runs

On **every prompt** — silently, before processing the user's message.

## What It Does

1. **Check global repos** (core, skills):
   - For each directory in `~/.claude/repos/mkurak/` that has a `.git/`:
     - Run `git fetch origin main --quiet`
     - Compare local HEAD with origin/main
     - If behind → `git pull --quiet`
     - Log: "🔄 {name} updated (v{old}→v{new})" — ONE line, no interruption

2. **Check project team repos:**
   - Read `.claude/.team-installs.json` (if exists)
   - For each installed team, check its `~/.claude/repos/mkurak/{name}/` source
   - Same fetch/compare/pull as above
   - If new files were added → create new symlinks in `.claude/`
   - If files were removed → clean broken symlinks

3. **Report:**
   - If updates happened: show a BRIEF one-liner per update (e.g., "🔄 software-team v1.0.0→v1.1.0")
   - If no updates: say NOTHING — completely silent
   - Never ask for confirmation — just update
   - Never interrupt the user's flow

## Implementation

```bash
# Silent version check (runs in background)
for team_dir in ~/.claude/repos/mkurak/*/; do
  [ -d "${team_dir}/.git" ] || continue
  
  cd "$team_dir"
  git fetch origin main --quiet 2>/dev/null
  
  LOCAL=$(git rev-parse HEAD 2>/dev/null)
  REMOTE=$(git rev-parse origin/main 2>/dev/null)
  
  if [ "$LOCAL" != "$REMOTE" ] && [ -n "$REMOTE" ]; then
    OLD_VERSION=$(jq -r '.version // "unknown"' team.json 2>/dev/null)
    git pull --quiet 2>/dev/null
    NEW_VERSION=$(jq -r '.version // "unknown"' team.json 2>/dev/null)
    echo "🔄 $(basename "$team_dir") updated (v${OLD_VERSION}→v${NEW_VERSION})"
    
    # Refresh symlinks for new files
    # (existing symlinks already point to the right place — only new files need linking)
  fi
done
```

## Important Rules

1. **NEVER ask for confirmation.** Auto-update, auto-pull, auto-refresh.
2. **NEVER block the user's prompt.** Version check is fast (git fetch = ~1s).
3. **Silent when no updates.** Don't say "everything is up to date" — just proceed.
4. **Brief when updates happen.** One line per updated repo, nothing more.
5. **Network failure = skip silently.** If git fetch fails (offline, timeout), ignore and continue.
6. **This runs on EVERY prompt.** Not once per session — every single prompt. Because another session might push changes between prompts.
