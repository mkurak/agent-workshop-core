# 🧬 Agent Workshop Core

Core infrastructure for all agent teams. Provides the memory system, journal, wiki, learning pipeline (inline markers + hook-driven capture), agent structure rules, docs-sync discipline, and automatic version checking.

## What's Inside

| Type | File | Purpose |
|------|------|---------|
| Skill | `skills/save-learnings/` | Save what was learned — project memory, **wiki (mandatory)**, team repo, journal; prepares doc drafts for `doc-impact` markers |
| Skill | `skills/wiki/` | Living project knowledge base (init, ingest, query, lint) |
| Skill | `skills/create-code-diagram/` | Generate a full-project Mermaid class diagram |
| Rule | `rules/memory-system.md` | 4-layer knowledge model (memory, journal, wiki, docs) + agent startup routine |
| Rule | `rules/agent-structure.md` | Agent configuration rules (children pattern, blueprint pattern) |
| Rule | `rules/version-check.md` | Automatic version checking on every prompt (hook-driven) |
| Rule | `rules/learning-capture.md` | Inline `<!-- learning -->` marker protocol — cheap, greppable, hook-processed |
| Rule | `rules/docs-sync.md` | Proactive docs sync: README / doc-site updates in the same turn as user-facing changes |
| Rule | `rules/team-repo-maintenance.md` | Governance for team / global repo changes: semver bump + conventional commit + PR flow |
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
Session Start (SessionStart hook):
  1. atl update --silent-if-clean  (auto-update cached repos)
  2. Agent reads its definition (from project .claude/agents/)
  3. Agent reads relevant wiki pages (.claude/wiki/)
  4. Agent reads project memory (.claude/agent-memory/)
  5. Agent reads recent journal entries (.claude/journal/)

Session Work:
  (normal conversation)
  When a learning moment occurs → agent drops an inline <!-- learning --> marker
  in its response. If the change is user-facing → docs-sync rule prompts the
  agent to update README / doc site in the same turn, or mark doc-impact on
  the learning marker for session-end processing.

Session End / PreCompact (SessionEnd + PreCompact hooks):
  1. atl learning-capture --silent-if-empty  (scans transcript for markers)
     - 0 markers → silent exit (zero tokens)
     - 1+ markers → report injected into context for /save-learnings to process
  2. /save-learnings --from-markers
     - Updates agent-memory (append)
     - Updates wiki pages (MANDATORY; replace/update — current truth)
     - Writes journal entry
     - Prepares doc drafts for doc-impact markers (never auto-pushed)
     - Commits + pushes team repo changes if any
```

The mechanism is split between:
- **`rules/learning-capture.md`** — inline marker protocol (what you drop and when)
- **`rules/docs-sync.md`** — proactive docs discipline (paired with learning-capture via `doc-impact`)
- **`atl learning-capture`** (CLI) — greps the transcript for markers; runs at hook time
- **`/save-learnings --from-markers`** — processes marker bodies into wiki + memory + drafts

### Automatic Version Check

On every prompt, the version-check rule silently:
1. Runs `git fetch` on all cached repos in `~/.claude/repos/agentteamland/`
2. If behind origin → auto `git pull`
3. Brief one-liner notification if updated (e.g., "🔄 software-team v1.0.0→v1.1.0")
4. Completely silent if no updates

No user confirmation needed. No interruption.

## License

MIT
