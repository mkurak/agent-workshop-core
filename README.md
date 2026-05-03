# 🧬 AgentTeamLand Core

> Core infrastructure for every AgentTeamLand team. Ships the journal + wiki knowledge system, the auto-trigger learning-capture loop, the agent + skill structure rules, the team-repo governance discipline, and the four global skills every project gets for free.

`core` is auto-installed as a dependency by every team in the registry. It defines the conventions every team relies on (`children/` + `learnings/` patterns, `knowledge-base-summary` frontmatter, the marker block contracts in `CLAUDE.md`) and ships the platform-wide rules that load into every Claude Code session.

`core@1.10.0` requires `atl >= 1.1.3` for the `atl learning-capture --commit-from-transcripts` mode that closes the long-session marker re-report bug.

## 📚 Documentation

Full docs live at **[agentteamland.github.io/docs](https://agentteamland.github.io/docs/)**.

### Global skills shipped by core

- [`/save-learnings`](https://agentteamland.github.io/docs/skills/save-learnings) — persist conversation learnings to journal + wiki + agent children + skill learnings; auto-rebuild Knowledge Base sections from frontmatter
- [`/wiki`](https://agentteamland.github.io/docs/skills/wiki) — living project knowledge base (init / ingest / query / lint)
- [`/create-pr`](https://agentteamland.github.io/docs/skills/create-pr) — auto-named branch + save-learnings + AI review chain + PR (optional auto-merge)
- [`/create-code-diagram`](https://agentteamland.github.io/docs/skills/create-code-diagram) — full-project Mermaid class diagram

### Rules shipped by core (auto-loaded into every session)

- [Karpathy guidelines](https://agentteamland.github.io/docs/guide/karpathy-guidelines) — four coding principles (Think Before Coding, Simplicity First, Surgical Changes, Goal-Driven Execution)
- [Team-repo maintenance](https://agentteamland.github.io/docs/authoring/team-repo-maintenance) — five-step PR discipline + NEVER-MERGE constraint + auto-merge exception
- [Knowledge system](https://agentteamland.github.io/docs/guide/knowledge-system) — the two-layer journal + wiki model
- [Children + learnings](https://agentteamland.github.io/docs/guide/children-and-learnings) — agent / skill structure pattern + auto-rebuild contract
- [Learning marker lifecycle](https://agentteamland.github.io/docs/guide/learning-marker-lifecycle) — `<!-- learning -->` markers, SessionStart hook, end-to-end flow
- [Claude Code conventions](https://agentteamland.github.io/docs/guide/claude-code-conventions) — `<!-- wiki:index -->` / `<!-- brainstorm:active -->` / `<!-- pending-implementation -->` marker blocks

### Operations + concepts

- [Governance](https://agentteamland.github.io/docs/guide/governance) — branch protection across 12 production repos + the team-repo-maintenance rule pair
- [Concepts](https://agentteamland.github.io/docs/guide/concepts) — team / agent / skill / rule / registry / inheritance mental model

## License

MIT.
