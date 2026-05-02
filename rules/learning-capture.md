# Learning capture (inline marker protocol)

## Who runs this

**You (the agent) drop markers inline as you speak.** The Claude Code harness auto-runs `atl session-start` at the start of every new session — and that wrapper, on its third internal step, scans the **previous** session's transcript files for `<!-- learning -->` markers via `atl learning-capture --previous-transcripts`. The output appears in your additionalContext on the first turn of the new session, naming a recommended invocation:

```
🧠 learning-capture: 7 unprocessed markers across 2 transcripts
  by kind: 3 decision, 2 pattern, 1 discovery, 1 bug-fix
  3 markers require doc drafts (README / doc site) — see docs-sync rule

→ Run: /save-learnings --from-markers --transcripts <path1>,<path2>
```

Follow that recommendation. Run `/save-learnings --from-markers --transcripts ...` on the first turn (or as soon as the conversation allows). The skill writes journal + wiki + agent children + skill learnings, advances the state file's `lastProcessedAt`, and the same markers will not re-report on the next session start.

Markers are the "save it if you see it" mechanism. They are cheap to drop (~40 tokens) and free to ignore when nothing interesting happened.

## What counts as a learning moment

Any of these, when they happen during a conversation, is a learning moment:

- **Bug fix** — a real bug was reproduced and fixed
- **Decision** — a choice was made between alternatives (JWT vs session, Redis vs memcached, 7d vs 15d refresh)
- **Pattern** — an approach turned out to be clean and reusable
- **Anti-pattern** — something was tried, failed, and we know why
- **Discovery** — a non-obvious fact about the system, library, or external service
- **Convention** — "from now on, we always / never do X"

Routine Q&A, file lookups, and mechanical edits are NOT learning moments. Don't mark every response.

## How to mark

Drop an HTML comment in your response text when a learning moment occurs. The comment is invisible in rendered output but preserved in the transcript the hook scans.

```
<!-- learning
topic: auth-refresh
kind: decision
doc-impact: readme
body: 7-day JWT refresh chosen because we want long sessions; user logs in once a week max.
-->
```

**Fields:**

- `topic` — kebab-case, one concept (auth-refresh, redis-ttl, build-pipeline). Becomes the wiki page name.
- `kind` — one of `bug-fix | decision | pattern | anti-pattern | discovery | convention`
- `doc-impact` — one of `none | readme | docs | both | breaking`. Default `none` when unsure.
- `body` — one to three sentences. **Always include the WHY.** A 6-month-old "we chose X" without reasoning is useless.

Multiple markers per response are fine when multiple learnings happen. Do NOT bundle unrelated learnings into one marker — each topic deserves its own.

## What happens after — the auto-trigger loop

```
[previous session ends] no hook, no work — markers sit in the transcript
        ↓
[new session starts]
        ↓
SessionStart hook fires → atl session-start --silent-if-clean
        ↓
   step 3: atl learning-capture --previous-transcripts
        → reads ~/.claude/state/learning-capture-state.json (per-project lastProcessedAt)
        → enumerates ~/.claude/projects/{slug}/*.jsonl modified after that timestamp
          (or last 7 days on first run for this project)
        → scans those transcripts for <!-- learning --> blocks
        → emits compact report to stdout
        ↓
Claude Code injects the stdout into additionalContext for the first model request
        ↓
[your turn, in the new session]
        ↓
You read the report; invoke /save-learnings --from-markers --transcripts <paths>
        ↓
/save-learnings:
   • categorizes each marker (decision / pattern / discovery / ...)
   • writes journal entry (.claude/journal/<date>_<agent>.md, idempotent by hash)
   • updates wiki pages (.claude/wiki/<topic>.md, replace-style for current truth)
   • rebuilds <!-- wiki:index --> marker block in CLAUDE.md
   • updates agent children/<topic>.md (with knowledge-base-summary frontmatter)
   • rebuilds agent.md Knowledge Base section from children frontmatter
   • updates skill learnings/<topic>.md (same frontmatter pattern)
   • rebuilds skill.md Accumulated Learnings section from frontmatter
   • for new skill / rule / agent / agent identity / skill.md core changes:
       AskUserQuestion (ONE multi-select prompt per run)
   • writes ~/.claude/state/learning-capture-state.json (closes the loop)
   • pushes team-repo writes (auto for maintainers, graceful fail for users)
        ↓
Markers are now persisted. Next session start, the same markers won't re-report.
```

This loop is end-to-end automatic except for **two human touch points**:

1. **You** invoke `/save-learnings --from-markers --transcripts ...` after seeing the additionalContext recommendation. Per Mesut's design, that's a single command call — no manual marker-by-marker review.
2. **The user** answers the AskUserQuestion gate when new structures (skill / rule / agent / identity / skill.md core change) are proposed. One multi-select prompt per run.

Everything else (journal, wiki, children, learnings, KB rebuilds, state advance) happens silently.

## Why inline markers, not a tool call?

A tool call per learning would double token cost and slow conversation. Inline markers are embedded in text you were going to produce anyway. A grep-level hook finds them at ~0 cost; the AI-heavy save-learnings work only runs when markers exist — boring sessions stay free.

## When to skip

- Purely conversational turns (greetings, clarifications, status questions)
- Reading a file and summarizing its contents (no decision, no discovery)
- Routine edits where nothing surprising happened
- A learning already captured by a recent marker in the same session (don't duplicate)

## Dual with docs-sync

The `doc-impact` field ties this rule to [docs-sync](docs-sync.md). If you mark `doc-impact: readme` or `docs`, docs-sync takes over to prepare the actual README / doc-site changes. You don't need to update docs manually in the same turn — marking is enough, as long as you do mark.

## When the hook isn't installed

Markers are harmless when no hook processes them — they're HTML comments, invisible in rendered output, inert as text. The capture habit is still valuable (markers are legible even to a human reader of the transcript).

For automatic processing, the user runs `atl setup-hooks`. That installs:
- `SessionStart → atl session-start --silent-if-clean` (the wrapper that triggers learning-capture as step 3)
- `UserPromptSubmit → atl update --silent-if-clean --throttle=30m` (cache pull throttle)

Without those hooks, the user must invoke `/save-learnings` manually at session boundaries; markers still accumulate in transcripts and remain available for whenever processing happens.

## History

This rule has gone through three shapes:

1. **Original (pre-`atl` versions):** "Claude should proactively save learnings at the end of every session." Worked sometimes; depended on Claude remembering a prose instruction. Unreliable.

2. **First atl version (v0.2.0 — `core@1.3.0`):** Inline markers + `atl learning-capture` registered on `SessionEnd` and `PreCompact` hooks. **Silently broken**: per Claude Code v2.1.x docs, those events do NOT deliver hook stdout to Claude's additionalContext. Marker reports went to debug logs and were lost. 324 markers across 9 sessions in the maintainer workspace produced zero auto-processing during the month it was in production. All actual save-learnings work in that period was triggered by manual user invocation, not hook output. See [claude-code-hook-output-events.md](https://github.com/agentteamland/workspace/blob/main/.claude/wiki/claude-code-hook-output-events.md) in workspace.

3. **Current (v1.1.0+ — `core@1.8.0`):** Hook moved to `SessionStart` via the new `atl session-start` wrapper, scanning the previous session's transcripts via the new `--previous-transcripts` mode. Output reaches additionalContext. State file (~/.claude/state/learning-capture-state.json) tracks `lastProcessedAt` per project, written by `/save-learnings` on successful completion. The loop closes deterministically.

The current shape was decided in the [self-updating-learning-loop](https://github.com/agentteamland/workspace/blob/main/.claude/docs/self-updating-learning-loop.md) brainstorm (workspace, 2026-05-02) and shipped via Phase 2.A (cli v1.1.0) + Phase 2.B (this rule + the save-learnings skill rewrite).
