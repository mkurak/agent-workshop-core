# Knowledge system — journal + wiki

(Renamed from `memory-system.md` in `core@1.8.0` to reflect the post-Q4 reality: there is no separate "memory" layer anymore. The two date-based layers — agent memory + journal — were merged into a single `journal/` layer in [self-updating-learning-loop](https://github.com/agentteamland/workspace/blob/main/.claude/docs/self-updating-learning-loop.md) Q4. The previous file name was misleading.)

## Two-Layer Knowledge

| Layer | Location | Purpose | Update style |
|---|---|---|---|
| **Journal** | `.claude/journal/{YYYY-MM-DD}_{agent-name}.md` | Date-based historical record. Per-agent learning history AND inter-agent signals (the two layers were merged in Q4 — they were redundant in practice). | Append-only |
| **Wiki** | `.claude/wiki/{topic}.md` | Topic-based current truth. Reflects what is true NOW; old facts are replaced, not appended. | Replace / update |

That's it. Two layers. Don't add more.

### Journal

**Location:** `.claude/journal/{date}_{agent-name}.md`

What goes here:
- Date-stamped narrative of what happened: discoveries, decisions, bug fixes, what worked, what didn't
- Cross-agent notes ("for whoever touches X next: ...")
- Auto-created artifact lists ("this session created wiki page Y, agent children file Z")
- User-approved structural changes (new skill / rule / agent decisions and rejections)

**Rules:**
- Filename: `{date}_{agent}.md`. Multiple agents working on the same date → multiple files; same agent multiple times in a day → all in one file (use sub-headings or addendum sections).
- Append-only. Existing entries are not edited; new entries go at the end.
- Idempotency: when `/save-learnings` writes a journal bullet, it hashes (kind + topic + body) and skips duplicates already in this file or any same-date `.claude/journal/*.md`.
- Never deleted (historical record).
- `*.local.md` filename pattern is gitignored — use it for genuinely private content (uncommon).

The journal layer is what `.claude/agent-memory/` USED to be (per-agent history) PLUS what the original journal layer was (cross-agent signals). Q4 of self-updating-learning-loop merged them because in practice the two layers had identical format (date + agent + narrative) and frequently cited each other anyway.

### Wiki

**Location:** `.claude/wiki/{topic}.md`

The project's living knowledge base. Unlike journal (historical record), wiki reflects **current truth** — when a fact changes, the page is updated, not appended.

**Rules:**
- Organized by topic, not by date (one page per concept)
- Updated from `<!-- learning -->` markers via `/save-learnings`, or directly from `/wiki ingest`
- Pages reflect what is true NOW — old info is replaced, not appended
- Cross-referenced: related pages link to each other
- `index.md` auto-maintained as table of contents
- A `<!-- wiki:index -->` marker block at the top of CLAUDE.md auto-aggregates the topic list (per [self-updating-learning-loop Q5](https://github.com/agentteamland/workspace/blob/main/.claude/docs/self-updating-learning-loop.md))
- Bootstrap: run `/wiki init` in a project without `.claude/wiki/` to scaffold it
- Lint with `/wiki lint` periodically

## Agent Startup Routine

At the start of every conversation, the agent reads (when applicable):

1. **Its own agent file** — from team, via symlink or copy. The agent.md ships with a Knowledge Base section auto-aggregated from children/*.md frontmatter (per [agent-structure.md](agent-structure.md)).
2. **CLAUDE.md `<!-- wiki:index -->` block** — auto-loaded; gives the knowledge map at zero cost. Agents discover relevant wiki pages from this list rather than scanning `.claude/wiki/` directly.
3. **Recent journal entries** when the task overlaps with prior work — `.claude/journal/` (last 3–5 entries by default; the agent extends scope when the task touches a long-running thread)
4. **Project-specific rules** at `.claude/docs/coding-standards/{app}.md` if present

The agent does NOT read all wiki pages. It reads the index (auto-loaded), and only follows links to detail pages when the task touches that domain. This keeps context tight while preserving discoverability.

## End of Conversation Routine — auto-trigger via SessionStart

The "save at session end" semantic is now implemented as **scan-on-next-session-start** because Claude Code's `SessionEnd` hook output never reaches the next-session Claude (per [learning-capture.md](learning-capture.md) "History" section). The auto-trigger flow is:

- **During conversation:** drop `<!-- learning -->` markers when learning moments occur. See [learning-capture.md](learning-capture.md) for marker format and discipline.
- **At next session start:** `atl session-start` wrapper → `atl learning-capture --previous-transcripts` → output appears in additionalContext → you invoke `/save-learnings --from-markers --transcripts ...` → loop closes.
- **When a change is user-facing:** in the same turn the change is made, also update the matching README / doc-site page. See [docs-sync.md](docs-sync.md).

If `atl setup-hooks` is not installed, markers still accumulate in the transcript and remain available for manual `/save-learnings` invocation. The hook flow is the automation; the marker discipline + manual invocation is the foundation.

## Why two layers, not three

Earlier versions of this rule defined three layers: **memory** (per-project, per-agent, append-only history), **journal** (per-project, cross-agent signals, append-only), **wiki** (per-project, topic-based, replace/update).

The first two were both date-based, append-only, narrative-shaped. In every workspace they ended up cross-referencing each other or redundantly capturing the same events. The "agent's private memory vs. broadcast to others" distinction was never enforced — anyone could read either layer.

Q4 of self-updating-learning-loop merged them because:
- Same format → no semantic separation
- Same audience (all agents read both)
- Same write pattern (append by date)
- The split added cognitive overhead ("is this for me or for others?") without producing different content

The merged layer is just `journal/`. Wiki stays separate because its paradigm (topic-based current truth) is genuinely different from journal's (date-based history).

## Per-team / per-project mirrors

The same two-layer system applies inside the user's project (`.claude/journal/`, `.claude/wiki/`) AND on the team-repo side for cross-project knowledge:

- **Agent children files** (`children/{topic}.md` in the team repo's agent directory) are the team-side equivalent of wiki — topic-based, replace/update, cross-project domain knowledge for the agent.
- **Skill learnings files** (`learnings/{topic}.md` in the team repo's skill directory) are the per-skill equivalent — same shape, scoped to the skill.

Both have a `knowledge-base-summary:` frontmatter field that's auto-aggregated into agent.md (Knowledge Base section) or skill.md (Accumulated Learnings section) — see [agent-structure.md](agent-structure.md) for the extraction pattern.

## History

- Pre-`atl`: hand-edited memory files, no convention. Each project rolled its own.
- `core@1.0.0`–`1.4.x`: three-layer model (memory + journal + wiki). Worked, but felt redundant.
- `core@1.8.0` (this version): two-layer model (journal + wiki) per Q4 of [self-updating-learning-loop](https://github.com/agentteamland/workspace/blob/main/.claude/docs/self-updating-learning-loop.md). Renamed from `memory-system.md` to `knowledge-system.md` to remove the misleading "memory" framing. Per-project agent-memory files in existing workspaces are migrated to `agent-memory-archive.md` snapshots in their respective `.claude/docs/` directories.
