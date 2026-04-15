# Memory & Journal System Rules

## Two-Layer Memory

### Project Memory (project-specific)
**Location:** `.claude/agent-memory/{agent-name}-memory.md`

Each project can have its own memory file for each agent. This file stores what was learned in that project. The agent reads this file at the start of a conversation.

**Rules:**
- The agent should read its own memory file at the start of a conversation (if it exists)
- Updated only via `/save-learnings` (manual editing is also possible)
- Project-based — different projects have different memories
- Format: date-headed, categorized (what worked / what didn't work / emerging)

### Team Knowledge Base (global)
**Location:** `~/.claude/repos/mkurak/{team}/agents/{agent}.md`

The agent's knowledge that applies across all projects. Rarely changes but learnings from any project can be added here as well.

**Rules:**
- Updated when "all projects" is selected via `/save-learnings`
- Automatic git commit + push after update
- This file is accessed via symlink from `~/.claude/agents/`

## Journal (cross-agent sharing)

**Location:** `.claude/journal/{date}_{agent-name}.md`

Information sharing between agents. When an agent discovers something, it writes to the journal; other agents read it in subsequent conversations.

**Rules:**
- Every agent can read the journal
- Every agent can write to the journal under its own name
- Journal files are never deleted (historical record)
- Date-based file name: `2026-04-13_api-agent.md`
- Journal is different from brainstorm files — journal contains short notes, brainstorm contains long discussions

## Agent Startup Routine

At the start of every conversation, the agent should read the following files (if they exist):

1. Its own agent file (from team, via symlink)
2. Project memory: `.claude/agent-memory/{agent-name}-memory.md`
3. Recent journal entries: `.claude/journal/` (last 5-10 entries)
4. Project-specific rules: `.claude/docs/coding-standards/{app}.md`

## End of Conversation Routine

**MANDATORY:** Before the conversation ends, proactively check if anything was learned. Do NOT wait for the user to ask — the user may forget. This is YOUR responsibility.

When the conversation involved any of the following, learnings MUST be saved:
- A bug was found and fixed
- A new pattern or approach was discovered
- Something didn't work and a workaround was applied
- A decision was made that affects future development
- A tool, library, or configuration behaved unexpectedly

**Process:**
1. Before ending, ask the user: "I learned some things in this session. Should I save them?"
2. List what was learned (briefly)
3. If approved, write to project memory, team repo (if general), and journal
4. If team repo updated, auto git push

If nothing was learned (simple Q&A, no development work), skip silently — don't ask unnecessarily.
