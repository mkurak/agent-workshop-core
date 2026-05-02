# Agent + skill structure rules

(Renamed in scope from "Agent Configuration Rules" in `core@1.8.0` — the same children + Knowledge-Base pattern now applies to skills too, per [self-updating-learning-loop Q3](https://github.com/agentteamland/workspace/blob/main/.claude/docs/self-updating-learning-loop.md). Single mental model across agent and skill.)

## Children Pattern (agent) — Mandatory

Every agent is organized in the following structure:

```
~/.claude/repos/agentteamland/{team}/agents/{agent-name}/
├── agent.md              ← Identity, area of responsibility, core principles (short, embedded)
└── children/             ← Detailed information, patterns, strategies (each topic in a separate file)
    ├── topic-1.md
    ├── topic-2.md
    └── ...
```

### Rules

1. **agent.md stays short.** Only: identity, area of responsibility (positive list), core principles (unchanging, short bullet points), Knowledge Base section (auto-aggregated from children/), "read children/" instruction.
2. **Everything detailed goes under children/.** Strategies, patterns, workflows, conventions — each in a separate .md file.
3. **New topic = new file.** Without touching agent.md by hand, add a .md file under children/. The Knowledge Base section in agent.md is rebuilt automatically by `/save-learnings` from each child file's frontmatter.
4. **Update = single file.** To update a topic, only the relevant children file is touched.
5. **Monolithic agent files are prohibited.** Piling all information into a single .md is prohibited — it becomes unmanageable.
6. **This pattern applies to all agents.** API, Socket, Worker, Flutter, React, Mail, Log, Infra — all follow the same structure.

## Learnings Pattern (skill) — Mandatory (new in core@1.8.0)

Every skill mirrors the agent structure:

```
~/.claude/repos/agentteamland/{team-or-core}/skills/{skill-name}/
├── SKILL.md              ← The skill's procedure (steps, identity, flow). Stays short.
└── learnings/            ← Accumulated edge cases, successful patterns, anti-patterns
    ├── topic-1.md
    ├── topic-2.md
    └── ...
```

This mirrors `children/` for agents. Same shape, same rules, same `knowledge-base-summary` frontmatter convention. The skill's `SKILL.md` ships with an "Accumulated Learnings" section auto-aggregated from `learnings/*.md` frontmatter — same mechanism as agent.md's Knowledge Base.

**Why mirror agents on skills?** Per Q3 of self-updating-learning-loop: the "self-improving skill" framing benefits from a structured place for accumulated wisdom that agents (Claude) can see when invoking the skill. Without `learnings/`, every skill use starts from zero on edge cases that came up in prior runs.

## Knowledge Base Format (agent.md) — Mandatory

In agent.md's Knowledge Base section, every children topic MUST be listed with:
1. **A heading** with the topic name (heading-cased from the filename)
2. **A 2-3 line summary** copied from the child file's `knowledge-base-summary` frontmatter field
3. **A detail link** pointing to the children file: `→ [Details](children/{topic}.md)`

**This section is auto-rebuilt** by `/save-learnings` from each child file's frontmatter. Hand-edits to the section are overwritten on the next save-learnings run; the source of truth is each child file's frontmatter.

### Required frontmatter on every children/*.md

```markdown
---
knowledge-base-summary: "<one-to-three-line summary used in agent.md Knowledge Base section>"
---

# <Topic Title>

<the actual content — patterns, strategies, examples — as long as needed>
```

The `knowledge-base-summary` field is REQUIRED. It's what feeds the Knowledge Base section. Without it, save-learnings either skips the topic in the rebuild OR (for new files it created itself) writes the field with a generated summary; in both cases the file should have one.

### Example agent.md Knowledge Base section (auto-rebuilt)

```markdown
## Knowledge Base

### Logging Strategy
Log every step in every handler. Production has no breakpoints — logs are your debugger.
No performance concern: pipeline is non-blocking (Channel → RMQ).
→ [Details](children/logging-strategy.md)

### Error Handling
Exception hierarchy: NotFoundException→404, ValidationException→422, ForbiddenException→403.
No try-catch in handlers — throw exception, global handler catches.
→ [Details](children/error-handling.md)
```

## Accumulated Learnings Format (SKILL.md) — Mandatory (new in core@1.8.0)

Same shape as agent.md's Knowledge Base, applied to SKILL.md. Each skill's SKILL.md gets an "Accumulated Learnings" section appended (or rebuilt in place if the section exists):

```markdown
## Accumulated Learnings

(Auto-rebuilt by /save-learnings from learnings/*.md frontmatter. Do not edit by hand.)

### Skip duplicate journal entries by hash
Hash (kind + topic + body); skip when same hash exists for the same date.
Catches the case where a marker was double-emitted in the transcript.
→ [Details](learnings/skip-duplicates-by-hash.md)

### When to escalate to AskUserQuestion
Only for new skill / new rule / new agent / agent.md identity / SKILL.md core.
Everything else is automatic — Mesut's #2 + #3 requirements.
→ [Details](learnings/when-to-askuser.md)
```

Each `learnings/*.md` file follows the same frontmatter convention as `children/*.md`:

```markdown
---
knowledge-base-summary: "<one-to-three-line summary>"
---

# <Topic>

<content>
```

## Identity / core changes — onay-gated (C-layer of self-updating-learning-loop)

The `agent.md` sections OTHER than Knowledge Base — **identity, area of responsibility, core principles** — are NOT touched by automatic save-learnings. The same goes for the `SKILL.md` core (steps, flow, identity).

When `/save-learnings` infers from accumulated markers that one of these sections needs to change (e.g., "api-agent's responsibility now also covers caching", or "the /create-pr skill's review chain needs a new step"), it raises an `AskUserQuestion` gate. The user approves, the file is updated; the user rejects, the proposal is logged to journal as "rejected" with the reason.

This is the C-layer of the Q2 (agent self-update) and Q3 (skill self-update) split:
- **A-layer (auto):** children/learnings file is created or updated
- **B-layer (auto):** Knowledge Base / Accumulated Learnings section is rebuilt from frontmatter
- **C-layer (gated):** core identity / responsibility / principles / skill flow changes

This split makes "knowledge accumulates" automatic and frictionless, while protecting the agent's / skill's identity from drift via the user's explicit approval per Mesut's #2 requirement (new structure creation can ask).

## Blueprint Pattern (agent) — Mandatory

Every agent has a **primary production unit** — the main thing it creates repeatedly. This unit MUST have a blueprint file in `children/` that contains:

1. **Template** — the structural skeleton of the production unit (code scaffold)
2. **Checklist** — everything that must be verified before the unit is complete
3. **Naming conventions** — how files, classes, methods are named
4. **Lifecycle** — creation → registration → testing flow

When the agent needs to create a new instance of its production unit, it reads the blueprint and follows it step by step.

| Agent | Primary Production Unit | Blueprint File |
|-------|------------------------|----------------|
| API Agent | Feature (Command/Query/Handler/Validator) | `children/workflows.md` |
| Socket Agent | Hub method + Event | `children/hub-method-blueprint.md` |
| Worker Agent | Scheduled Job | `children/job-blueprint.md` |
| Flutter Agent | Screen / Widget | `children/screen-blueprint.md` |
| React Agent | Component / Page | `children/component-blueprint.md` |

### Why this matters

Without a blueprint, the agent guesses how to create new units. With a blueprint:
- Every unit follows the same structure
- Nothing is forgotten (checklist guarantees completeness)
- New team members (or new Claude sessions) produce consistent output
- Quality is repeatable, not accidental

(Skills don't have a blueprint pattern — a skill IS the procedure, not a template-driven unit. The Accumulated Learnings section is the skill's equivalent of a blueprint's checklist: things to remember, edge cases to watch for.)

## Migration from pre-1.5.0 layouts

For teams already shipping under `core@<1.5.0`:

- **Agents already have `children/`** — keep it. Add `knowledge-base-summary` frontmatter to each existing child file (the file's content is unchanged; only frontmatter is added at the top).
- **Skills do NOT have `learnings/` yet** — bootstrap: create the empty directory, append `## Accumulated Learnings` section to the SKILL.md (initially empty body), let it grow naturally as `/save-learnings` runs.
- **agent.md's Knowledge Base section** — keep what's there; the next `/save-learnings` run will rebuild it from frontmatter (existing summaries preserved if frontmatter copies them).

The migration is per-team and is owned by Phase 2.C of self-updating-learning-loop (separate PRs per team repo: software-project-team, design-system-team, etc.).

## History

- `core@1.0.0`: agent children pattern introduced. Knowledge Base section was hand-maintained.
- `core@1.8.0` (this version): Q3 extends children pattern to skills (`learnings/` mirror of `children/`). Knowledge Base + Accumulated Learnings sections become auto-rebuild from frontmatter. C-layer onay gate for identity / core changes formalized as part of the rule. Renamed from "Agent Configuration Rules" to "Agent + skill structure rules" to reflect the broader scope.
