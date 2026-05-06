# Docs sync (proactive)

## Who runs this

**You (the agent) and the [`/docs-sync` skill](../skills/docs-sync/skill.md) cooperate.** When user-facing behavior changes, you mark the change with a `<!-- learning ... doc-impact: ... -->` marker (or update docs in the same turn for trivial cases). The `/docs-sync` skill — invoked from `/create-pr` Step 4.5 OR manually — drains the marker queue and runs a comparison-driven audit, presenting accept / reject drafts for every drift item.

This rule defines the *contract* (what counts as user-facing, what `doc-impact` value to use, when same-turn updates suffice). The `/docs-sync` skill defines the *mechanism* (how markers and drift items become drafts).

## What counts as user-facing behavior

Changes the end user of the package or project would notice or need to know:

- New feature (CLI command, skill, rule, agent type, endpoint, UI element)
- Behavior change (default value, return shape, file layout, error message format)
- Breaking change (removed/renamed command, incompatible config, new required env var)
- New dependency or environment requirement
- Security-relevant change (auth flow, permissions, secret handling)
- New lifecycle event or hook

NOT user-facing: internal refactors, added tests, comment or formatting changes, bug fixes that restore advertised behavior without altering the contract.

## The two paths

### Path A — Mark with `doc-impact` (primary, post-Phase-3)

Drop a learning-capture marker with the appropriate `doc-impact` value in the same response as the change:

```
<!-- learning
topic: install-refresh-flag
kind: discovery
doc-impact: docs
body: atl v1.0.0+ install is idempotent first-time-only by default; the new --refresh flag explicitly overwrites unmodified copies. Update cli/install doc page to call out both the default and the flag.
-->
```

The `/docs-sync` skill consumes this marker — automatically when invoked from `/create-pr` Step 4.5 (the ship-time path), or manually when the user runs `/docs-sync` for a periodic catch-up. The skill produces an accept / reject draft against the right doc page.

This is the **default path** for non-trivial changes. Marker discipline is cheap (~50 tokens), and the skill handles the heavy lifting (target page selection, voice matching, bilingual mirror generation).

### Path B — Update docs in the same turn (only when trivial AND obvious)

For changes where the doc update is one line, the target is obvious, and the page already lives nearby in the diff (e.g., a code comment, embedded `--help` text, a CLI flag table you just edited two lines above), update the doc inline rather than queue a marker.

Examples that fit Path B:
- Changing a default value AND its single-line documentation in the same source file.
- Renaming a flag AND its `--help` description.
- Updating a value in a YAML schema AND the matching reference doc table when both are in the same PR diff.

Examples that fit Path A (NOT B):
- New CLI command (target page selection, anchor parity, bilingual mirror needed).
- Behavior change spanning multiple doc surfaces.
- Anything that touches the docs site under `repos/docs/site/` (the bilingual mirror is non-trivial — the parity-checker sub-agent handles it).

When in doubt, prefer Path A. The `/docs-sync` skill is built to be cheap on boring sessions (pre-flight short-circuits when no markers + no diff matches).

**Drafts are always presented for review — they are never auto-pushed to public repos.** PR creation goes through `/create-pr`, which inherits the never-merge discipline from team-repo-maintenance.

## Bilingual / multi-locale docs

The `agentteamland/docs` site ships EN canonical + TR mirror under `/tr/`. The Phase 3 policy (Q5 of the docs-sync-automation brainstorm) is now formalized:

1. **EN is canonical, TR is derived.** TR is never authored independently — it is generated from current EN.
2. **When EN changes, TR is fully regenerated from current EN** (not patched at the diff site). Structural drift becomes impossible by construction.
3. **Translation quality bar applies** — no code-switching in Turkish prose; technical identifiers stay English; verbs / nouns / connecting prose translated to native Turkish. Full ruleset in [`/docs-sync` skill's parity-checker procedure](../skills/docs-sync/learnings/parity-checker.md).

The `parity-checker` sub-agent (invoked by `/docs-sync` Phase 4) handles the regeneration. As an agent dropping markers, your job is just to drop the `doc-impact` marker for the EN change — the bilingual mirror generation happens automatically downstream.

Never silently let mirrors drift. A stale TR page is a visible quality problem on the live site, and the parity-checker's structural audit (`/docs-sync --audit`) will surface it.

## Embedded help text and machine-generated docs

Some docs live inside code (CLI `--help` strings, API OpenAPI specs, auto-generated reference pages). When you change a command, these count as docs too. Update them in the same turn as the behavior change — don't let `--help` say something the binary doesn't do.

## Uncertainty is fine — silence isn't

If you change something and honestly don't know whether it's user-facing, mark it explicitly:

```
<!-- learning
topic: internal-cache
kind: pattern
doc-impact: none
body: Added internal LRU cache in updater package. Not exposed via CLI — no docs impact.
-->
```

An explicit `doc-impact: none` creates an audit trail showing the decision was made. Silent skipping leaves no trail — weeks later nobody knows if docs were considered or forgotten.

## Uniform stub policy (post-Phase-2)

After the docs-sync-automation brainstorm's Phase 2 stub conversion (shipped 2026-05-03), every public agentteamland repo's `README.md` is a **redirect-style stub** — short identity statement + link to the canonical docs page. Drift-prone content lives at exactly one URL: `https://agentteamland.github.io/docs/`.

This changes how `doc-impact` should be set:

- `doc-impact: readme` — should be **rare**. Use it only for identity / link / license changes (e.g., updating the stub's pointer URL when a doc page moves, fixing a broken link in a stub, MIT license year bump). README is documentation of the stub policy itself.
- `doc-impact: docs` — should be the **common case**. Every user-facing change goes here. The doc site is where evolving content lives.
- `doc-impact: both` — rare but legitimate when an actual stub touch-up *and* a docs-site change ride together (e.g., renaming a doc page that the stub points at).
- `doc-impact: breaking` — for changes that require a CHANGELOG entry alongside docs site updates (deprecations, removed commands, incompatible config changes).
- `doc-impact: none` — internal change, no user-facing surface affected.

The `.github` org-profile repo is the single exception to the uniform stub policy — its README is the org's public landing card and carries content. See [stub-policy-doc-impact-semantics wiki](https://github.com/agentteamland/workspace/blob/main/.atl/wiki/stub-policy-doc-impact-semantics.md) for the historical context that produced these semantics.

## Relation to learning-capture

docs-sync is a **specialization** of the [learning-capture](learning-capture.md) protocol. Every change that qualifies here also qualifies as a learning and carries a `doc-impact` value. The two rules cooperate at three layers:

- **learning-capture** — general knowledge preservation (journal, wiki, agent children, skill learnings).
- **docs-sync** (this rule) — external-facing documentation discipline (README + docs site).
- **`/docs-sync` skill** — the automation that drains markers + audits drift + regenerates bilingual mirrors.

Together they give the user one guarantee: nothing that changes the public surface of a project goes undocumented, and nothing the team learns goes unremembered.

## History

Before this rule existed, documentation updates were ad-hoc — the user had to remember to ask, and the agent usually didn't volunteer. Incidents where a feature shipped with stale README or an out-of-date doc site were common. Pairing an inline "update docs in the same turn" rule with a session-end marker backstop closes the gap on both ends (active and forgetful).

The Phase 3 update of this rule (2026-05-04) coincided with the `/docs-sync` skill landing in `core@1.11.0`. Pre-Phase-3 the rule asked the agent to scan for relevant docs in the same response — useful but slow and inconsistent. Post-Phase-3 the agent's job is reduced to dropping a `doc-impact` marker; the `/docs-sync` skill handles target page selection, draft authoring, and bilingual mirror generation when invoked from `/create-pr` Step 4.5 OR manually for a periodic catch-up.

The uniform stub policy section codifies the post-Phase-2 reality: every public agentteamland repo's README is a stub, so `doc-impact: readme` should be rare and `doc-impact: docs` should be the common case. This rule is the canonical place that documents that semantic shift.
