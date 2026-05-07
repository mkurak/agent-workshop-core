---
name: docs-sync
description: "Drain marker-driven docs queue, run drift-detector against the changed scope, regenerate TR mirrors via parity-checker, present accept/reject drafts. Boring sessions free (pre-flight short-circuit). Invoked from /create-pr Step 4.5 OR manually for catch-up audits. Composite design from the docs-sync-automation brainstorm (Q1+Q3+Q4+Q5+Q7)."
argument-hint: "[--markers | --audit | --since=<tag>] [--force]"
---

# /docs-sync Skill

## Purpose

Keep docs (README + the `agentteamland/docs` site) in sync with the **current shipping state** of every public agentteamland repo. Composite of three drift sources, each handled by its own sub-agent:

1. **Marker-driven** — `<!-- learning doc-impact: ... -->` markers from prior conversations are drained into doc drafts.
2. **Diff-driven** — when invoked from `/create-pr` Step 4.5, the PR's diff is scoped: cli-flag-tarayıcı / versiyon-referansı-tarayıcı / kapsama-denetleyicisi sub-checks run only over the changed surfaces. (The sub-check identifiers are deliberately Turkish — they match the runtime sub-agent names referenced in `repos/cli/internal/docssync/state.go`.)
3. **Comparison-driven** — full drift audit against the docs site: every CLI flag, every version pin, every team / skill / rule entry is verified against the source repos.

For each drift item, an inline draft is presented to the user with accept / reject / edit choices. Accepted drafts commit to the active branch (under `/create-pr`) or to a fresh branch (manual mode).

This skill is the **implementation half** of the docs-sync-automation brainstorm. The companion rule [`core/rules/docs-sync.md`](../../rules/docs-sync.md) defines marker semantics + uniform stub policy; this skill drains and audits.

## Flags

| Flag | Default | Effect |
|---|---|---|
| (no flag) | full sweep | Run BOTH marker drain AND comparison-driven audit (also re-checks coverage baseline). The default for manual catch-up. |
| `--markers` | OFF | Drain pending marker queue ONLY. Skip drift-detector. Useful when you know there are unprocessed markers and want fast turnaround. |
| `--audit` | OFF | Comparison-driven full audit ONLY. Skip marker drain. Useful periodically (release-time, end-of-month). |
| `--since=<tag>` | OFF | Diff vs `<tag>` + full audit. The release-time sweep — typically `--since=v1.1.4` after cutting a new release. |
| `--force` | OFF | Bypass the recent-audit warning (last full audit < 7 days ago) and re-run regardless. |

`--auto` (the `/create-pr`-driven mode) is NOT a user flag — it is set internally when `/create-pr` Step 4.5 invokes this skill with PR-diff context.

## Trigger contexts

Two contexts share the same skill core:

### A) `/create-pr`-driven (default, ship-time)

`/create-pr` Step 4.5 invokes this skill with:

- The PR's transcript path(s) — to extract pending `<!-- learning doc-impact: ... -->` markers.
- The PR's git diff — to scope drift-detector to changed surfaces.

Wisdom + docs + code ship in **one atomic PR**. The review chain (Step 5) sees doc drafts too. See [`/create-pr`](../create-pr/skill.md) Step 4.5 spec.

### B) Manual `/docs-sync` (catch-up)

User invokes from the docs repo or any cached agentteamland repo. No PR context; the skill runs the full sweep against the **current `main`** of every public repo and the live docs site.

## Pre-flight (boring sessions stay free) {#pre-flight}

Before invoking any sub-agent — even before reading transcripts or fetching repos — short-circuit on emptiness:

```
1. Pending markers with doc-impact != none in the input transcripts?
   → Counted via grep; ZERO marker-impact lines means NOTHING to drain.

2. PR diff (when in /create-pr mode) touches a doc-affecting surface?
   → grep over the diff: cmd/atl/commands/, skills/, rules/, agents/, team.json,
     scripts/, docs/site/.
   → No match = NOTHING to compare.

3. Both paths empty?
   → Print "✅ /docs-sync: no doc updates needed" and exit 0.
```

For non-doc-affecting PRs (typo fixes, tests, internal refactors), pre-flight always returns empty. **Zero token cost.**

For manual `--audit` runs, pre-flight is shorter:

- Last full audit < 7 days ago (per state file) → print recent-audit warning + exit unless `--force`.
- New release tag since last audit? → run sweep. No new tag + no flag-relevant changes → "audit baseline still valid, exit".

## Flow

The flow is sequential, with parallel sub-agent invocations within each phase. Each phase has a clear precondition and postcondition.

### Phase 1 — Marker drain (when applicable)

Skip when `--audit` is set OR pre-flight reported zero pending markers.

**1a — Extract pending markers.**

Read input transcripts, grep for `<!-- learning ... -->` blocks, parse YAML fields (`topic`, `kind`, `doc-impact`, `body`). Filter to markers with `doc-impact ∈ {readme, docs, both, breaking}` (drop `none`). Hash each marker by `(topic + body)` and skip ones already in the state file's `processedMarkers` set.

**1b — Group markers by target document.**

- `doc-impact: readme` → README of the marker's source repo.
- `doc-impact: docs` → docs site page that covers the marker's `topic`.
- `doc-impact: both` → README + docs page.
- `doc-impact: breaking` → docs page + a CHANGELOG note (if the repo carries one).

Per [stub-policy-doc-impact-semantics](https://github.com/agentteamland/workspace/blob/main/.atl/wiki/stub-policy-doc-impact-semantics.md): post-Phase-2, `doc-impact: readme` should be RARE (only identity / link / license changes — README is a stub). Most user-facing changes route to `docs`.

**1c — For each grouped target, invoke `doc-rewriter` sub-agent.**

The `doc-rewriter` sub-agent (see [learnings/doc-rewriter.md](learnings/doc-rewriter.md)) reads the target page's current content + the marker bodies + any related markers in the same group, then produces a doc draft (EN canonical).

**1d — Present each draft inline for accept / reject / edit.**

```
📝 Doc draft for site/cli/install.md:
<diff>
Accept? (y/n/edit)
```

Accepted drafts get staged to the working tree. Rejected drafts get logged to journal as "rejected — <reason>". Edits trigger a re-render via `doc-rewriter`.

### Phase 2 — Diff-driven sub-checks (only when `/create-pr`-driven)

Skip in manual mode (no PR diff context).

Spawn `drift-detector` sub-agent (see [learnings/drift-detector.md](learnings/drift-detector.md)) with the PR's diff scope. The sub-agent runs the 3 sub-checks IN PARALLEL (single Task tool call):

| Sub-check | Catches |
|---|---|
| `cli-flag-tarayıcı` | CLI flag drift — `--help` strings vs. doc page tables. |
| `versiyon-referansı-tarayıcı` | Version pins (`v\d+\.\d+\.\d+`, `requires.atl >=`). |
| `kapsama-denetleyicisi` | Coverage gaps — does every shipped skill / rule / agent have a docs page? |

Diff scope means: only touch the parts of the source repos the PR actually changed. If the PR only adds a new skill, only the coverage check runs (fastest path).

### Phase 3 — Comparison-driven audit (only when manual `--audit` or default sweep or `--since=<tag>`)

Skip in `/create-pr` mode (Phase 2 already ran the scoped version).

Same `drift-detector` sub-agent, but with **full repository scope** — every CLI command, every version reference across every doc page, every shipped skill / rule / agent.

For `--since=<tag>` the diff vs. that tag is added to the scope: surfaces touched since the tag get extra scrutiny.

### Phase 4 — Bilingual handling (parity-checker)

For every accepted EN draft from Phase 1 OR every drift item Phase 2/3 produced, the `parity-checker` sub-agent (see [learnings/parity-checker.md](learnings/parity-checker.md)) generates the matching `/tr/` page from the **current EN** and presents it as a paired draft.

EN canonical, TR derived. TR is **fully regenerated** from current EN — never patched at the diff site. Translation quality bar (no code-switching, native Turkish prose, technical terms English only) is enforced per the rules in [learnings/parity-checker.md](learnings/parity-checker.md).

If the parity-checker reports a structural mismatch on any TR page (Phase 3 only — same-pass translations are clean by construction), it routes that page through Mode 2 regeneration as part of the same audit.

### Phase 5 — Drafts → branch / commit (mode-dependent)

**`/create-pr`-driven:** accepted drafts already staged on the active feature branch. Commit them with the message: `docs: <type> drift-fix from <PR title>` (one commit per drift cluster). The drafts ride along in the same PR.

**Manual mode:** create or reuse a `docs/sync-<YYYY-MM-DD>` branch in `agentteamland/docs` (and / or peer repos when README touches happened). Stage drafts, commit per cluster, push, open a PR per repo. Each PR follows team-repo-maintenance discipline.

### Phase 6 — State file write

Update `~/.claude/state/docs-sync-state.json` with:

- New `processedMarkers` hashes (FIFO-capped at 5000).
- `lastFullAudit` timestamp (when comparison-driven phase ran).
- `lastMarkerDrain` timestamp (when marker phase ran).
- `coverageBaseline` (latest skills / rules / agents inventory if Phase 2/3 ran).
- `lastReleaseTagSeen` (latest `cli` tag observed).

This write is the closing bracket. Subsequent invocations short-circuit at pre-flight when nothing changed.

State file write is delegated to a future CLI command: `atl docs-sync --commit-from-state` (see Phase 3 implementation plan, item 6). Until that lands, this skill writes the state file directly with atomic semantics (`tmp + rename`).

### Phase 7 — Telemetry summary

Single line at the end:

```
✅ /docs-sync done (3.2s, 0 sub-agents invoked, 0 drift items)
✅ /docs-sync done (47s, 3 sub-agents invoked, 5 drift items, 2 drafts accepted)
```

Boring sessions are obvious from the elapsed time + zero counts. Productive sessions surface the work done.

## State file format

`~/.claude/state/docs-sync-state.json`:

```json
{
  "projects": {
    "<slug>": {
      "lastFullAudit": "2026-05-04T15:00:00Z",
      "lastMarkerDrain": "2026-05-04T15:23:00Z",
      "processedMarkers": ["hash1", "hash2", ...],
      "coverageBaseline": {
        "skills": ["save-learnings", "wiki", ...],
        "rules": [...],
        "agents": [...]
      },
      "lastReleaseTagSeen": "v1.1.4"
    }
  }
}
```

`<slug>` follows the same kebab-cased cwd convention as `~/.claude/state/learning-capture-state.json`.

## Sub-agent contracts

Each sub-agent has its own procedure document under `learnings/`:

- **[`drift-detector`](learnings/drift-detector.md)** — runs the 3 drift sub-checks. Output: list of drift items with file path + line number + suggested fix.
- **[`parity-checker`](learnings/parity-checker.md)** — bilingual mirror generator (EN → TR full regeneration) + translation quality bar enforcement.
- **[`doc-rewriter`](learnings/doc-rewriter.md)** — applies marker bodies / drift items to doc pages. Produces unified diffs as drafts.

All three are invoked via the `Task` tool with `subagent_type: general-purpose`. Parallel invocations (single Task batch) reduce wall-clock cost ~3× when all three trigger.

## Important rules

1. **Boring sessions stay free.** Pre-flight ALWAYS runs first; zero markers + zero diff matches → exit immediately, no sub-agent calls.
2. **EN canonical, TR derived.** Never author TR independently; always regenerate from current EN. Structural drift becomes impossible by construction.
3. **Inline accept / reject loop, no auto-push.** Every draft surfaces to the user before staging. Even in `/create-pr` mode — drafts ride along in the PR but they go through the user's accept gate first.
4. **Hash-dedup processed markers.** A marker that was drained on a prior run does not re-drain. State file is the source of truth.
5. **Recent-audit warning.** Manual `--audit` < 7 days since last full audit → confirm before running. `--force` overrides.
6. **Translation quality bar (parity-checker).** No code-switching in Turkish prose. File paths, command names, code samples, identifier names stay English; verbs, nouns, connecting prose translated to native Turkish. Full ruleset in [learnings/parity-checker.md](learnings/parity-checker.md).
7. **Atomic state file write.** Use `tmp + rename` semantics; partial writes never overwrite a valid state file.
8. **Never auto-push to public repos.** Drafts always pass through the user's accept gate. PR creation goes through `/create-pr` (which has its own merge discipline — Claude never merges).

## Related

- [`/create-pr`](../create-pr/skill.md) — invokes this skill at Step 4.5 (the ship-time path).
- [`/save-learnings`](../save-learnings/skill.md) — produces the `<!-- learning -->` markers this skill drains.
- [`docs-sync` rule](../../rules/docs-sync.md) — marker semantics + uniform stub policy.
- [`learning-capture` rule](../../rules/learning-capture.md) — `doc-impact` field definition (input to this skill).
- [docs-sync-automation brainstorm](https://github.com/agentteamland/workspace/blob/main/.atl/brain-storms/docs-sync-automation.md) — the design that produced this skill (Q1+Q2+Q3+Q4+Q5+Q7 settled).

## History

- **2026-05-02**: docs-sync-automation brainstorm started. Phase 1 (clean baseline) shipped 11 PRs across 16 repos.
- **2026-05-03**: Phase 2 stub conversion shipped (every public agentteamland README is now a stub).
- **2026-05-04**: Phase 3 architecture closed (Q1-Q7 all settled); Phase 2.D approved + executed (41 TR pages regenerated across 4 sprints).
- **2026-05-04+**: Phase 3 implementation begins. This skill is item 1+2 of the Phase 3 plan. Subsequent items: rule updates (3+5), `/create-pr` Step 4.5 (4), `atl docs-sync --commit-from-state` CLI command (6).
