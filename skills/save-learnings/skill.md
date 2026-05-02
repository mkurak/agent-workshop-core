---
name: save-learnings
description: "Persist conversation learnings to journal + wiki + agent children + skill learnings, with auto-rebuild of agent.md / skill.md Knowledge Base sections from frontmatter. Multi-transcript input via --transcripts. Updates the learning-capture state file at completion (closes the auto-trigger loop). AskUserQuestion gates only for new structure creation: new agent, new skill, new rule, agent identity change, skill core change. Everything else is automatic."
argument-hint: "[agent-name] | --from-markers [--transcripts a.jsonl,b.jsonl,...]"
---

# /save-learnings Skill

## Purpose

Persist what was learned in this conversation (or in the marker-tagged regions of multiple transcripts), then update the state file so the same markers don't re-process on the next session. This skill is the **processing half** of the auto-trigger loop: `atl session-start` hook scans transcripts → reports unprocessed markers → Claude invokes this skill → markers land in journal/wiki/agent-children/skill-learnings → state file marks them as processed.

This skill writes a lot. Most writes are automatic — the user doesn't see prompts. The exceptions, where `AskUserQuestion` IS used:

1. **New skill creation** (when a workflow pattern repeated 2+ times)
2. **New rule creation** (when an "always do X" / "never do Y" convention crystallizes)
3. **New agent creation** (when a domain area is unmistakably a separate agent)
4. **Existing agent identity change** (when responsibility/principles need to shift — Q2 C-layer of self-updating-learning-loop)
5. **Existing skill core change** (when a skill's steps need to change — Q3 C-layer)

Everything else (journal entries, wiki updates, agent children, skill learnings, KB section auto-rebuilds) happens silently. The user reads the final summary; they don't approve each write.

## Three invocation modes

| Mode | Invocation | When |
|---|---|---|
| **Hook mode (auto-trigger)** | `/save-learnings --from-markers --transcripts a.jsonl,b.jsonl,c.jsonl` | `atl session-start` reports markers → Claude calls this on next turn |
| **Single-transcript mode** | `/save-learnings --from-markers` | Legacy: scans the current session's own transcript only |
| **Manual mode** | `/save-learnings [agent-name]` | User explicit: analyzes the live conversation (no markers required) |

In hook mode, the transcript paths are given. In single-transcript mode, the current session's transcript path is resolved from Claude Code context. In manual mode, no transcripts are read — the analysis runs against the conversation in memory.

## Flow

### 0. Parse invocation

```
if --from-markers and --transcripts:
    transcripts = parse comma-separated list
    mode = "hook"
elif --from-markers:
    transcripts = [current session transcript]
    mode = "single"
else:
    transcripts = []
    mode = "manual"
    agent_name = argument or inferred from edited files
```

### 1. Identify the active agent (manual mode only)

If a name was given as argument, use it. If not, infer from which agent files / agent-domain code were edited in this conversation. Pick the agent whose domain is closest to the work done.

In hook + single modes, agent identification happens per-marker (each marker may belong to a different agent).

### 2. Collect learnings

**In manual mode**: scan the conversation for learnings across these categories:
- **Patterns that worked / didn't work**
- **Emerging patterns** (not yet certain)
- **Process improvements**
- **Repeating workflows** (2+ times in this session — candidate for new skill)
- **New conventions** (always/never — candidate for new rule)
- **Bug fixes** (with root cause + fix path)
- **Discoveries** (non-obvious facts about systems / libraries / behavior)

**In hook + single modes**: extract every `<!-- learning -->` block from the listed transcripts. Each block already has `topic`, `kind`, `doc-impact`, `body`. Use those fields directly; don't re-classify.

**Deduplication**: hash each learning by (topic + body). If the same hash already appears in `.claude/journal/` for the same agent + same calendar date, skip it (the marker was double-emitted, or this is a re-run after a prior partial save).

### 3. Decide destination layers per learning

| Learning shape | Destination |
|---|---|
| Time-stamped narrative ("we tried X, then Y, then Z worked") | Journal entry only |
| Topic-shaped current truth ("the right way to do auth is …") | Wiki page (replace if exists) + journal entry |
| Domain knowledge for a specific agent ("api-agent: prefer prepared statements") | Agent's `children/{topic}.md` (Q2 A-layer) + journal entry |
| Domain knowledge for a specific skill ("save-learnings: skip duplicates by hash") | Skill's `learnings/{topic}.md` (Q3 A-layer) + journal entry |
| Repeating workflow (across this session OR cross-session per memory check) | **AskUserQuestion → new skill** |
| Convention crystallized ("never auto-merge", "always use prepared statements") | **AskUserQuestion → new rule** via `/rule` |
| Domain area without an owning agent | **AskUserQuestion → new agent** |
| Existing agent's identity expanded (e.g. api-agent now also covers caching) | **AskUserQuestion → agent.md core update** (Q2 C-layer) |
| Existing skill's core flow needs change | **AskUserQuestion → skill.md core update** (Q3 C-layer) |

The first four rows happen silently. The last five rows go through `AskUserQuestion` per the platform's requirement #2: *"If a new agent or skill is required, or if new ones need to be created, that should be possible. In cases like these, the user can be informed and asked for approval."* (Translation of the maintainer's verbatim TR statement; original kept in journal/brainstorm history.)

### 4. Write journal (Q4 — single layer for time-based content)

File: `.claude/journal/{YYYY-MM-DD}_{agent-name}.md`

If the file exists for this date+agent, append. If it doesn't, create it with YAML frontmatter:

```markdown
---
date: 2026-05-02
agent: <agent-name>
tags: [learning, <kind1>, <kind2>, ...]
---

## Summary

<one-paragraph what was done in this conversation / batch of transcripts>

## Learnings

- <short bullet per learning, with topic + kind + body>
- ...

## Auto-Created

- <list of any new files created — children/learnings/wiki pages>
- ...

## User-Approved Structural Changes

- <list of new skills / rules / agents / identity changes that AskUserQuestion approved>
  (or "(none)" when no structural changes were proposed in this run)

## Notes for Other Agents

- <cross-cutting notes, when applicable>
```

**Idempotency**: before writing each bullet, hash (kind + topic + body) and check if that hash already appears in any `.claude/journal/{date}_*.md` for the same date. Skip duplicates.

> **Note on the 3-layer → 2-layer transition (Q4 of self-updating-learning-loop)**: previous versions of this skill wrote to both `.claude/agent-memory/{agent}-memory.md` AND `.claude/journal/{date}_{agent}.md`. After the v1.1.0 ship, agent-memory has been merged into journal; only journal/ is written. Existing agent-memory files in workspaces will be migrated by Phase 2.B's workspace-migration PR.

### 5. Update wiki

For each learning whose shape is "topic-shaped current truth":

1. Determine wiki page slug from the learning's topic (kebab-case)
2. If `.claude/wiki/{slug}.md` exists → update (REPLACE outdated section, don't append)
3. If it doesn't → create with the standard wiki page format (Last updated, Current truth section, Related section)
4. Add a one-line entry to `.claude/wiki/index.md` if not already there

Wiki pages reflect **current truth** — when a fact changes, old fact is replaced. Don't pile up dated sections (that's what journal is for).

### 6. Rebuild CLAUDE.md wiki:index marker block

After any wiki page write/update/delete, regenerate the `<!-- wiki:index:start ... wiki:index:end -->` block at the top of CLAUDE.md (after the H1 + intro, near other marker blocks). Block contents:

```markdown
<!-- wiki:index:start -->
## 📚 Knowledge map

Knowledge lives in `.claude/wiki/` (current truth, topic-organized) and `.claude/journal/` (historical record, date-based).

**Wiki topics:**
- [topic-1](.claude/wiki/topic-1.md) — one-line summary
- [topic-2](.claude/wiki/topic-2.md) — one-line summary
- ...

**Discipline:** Before working on a topic, scan this list. If a topic looks relevant, read the page.
<!-- wiki:index:end -->
```

The summaries come from the first non-frontmatter, non-heading line of each wiki page (or from `index.md`'s existing one-liner if newer is unavailable). Sort topics by filename for determinism.

If the block doesn't exist yet (first run), insert it after the existing H1 + intro paragraph, before the first H2.

### 7. Update agent's children/ (Q2 A-layer — automatic)

For each learning destined for `children/{topic}.md`:

1. Resolve the agent's repo path: `~/.claude/repos/agentteamland/{team}/agents/{agent}/`
2. If `children/{topic}.md` exists → update (typically: append a new bullet under an existing section, or add a new section)
3. If not → create with YAML frontmatter:

```markdown
---
knowledge-base-summary: "<one-line summary of what this topic covers — used by agent.md KB section auto-rebuild>"
---

# <Topic Title>

<body>
```

The `knowledge-base-summary` frontmatter field is what step 8 reads to rebuild the agent.md Knowledge Base section. **Always write this field on file creation**; if updating an existing children file, leave the summary intact unless the topic's scope materially changed.

### 8. Rebuild agent.md Knowledge Base section (Q2 B-layer — automatic)

If any `children/*.md` was created or had its `knowledge-base-summary` updated in step 7, regenerate the **Knowledge Base** section in the agent's `agent.md`:

1. List every `children/*.md` file
2. Read each frontmatter `knowledge-base-summary`
3. Sort by filename
4. Replace the existing `## Knowledge Base` section content with:

```markdown
## Knowledge Base

### <Topic 1 (heading-cased from filename)>
<knowledge-base-summary>
→ [Details](children/topic-1.md)

### <Topic 2>
<knowledge-base-summary>
→ [Details](children/topic-2.md)

...
```

This section is fully derived from frontmatter. Hand-edits to it are overwritten on the next save-learnings run; the source of truth is each child file's frontmatter.

The `## Knowledge Base` heading itself, the agent's identity / responsibility / principles sections, and everything else in agent.md are **NOT touched** by this skill in automatic mode.

### 9. Update skill's learnings/ (Q3 A-layer — automatic, mirrors step 7)

For each learning destined for a specific skill:

1. Resolve the skill path: `~/.claude/repos/agentteamland/{team-or-core}/skills/{skill}/`
2. Ensure `learnings/` subdirectory exists (create if first learning for this skill)
3. Create or update `learnings/{topic}.md` with the same `knowledge-base-summary:` frontmatter pattern as children/

### 10. Rebuild skill.md Accumulated Learnings section (Q3 B-layer — automatic, mirrors step 8)

If any `learnings/*.md` was created/updated in step 9, regenerate the `## Accumulated Learnings` section in the skill's `skill.md`:

```markdown
## Accumulated Learnings

(Auto-rebuilt by /save-learnings from learnings/*.md frontmatter. Do not edit by hand.)

### <Topic 1>
<knowledge-base-summary>
→ [Details](learnings/topic-1.md)

### <Topic 2>
<knowledge-base-summary>
→ [Details](learnings/topic-2.md)

...
```

If `## Accumulated Learnings` doesn't exist yet, append it at the end of skill.md (after the existing content). Subsequent rebuilds replace the section in place.

### 11. AskUserQuestion gates (Q2 C-layer + Q3 C-layer + new structure)

For each learning whose destination from step 3 is one of the gated rows, prepare a single AskUserQuestion with multiple-choice options. Batch all gated learnings into ONE question per skill run when possible (don't ask 5 separate questions in 5 separate prompts).

Question format example:

```
Three structural changes are proposed by this batch of learnings. Approve which?

[1] New skill: /repo-status
    Reason: pattern "check repo state across all peer repos" appeared 3 times
    Files: .claude/skills/repo-status/skill.md (new, ~100 lines)

[2] Update agent identity: api-agent
    Reason: domain expanded to include caching strategy
    Diff: agent.md "Responsibility" section adds a caching paragraph
    
[3] Update skill.md core: /create-pr
    Reason: review chain now needs a CI-pre-merge poll loop
    Diff: skill.md step 6 adds a `until COMPLETED && SUCCESS` poll

Approve which? (multi-select)
[ ] (1) Create skill /repo-status
[ ] (2) Update api-agent identity
[ ] (3) Update /create-pr skill.md
[ ] All
[ ] None (skip all structural changes; everything else still saves)
```

For each approved item, apply the change. For each rejected item, log the rejection to journal under "## User-Approved Structural Changes" with reason "rejected".

When AskUserQuestion isn't applicable (e.g., running non-interactively in a hook context where no user is present), default to "skip all" and log every gated proposal as "deferred — surfaced next interactive session". The state file write in step 12 still proceeds for the non-gated learnings.

### 12. Write state file (closes the auto-trigger loop)

After all writes from steps 4-11 complete (success or partial-success), update `~/.claude/state/learning-capture-state.json`:

```bash
# Pseudocode — actual implementation uses Read + Edit tools to manipulate JSON
state_file = ~/.claude/state/learning-capture-state.json
project_slug = current project's Claude Code session slug (replace / with - in cwd path)
latest_modtime = max(modtime of each transcript in --transcripts list)

state = read state_file (or empty if missing)
state.projects[project_slug].lastProcessedAt = latest_modtime
write state atomically (tmp file + rename)
```

This is what tells the next `atl learning-capture --previous-transcripts` run that these markers are processed — they won't be re-reported on the next SessionStart hook fire.

**Atomicity**: write to a temp file first, then `mv` over the real path. If anything in steps 4-11 silently failed, that's still better than re-processing every marker forever; the next save-learnings invocation can re-derive what's missing from the journal/wiki/etc. (those are append-only or replace-with-frontmatter, both idempotent).

**On total failure** (e.g., disk full, all writes failed): do NOT update state file. Next session re-reports — losing nothing.

### 13. Push team repo (if any team-repo files were modified)

For each team repo whose files (children/, learnings/, agent.md, skill.md, known-issues.md, etc.) were modified by this run:

```bash
cd ~/.claude/repos/agentteamland/{team-name}
git add -A
git commit -m "learn: <one-line summary of the batch>"
git push
```

This pushes to the **user's local clone** of the team repo. The clone's origin is the public `agentteamland/{team}` repo, so users without push permission will see a permission-denied error here. That's expected — they keep their changes locally; the upstream contribution flow (separate brainstorm: `upstream-contribution-stream`) is what eventually packages them as a PR.

For team-repo maintainers with direct push access, this push lands on a PR branch (created automatically) and goes through the standard branch-protection PR gate. Public users without push permission see the permission error and stay local-only — their changes are preserved in the project clone for later upstream contribution.

### 14. Report to user

```
📝 Learnings saved (mode: hook | single | manual)

  Markers processed: <N> across <M> transcripts
  Journal: 1 entry (.claude/journal/2026-05-02_<agent>.md)
  Wiki: <X> pages updated, <Y> new pages, <Z> wiki:index marker rebuilt
  Agent children: <A> updated across <B> agents (KB sections auto-rebuilt)
  Skill learnings: <C> updated across <D> skills (Accumulated Learnings auto-rebuilt)

  Structural changes:
    Approved by user:
      • <list, or "(none)">
    Rejected:
      • <list, or "(none)">

  State file updated: <slug> last-processed-at = <timestamp>
  Team repos pushed: <list, or "(none — no team-repo writes)">
```

Keep the report concise. Detail belongs in journal; this report is just the "did the loop close?" summary.

## Important rules

1. **No confirmation for non-structural writes.** Memory/journal/wiki/agent-children/skill-learnings all happen silently. The user sees the result, not a question per write.

2. **AskUserQuestion ONLY for new structures or identity changes.** New skill, new rule, new agent, agent.md identity update, skill.md core update. These are gated per platform requirement #2.

3. **State file write is the closing bracket.** Until the state file updates, the markers remain "unprocessed" and will re-report on next SessionStart. This is the safety net against partial-write data loss.

4. **Sensitive information filter.** Passwords, tokens, API keys, secrets are NEVER written to journal/wiki/team repos. If a marker body contains what looks like a credential, redact and write the redacted form.

5. **Idempotent everywhere.** Re-running this skill on the same markers should produce no incremental change to journal (dedup by hash), no change to wiki (replace-with-same is a no-op), and no change to agent KB sections (full rebuild from frontmatter is deterministic). The state-file timestamp may advance slightly between runs; that's fine.

6. **Team repo push is automatic for maintainers, fails gracefully for users.** Don't try to PR-instead-of-push when the push fails — that's the upstream-contribution-stream brainstorm's job. Just log the failure to journal and move on.

7. **Approval-gate batching.** When multiple structural changes appear in one batch of markers, ask one AskUserQuestion with multi-select, not N separate prompts. Platform requirement #3 ("manual operation must not be required") still allows the gates from #2, but should keep the friction at one decision point per save-learnings run.

8. **Skill creation threshold = 2 instances.** Don't auto-propose a new skill on a single workflow occurrence. Either the same workflow appears 2+ times in this batch of markers, OR it appears once in the markers AND once in journal history (cross-session pattern detection).

9. **Rule creation criteria = unambiguous "always X" / "never Y" wording in the marker body.** Hedged language ("we should probably do X") goes to wiki, not rule.

## History

This skill was introduced in `core@1.0.0` (manual mode only) and rewritten in `core@1.8.0` (Phase 2.B of self-updating-learning-loop) to add: hook mode (`--from-markers --transcripts`), multi-transcript input, state-file write, AskUserQuestion gates batched per run, and the Q2/Q3 layered model that auto-grows agent `children/` and skill `learnings/` directories with KB-section auto-rebuild.

Phase 2.A (`atl v1.1.0`) shipped the SessionStart wrapper that triggers this skill via additionalContext. Phase 2.C migrated 195 children files in software-project-team + design-system-team to `knowledge-base-summary` frontmatter. Phase 2.D (docs site refresh) is the remaining work tracked in workspace `CLAUDE.md`.

## Accumulated Learnings

(Auto-rebuilt by /save-learnings from learnings/*.md frontmatter. Do not edit by hand. Currently empty — populates as the skill is used and edge-case learnings accumulate.)
