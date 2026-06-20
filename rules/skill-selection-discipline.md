# Skill selection discipline

When responding to a user prompt, the agent must give explicit consideration to skill selection — particularly when the project has multiple teams installed.

## Why this rule exists

ATL projects can have multiple teams installed simultaneously (e.g., `software-project-team` for codebase work + `personal-advisory-team` for life-advice work, both rooted in the same `.atl/`). Their skill sets do NOT overlap, but their *trigger surfaces* can — a user prompt may sound applicable to either, neither, or both. Selecting the wrong skill produces bad output (a software answer to a personal question) or missed automation (a turn that should have invoked a lens-skill but didn't).

Auto-activation mechanisms (per-team watch hooks, central dispatchers, lazy-load arbitration) were considered and rejected on cost × determinism grounds (rejection brainstorm: `auto-team-activation.md`, 2026-05-07). Skill selection therefore remains the agent's responsibility. This rule codifies the diligence required to make that responsibility reliable.

## What the agent must do

When the prompt could plausibly invoke a skill — i.e., it's not a trivial conversational turn — work through these in order:

1. **Survey the union of installed skills.** Don't default to the most-recently-used team. Yesterday's session was a software refactor; today's prompt is "I have a meeting with Alex tomorrow — what should I ask?" — the right skill probably lives in a different team.
2. **Match prompt intent to each candidate skill's `description` frontmatter.** That field is what the skill exists for. Don't infer from the skill's name alone — `/save-learnings` and `/wiki ingest` look superficially similar but cover different scopes.
3. **When more than one skill could apply, name them and disambiguate.** Either pick the strongest match with a one-line justification ("This is a personal matter — I'll start with /journal") OR ask one clarifying question. Don't pick silently.
4. **When no installed skill applies, say so explicitly.** Silent non-invocation is indistinguishable from oversight; an explicit "no skill applies here, responding directly" closes the loop.

## Failure modes to watch

- **Recency bias** — defaulting to the team used in the previous turn, irrespective of the current prompt's domain.
- **Team bias** — defaulting to the team installed first, or the team the user works on most frequently.
- **Name-based skill confusion** — picking by skill name without reading `description`. Frontmatter exists for this reason.
- **Silent skip** — answering directly when an installed skill could have done the job. If you skip, name the skill you considered and state why it didn't fit.

## When this rule does NOT apply

- **Trivial turns** (greetings, status questions, factual lookups that don't trigger any workflow).
- **User explicitly named a skill** (e.g., "/save-learnings" — invoke as named, no second-guessing).
- **No teams installed** — only built-in Claude Code commands are in scope.

## How this rule interacts with other rules

- **Karpathy `Think Before Coding`** — this rule's step 1 (survey) is a domain-specific application of "state assumptions explicitly."
- **Karpathy `Goal-Driven Execution`** — when uncertain which skill applies, the disambiguation question (step 3) is the verification step before action.
- **`learning-capture`** — when a skill selection decision is non-obvious and gets corrected, that's worth a `<!-- learning -->` marker (kind: `decision`, doc-impact: `none`) so future sessions don't repeat the miss.

## History

Added 2026-05-07 alongside the rejection of `auto-team-activation` (workspace brainstorm, same date). Mesut surfaced the multi-team coordination problem during a session originally targeting `profile-memory-layer`; auto-activation alternatives were surveyed; cost × determinism tradeoff drove the rejection; this rule is the explicit replacement for the missing automation. Without it, the rejection would have left the discipline gap open.
