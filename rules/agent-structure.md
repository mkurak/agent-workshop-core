# Agent Configuration Rules

## Children Pattern (Mandatory)

Every agent is organized in the following structure:

```
~/.claude/agents/{agent-name}/
├── agent.md              ← Identity, area of responsibility, core principles (short, embedded)
└── children/             ← Detailed information, patterns, strategies (each topic in a separate file)
    ├── topic-1.md
    ├── topic-2.md
    └── ...
```

### Rules

1. **agent.md stays short.** Only: identity, area of responsibility (positive list), core principles (unchanging, short bullet points), "read children/" instruction.
2. **Everything detailed goes under children/.** Strategies, patterns, workflows, conventions -- each in a separate .md file.
3. **New topic = new file.** Without touching agent.md, add a .md file under children/. The agent auto-discovers it via the "read all .md files under children/" instruction.
4. **Update = single file.** To update a topic, only the relevant children file is touched.
5. **Monolithic agent files are prohibited.** Piling all information into a single .md is prohibited -- it becomes unmanageable.
6. **This pattern applies to all agents.** API, Socket, Worker, Flutter, React, Mail, Log, Infra -- all follow the same structure.

## Knowledge Base Format (Mandatory)

In agent.md's Knowledge Base section, every children topic MUST be listed with:
1. **A heading** with the topic name
2. **A 2-3 line summary** explaining what the topic covers and key points
3. **A detail link** pointing to the children file: `→ [Details](children/{topic}.md)`

This format ensures:
- The agent understands each topic from the summary alone for quick decisions
- The agent reads the full children file only when the task requires that specific knowledge
- New topics added to children/ are also reflected in agent.md's Knowledge Base section

### Example Format

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
