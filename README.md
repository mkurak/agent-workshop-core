# 🧬 Agent Workshop Core

Core infrastructure for all agent teams. Provides the memory system, journal, and learning pipeline that makes agents smarter over time.

## What's Inside

| Type | File | Purpose |
|------|------|---------|
| Skill | `skills/save-learnings/` | Save what was learned in a session — project memory, team repo, or both |
| Rule | `rules/memory-system.md` | Memory & journal system rules (auto-loaded every session) |
| Template | `templates/agent-memory.md` | Template for per-project agent memory files |
| Template | `templates/journal-entry.md` | Template for inter-agent journal entries |

## Installation

```bash
/team install https://github.com/mkurak/agent-workshop-core.git
```

> Requires [Agent Team Manager](https://github.com/mkurak/agent-workshop-agent-team-manager-skill) to be installed first.

## How It Works

### Two-Layer Memory

```
Team Repo (global knowledge)          Project Memory (project-specific)
~/.claude/repos/mkurak/{team}/agents/agent.md   .claude/agent-memory/{agent}-memory.md
├── Patterns that work everywhere      ├── What worked in THIS project
├── Anti-patterns to avoid             ├── Project-specific discoveries
└── Updated via /save-learnings        └── Updated via /save-learnings
    + auto git push                        (stays local)
```

### Session Lifecycle

```
Session Start:
  1. Agent reads its own definition (from team repo via symlink)
  2. Agent reads project memory (.claude/agent-memory/)
  3. Agent reads recent journal entries (.claude/journal/)
  4. Agent reads project-specific rules (.claude/docs/coding-standards/)

Session Work:
  (normal work happens)

Session End:
  /save-learnings
  1. Analyzes what was learned in this session
  2. Asks user: project-specific or global?
  3. Writes to project memory and/or team repo
  4. Auto git push if team repo updated
  5. Writes journal entry for other agents
```

### Journal (Inter-Agent Communication)

Agents share knowledge through journal entries:

```
.claude/journal/
├── 2026-04-13_api-agent.md     ← "EF Include chain >3 kills performance"
├── 2026-04-13_flutter-agent.md ← "Riverpod AsyncValue pattern works well"
└── 2026-04-14_api-agent.md     ← "Redis pipeline batching 10x faster"
```

Each agent reads the journal at session start. Cross-cutting discoveries are shared automatically.

## Dependency

This is a **core dependency** — teams that use agents should install this first. Teams declare their dependency in `team.json`:

```json
{
  "name": "software-project-team",
  "dependencies": [
    "https://github.com/mkurak/agent-workshop-core.git"
  ]
}
```

When `/team install` sees dependencies, it installs them first.

## License

MIT
