# 🧬 AgentTeamLand Core

Core infrastructure for all agent teams. Provides the journal + wiki knowledge system, the auto-trigger learning-capture loop (inline markers + SessionStart hook + state-file-driven processing), the agent + skill structure rules, docs-sync discipline, and automatic version checking.

## What's Inside

| Type | File | Purpose |
|------|------|---------|
| Skill | `skills/save-learnings/` | Persist conversation learnings to journal + wiki + agent children + skill learnings; auto-rebuild agent.md / skill.md Knowledge Base sections from frontmatter; multi-transcript input via `--transcripts`; `AskUserQuestion` only for new structure creation |
| Skill | `skills/wiki/` | Living project knowledge base (init, ingest, query, lint) |
| Skill | `skills/create-code-diagram/` | Generate a full-project Mermaid class diagram |
| Skill | `skills/create-pr/` | Auto-named branch + save-learnings + AI review chain + PR; optional auto-merge |
| Rule | `rules/knowledge-system.md` | 2-layer knowledge model (journal, wiki) + agent startup routine. Renamed from `memory-system.md` in core@1.8.0 |
| Rule | `rules/agent-structure.md` | Agent + skill structure (children + learnings patterns; Knowledge Base + Accumulated Learnings auto-rebuild from frontmatter; C-layer onay-gate for identity changes) |
| Rule | `rules/version-check.md` | Automatic version checking on every prompt (hook-driven via `atl session-start`) |
| Rule | `rules/learning-capture.md` | Inline `<!-- learning -->` marker protocol; SessionStart-driven capture loop |
| Rule | `rules/docs-sync.md` | Proactive docs sync: README / doc-site updates in the same turn as user-facing changes |
| Rule | `rules/team-repo-maintenance.md` | Governance for team / global repo changes: semver bump + conventional commit + PR flow |
| Rule | `rules/karpathy-guidelines.md` | Four coding principles (Think Before Coding, Simplicity First, Surgical Changes, Goal-Driven Execution) |
| Rule | `rules/branch-hygiene.md` | Branch state hygiene |
| Template | `templates/journal-entry.md` | Template for journal entries |

## Installation

Core is installed automatically by `install.sh` (bootstrap). No manual installation needed.

## How It Works

### Two-Layer Knowledge

```
Team Repo (cross-project knowledge)                  Project (project-specific record)
~/.claude/repos/agentteamland/{team}/                .claude/
├── agents/{agent}/                                  ├── journal/{date}_{agent}.md  ← per-agent history
│   ├── agent.md  ← identity + auto-rebuild KB       │                                + inter-agent signals
│   └── children/{topic}.md                          ├── wiki/{topic}.md  ← current truth, topic-organized
│       ↳ knowledge-base-summary frontmatter         │                       (also auto-loaded into CLAUDE.md
│                                                    │                        as <!-- wiki:index --> block)
└── skills/{skill}/                                  └── docs/{topic}.md  ← settled brainstorm decisions
    ├── skill.md  ← procedure + auto-rebuild
    │              "Accumulated Learnings"
    └── learnings/{topic}.md
        ↳ knowledge-base-summary frontmatter
```

The previous "memory" layer is gone — see [knowledge-system.md](rules/knowledge-system.md) "Why two layers, not three".

### Session Lifecycle (post-v1.1.0)

```
Session Start (SessionStart hook → atl session-start --silent-if-clean):
  Step 1. atl update --silent-if-clean        (cache pull)
  Step 2. Per-project migration + auto-refresh of unmodified copies
  Step 3. atl learning-capture --previous-transcripts
            → reads ~/.claude/state/learning-capture-state.json
            → scans previous transcripts for unprocessed <!-- learning --> markers
            → emits compact report to stdout
            → output reaches Claude's additionalContext on first turn

  Agent startup routine (per knowledge-system.md):
    - Reads agent.md (with auto-rebuilt Knowledge Base section)
    - Sees <!-- wiki:index --> block in CLAUDE.md (auto-loaded knowledge map)
    - Reads recent journal entries when relevant
    - Reads project-specific rules

Session Work:
  (normal conversation)
  When a learning moment occurs → drop an inline <!-- learning --> marker
  in your response. If the change is user-facing → docs-sync rule kicks in:
  update README / doc site in the same turn, or mark doc-impact on the
  marker for next-session processing.

Next Session Start:
  Step 3 of session-start finds the markers from THIS session in the
  previous transcript. Output reaches additionalContext. You invoke:

    /save-learnings --from-markers --transcripts <path1>,<path2>,...

  /save-learnings:
    - Writes journal entry (idempotent by hash)
    - Updates wiki pages (replace-style for current truth)
    - Rebuilds <!-- wiki:index --> block in CLAUDE.md
    - Updates agent children/{topic}.md (with knowledge-base-summary frontmatter)
    - Rebuilds agent.md Knowledge Base section from children frontmatter
    - Updates skill learnings/{topic}.md (same frontmatter)
    - Rebuilds skill.md Accumulated Learnings section
    - For new skill / rule / agent / agent.md identity / skill.md core changes:
        AskUserQuestion (ONE multi-select prompt per run)
    - Writes ~/.claude/state/learning-capture-state.json (closes the loop)
    - Pushes team-repo writes (auto for maintainers, graceful fail otherwise)
```

The mechanism is split between:
- **`rules/learning-capture.md`** — inline marker protocol + SessionStart-driven capture flow
- **`rules/knowledge-system.md`** — journal + wiki layer definitions + agent startup routine
- **`rules/agent-structure.md`** — children + learnings patterns + KB / Accumulated Learnings auto-rebuild
- **`rules/docs-sync.md`** — proactive docs discipline (paired with learning-capture via `doc-impact`)
- **`atl session-start`** (CLI wrapper) — composite boot-time tasks
- **`atl learning-capture --previous-transcripts`** (CLI) — multi-transcript marker scan + state-file-driven dedup
- **`/save-learnings --from-markers --transcripts ...`** — processes markers into journal + wiki + children + learnings + advances state file

### Automatic Version Check

On `SessionStart` and (throttled) on every `UserPromptSubmit`, version-check silently:
1. Runs `git fetch` on all cached repos in `~/.claude/repos/agentteamland/`
2. If behind origin → auto `git pull`
3. Brief one-liner notification if updated (e.g., `🔄 software-project-team 1.1.4 → 1.2.0`)
4. Completely silent if no updates

No user confirmation needed. No interruption.

## License

MIT
