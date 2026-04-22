# 🧬 Agent Workshop Core

Core infrastructure for all agent teams. Provides the memory system, journal, learning pipeline, agent structure rules, and automatic version checking.

## What's Inside

| Type | File | Purpose |
|------|------|---------|
| Skill | `skills/save-learnings/` | Save what was learned in a session — project memory, team repo, or both |
| Rule | `rules/memory-system.md` | Memory & journal system rules (auto-loaded every session) |
| Rule | `rules/agent-structure.md` | Agent configuration rules (children pattern, blueprint pattern) |
| Rule | `rules/version-check.md` | Automatic version checking on every prompt |
| Rule | `rules/karpathy-guidelines.md` | Four coding principles (Think Before Coding, Simplicity First, Surgical Changes, Goal-Driven Execution) |
| Template | `templates/agent-memory.md` | Template for per-project agent memory files |
| Template | `templates/journal-entry.md` | Template for inter-agent journal entries |

## Installation

Core is installed automatically by `install.sh` (bootstrap). No manual installation needed.

## How It Works

### Two-Layer Memory

```
Team Repo (global knowledge)                    Project Memory (project-specific)
~/.claude/repos/agentteamland/{team}/agents/agent.md    .claude/agent-memory/{agent}-memory.md
├── Patterns that work everywhere                ├── What worked in THIS project
├── Anti-patterns to avoid                       ├── Project-specific discoveries
└── Updated via /save-learnings + auto push      └── Updated via /save-learnings (stays local)
```

### Session Lifecycle

```
Session Start:
  1. Auto version check (silent git fetch/pull if behind)
  2. Agent reads its definition (from project .claude/agents/)
  3. Agent reads project memory (.claude/agent-memory/)
  4. Agent reads recent journal entries (.claude/journal/)

Session Work:
  (normal work happens)

Session End:
  /save-learnings (MANDATORY — agent proactively checks)
  1. Analyzes what was learned
  2. Asks user: project-specific or global?
  3. Writes to project memory and/or team repo
  4. Auto git push if team repo updated
  5. Writes journal entry for other agents
```

### Automatic Version Check

On every prompt, the version-check rule silently:
1. Runs `git fetch` on all cached repos in `~/.claude/repos/agentteamland/`
2. If behind origin → auto `git pull`
3. Brief one-liner notification if updated (e.g., "🔄 software-team v1.0.0→v1.1.0")
4. Completely silent if no updates

No user confirmation needed. No interruption.

## License

MIT
