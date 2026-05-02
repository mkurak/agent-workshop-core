# Docs sync (proactive)

## Who runs this

**You (the agent) are responsible.** When you change user-facing behavior, you also sync the matching documentation. docs-sync is preventive — it catches things in the same turn as the change, before the session-end hook becomes the last line of defense.

## What counts as user-facing behavior

Changes the end user of the package or project would notice or need to know:

- New feature (CLI command, skill, rule, agent type, endpoint, UI element)
- Behavior change (default value, return shape, file layout, error message format)
- Breaking change (removed/renamed command, incompatible config, new required env var)
- New dependency or environment requirement
- Security-relevant change (auth flow, permissions, secret handling)
- New lifecycle event or hook

NOT user-facing: internal refactors, added tests, comment or formatting changes, bug fixes that restore advertised behavior without altering the contract.

## The two-phase rule

### Phase 1 — Same turn as the change (primary)

After committing a user-facing change, scan in the same response for documentation that needs to match:

- `README.md` in the repo you touched (often has a command table, feature list, or quick-start section)
- A dedicated doc site (`docs/`, `site/`, `website/`; frameworks like VitePress / MkDocs / Docusaurus)
- CLI `--help` strings and embedded usage text in source code
- `CHANGELOG.md` / `RELEASE-NOTES.md` if the repo keeps them
- Shared project context files (`CLAUDE.md`, `AGENTS.md`, team-level state snapshots)

**Treat docs as part of the change, not a follow-up chore.** Preferred ordering: code → tests → docs, all in the same commit or PR.

### Phase 2 — Session-end backstop

If you genuinely cannot finish Phase 1 — the change is pending user review, the doc requires a large translation pass, or you're uncertain which of several docs is canonical — leave a learning-capture marker with `doc-impact`:

```
<!-- learning
topic: install-refresh-flag
kind: discovery
doc-impact: docs
body: atl v1.0.0+ install is idempotent first-time-only by default; the new --refresh flag explicitly overwrites unmodified copies. Update cli/install doc page + README to call out both the default and the flag.
-->
```

The next session's SessionStart hook will surface the marker via `atl learning-capture --previous-transcripts`; you (or whoever opens the next session) decide which docs to update. **Drafts are presented for review — they are never auto-pushed to public repos.**

## Bilingual / multi-locale docs

If the project ships docs in multiple languages (for example EN canonical + TR mirror at `/tr/`, or any other locale setup), updating only one side is NOT acceptable. Your options:

- Update both sides in the same change. Direct translation is fine for technical content.
- Update one side fully, translate the other as direct prose, flag it with `<!-- TODO: native-speaker review -->`.
- Leave a `doc-impact: docs` marker noting that all locales need the change.

Never silently let mirrors drift. A stale TR page is a visible quality problem on the live site.

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

## Relation to learning-capture

docs-sync is a **specialization** of the [learning-capture](learning-capture.md) protocol. Every change that qualifies here also qualifies as a learning and carries a `doc-impact` value. The two rules cooperate:

- **learning-capture** — general knowledge preservation (journal, wiki)
- **docs-sync** — external-facing documentation discipline

Together they give the user one guarantee: nothing that changes the public surface of a project goes undocumented, and nothing the team learns goes unremembered.

## History

Before this rule existed, documentation updates were ad-hoc — the user had to remember to ask, and the agent usually didn't volunteer. Incidents where a feature shipped with stale README or an out-of-date doc site were common. Pairing an inline "update docs in the same turn" rule with a session-end marker backstop closes the gap on both ends (active and forgetful).
