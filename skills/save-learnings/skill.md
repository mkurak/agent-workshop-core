---
name: save-learnings
description: "Save learnings at the end of a conversation. Project-specific information is written to agent-memory, general information is written to the agent file in the team repo and automatically pushed."
argument-hint: "[agent-name]"
---

# /save-learnings Skill

## Purpose

Called at the end of each conversation. Persists things learned during this conversation (new patterns, anti-patterns, discoveries, lessons from mistakes). This way the agent becomes smarter in subsequent conversations.

## Flow

### 1. Identify the Active Agent

If an agent name is given as an argument, use it. If not, infer from context which agent was working in this conversation (which files were edited, which directories were touched).

### 2. Analyze the Conversation

What was learned in this conversation? Scan according to these categories:

- **Patterns that worked** — "We did it this way and it worked well"
- **Patterns that didn't work** — "We tried this but it caused problems, because of this reason"
- **Emerging patterns** — "Not certain yet but there's this tendency"
- **Process improvements** — "This step was missing / unnecessary in the agent's workflow"
- **New rules** — "From now on we should always / never do this"

### 3. Summarize and Confirm with User

Show the found learnings to the user and ask:

```
We learned the following in this conversation:

1. [Worked] When EF Core Include chains exceed 3, projection should be used
2. [Anti-pattern] Storing values larger than 1MB in Redis causes timeouts
3. [Process] Consumers should declare their own topologies

Should I save these? For each one:
- This project only (memory)
- All projects (team repo)
- Don't save (skip)
```

Offer options for each learning using AskUserQuestion.

### 4. Write to Project Memory (project-specific ones)

File: `.claude/agent-memory/{agent-name}-memory.md`

Create if it doesn't exist. If it exists, append. Format:

```markdown
## {Date}

### What Worked
- {learning} — Evidence: {what happened}

### What Didn't Work
- {learning} — Evidence: {what happened}

### Emerging Patterns
- {observation} — Not yet verified
```

### 5. Write to Team Repo (general ones)

The agent file is edited via symlink — it actually updates `~/.claude/repos/mkurak/{team}/agents/{agent}.md`.

Types of updates to make:
- **New rule** — Add to the relevant section in the agent file
- **New pattern** — Add to the relevant children section
- **Workflow update** — Update the workflow steps

### 6. Push Team Repo (if there's a general update)

```bash
cd ~/.claude/repos/mkurak/{team-name}
git add -A
git commit -m "learn: {short learning summary}"
git push
```

Notify the user: "Team repo updated and pushed."

### 7. Write to Journal (if available)

If the core's journal system is active, also write learnings to the journal — so other agents can benefit.

File: `.claude/journal/{date}_{agent-name}.md`

```markdown
---
date: {date}
agent: {agent-name}
tags: [learning, {category}]
---

## Learnings

- {learning list}

## Notes for Other Agents

- {cross-cutting information if any}
```

## Important Rules

1. **Can be called at the end of every conversation.** Not mandatory but encouraged.
2. **Does not write without user confirmation.** Show learnings, confirm, then write.
3. **Git push is automatic.** If there's a team repo update, commit + push is performed.
4. **Project memory file is created if it doesn't exist.** Starts empty on the first conversation.
5. **Does not overwrite, appends.** Added with a date heading.
6. **Sensitive information check.** Information like passwords, tokens, secrets are not written to memory.
