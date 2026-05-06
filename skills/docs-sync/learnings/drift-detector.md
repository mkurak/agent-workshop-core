---
knowledge-base-summary: "Three sub-checks run in parallel under one Task call: cli-flag-tarayıcı (CLI flag drift), versiyon-referansı-tarayıcı (version pin staleness), kapsama-denetleyicisi (skill/rule/agent → doc page coverage). Output is a structured drift list — file path, line, current value, current truth, suggested fix. Q4 of the docs-sync-automation brainstorm formalized the 3-check shape."
---

# drift-detector sub-agent

Procedure document for the `drift-detector` sub-agent invoked by `/docs-sync` Phase 2 (diff-driven, scoped) and Phase 3 (comparison-driven, full).

## Invocation contract

`/docs-sync` invokes this sub-agent via the `Task` tool with:

```
subagent_type: general-purpose
prompt: "<scope description> + <which sub-checks to run> + <expected output format>"
```

The three sub-checks ALWAYS run in parallel — single `Task` tool call, single response. Parallelism cuts wall-clock cost by roughly 3×.

The sub-agent's job:

1. Read the source-of-truth surface (CLI binary `--help`, source file, team.json, etc.).
2. Read the docs-side claim (page table row, version pin, missing-page check).
3. Compare; produce a structured drift item per mismatch.

The sub-agent does NOT write fixes. It reports drift; `doc-rewriter` (a separate sub-agent) authors the actual draft. This split keeps each sub-agent's responsibility narrow.

## Output format

Every drift item is one record:

```yaml
- subcheck: cli-flag-tarayıcı | versiyon-referansı-tarayıcı | kapsama-denetleyicisi
  severity: high | medium | low
  source-truth:
    location: path/to/source-file.go:NN  (or "binary --help output")
    value: "<the current truth — flag list, current shipping version, etc.>"
  doc-claim:
    location: site/cli/install.md:NN  (or "missing entirely")
    value: "<what the doc currently says — or 'no entry'>"
  suggested-fix: "<one-line description of what should change>"
  related-markers: []  # marker hashes IF this drift is also covered by a pending marker (avoids double-fix)
```

Markdown table summary at the top is fine; the structured records are what `doc-rewriter` consumes.

## Sub-check 1: `cli-flag-tarayıcı`

**Catches Mode 1 of the original 3 failure modes** (speculative writing — docs claim a flag the binary does not have).

**Source of truth**: `atl <command> --help` output for every shipped command, OR source parsing of `cmd/atl/commands/*.go` (cobra command definitions). The CLI is checked into the workspace at `repos/cli/`; running the binary requires a built atl, but parsing source is always available.

**Doc-side claim**: each `/cli/<command>.md` page's flag table.

**Comparison rules**:

- Every flag in `--help` output should appear in the doc page's flag table. Missing flag in docs = drift.
- Every flag in the doc page's flag table should exist in the binary. Doc claims a flag the binary does NOT have = drift (Mode 1 failure).
- Default values, descriptions, and short aliases should match.

**False-positive guard**: skip `--help` and `--version` (universal, not always tabled). Skip flags marked `hidden` in cobra metadata.

**Severity**:

- `high` — doc claims a flag that does not exist (users will hit "unknown flag" errors).
- `medium` — flag exists in binary but missing from doc table (discoverability gap).
- `low` — description / default value mismatch (cosmetic).

## Sub-check 2: `versiyon-referansı-tarayıcı`

**Catches Mode 2 of the original 3 failure modes** (time-frozen identity — docs reference a version that is no longer current).

**Source of truth**:

- Current shipping `atl` version: latest tag in `repos/cli` (or the GitHub Releases API).
- Current shipping team versions: each team's `team.json` `version` field.
- `requires.atl` minimum: each team's `team.json` `requires.atl` field.

**Doc-side claim**: every regex match for version patterns across `repos/docs/site/**/*.md`:

- `v\d+\.\d+\.\d+` (literal version refs).
- `(>=|=|\^|~)\s*\d+\.\d+\.\d+` (constraint refs).
- "Requires atl" + version (free-text refs).
- `Latest version:` / `Son sürüm:` headers in team pages.

**Comparison rules**:

- Doc page references `atl v1.1.0` but current shipping is `v1.1.4` → drift.
- Doc page references `software-project-team@1.2.0` but `team.json` says `1.2.1` → drift.
- Doc page says `requires.atl >= 0.2.0` but `team.json` says `>=1.1.3` → drift.

**False-positive guard**: history sections, changelog entries, and "v0.1.0 (2026-04-17)" style historical references are NOT drift — they are intentional time-stamps. Detect via heading context (`## History`, `## Tarihçe`, `## Yayımlananlar`, `## What shipped`) and skip refs inside.

**Severity**:

- `high` — version pin in install instructions / quickstart pages stale enough that copy-paste fails.
- `medium` — team page header version stale.
- `low` — narrative reference stale but copy-paste still works.

## Sub-check 3: `kapsama-denetleyicisi`

**Catches Mode 3 of the original 3 failure modes** (coverage gaps — a shipped skill / rule / agent has no docs page, OR a docs page describes something that no longer ships).

**Source of truth**:

- Walk every team repo: `repos/<team>/skills/`, `repos/<team>/rules/`, `repos/<team>/agents/` directories.
- Walk core: `repos/core/skills/`, `repos/core/rules/`.
- Build the inventory from `team.json` `skills` / `rules` / `agents` arrays (the canonical declared list).

**Doc-side**: pages under `repos/docs/site/skills/`, `repos/docs/site/cli/` (commands), `repos/docs/site/teams/`, etc.

**Comparison rules — bidirectional**:

- **In source not in docs**: every shipped item should have a docs page. Missing page = drift (severity high if user-facing skill / command, medium for internal rule).
- **In docs not in source**: a page describing a deprecated / removed item is drift (severity high — false promise to users).

**Incremental optimization**: read the previous coverage baseline from state file. If the inventory has not changed since `lastFullAudit`, skip this sub-check entirely.

**False-positive guard**: pages explicitly marked `archived` (e.g., the `create-project` page after the 2026-05-04 archive) are excluded from "in docs not in source" drift — the archive is the documentation.

**Severity**:

- `high` — user-facing skill / command without a doc page.
- `medium` — rule without a doc page.
- `low` — internal helper without a doc page.

## When the sub-agent should NOT report drift

False-positive discipline is critical — the [docs-audit-false-positive-rate wiki page](https://github.com/agentteamland/workspace/blob/main/.atl/wiki/docs-audit-false-positive-rate.md) recorded ~40% hallucinated findings on the 2026-05-02 sweep. Hard rules to suppress noise:

1. **Always grep the source file before reporting a missing claim.** If the sub-agent thinks "the docs say X but X is missing", it MUST quote the source file's content via grep before claiming the gap.
2. **Quote the doc claim verbatim.** Drift items must include the exact source text — no paraphrasing. This catches hallucinated "the docs say Y" claims at review time.
3. **Skip narrative drift.** "The docs say `flag` is the recommended approach but the source uses `option`" is not drift unless the source genuinely contradicts the doc claim.

## Performance budget

- Single sub-agent invocation: target < 10 seconds wall-clock for full-scope audit.
- Source-of-truth file reads: cap at 200 file reads per sub-check (split scope if exceeded; report partial scope if hit).
- No network calls inside the sub-agent — all data is local (workspace clones every peer repo).

## Related

- [parity-checker.md](parity-checker.md) — bilingual mirror generator. Drift-detector reports drift; parity-checker handles the TR side.
- [doc-rewriter.md](doc-rewriter.md) — converts drift items + marker bodies into actual doc drafts.
- [docs-audit-false-positive-rate](https://github.com/agentteamland/workspace/blob/main/.atl/wiki/docs-audit-false-positive-rate.md) — historical incident that drove the false-positive guards in this procedure.
