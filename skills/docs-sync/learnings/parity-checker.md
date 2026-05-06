---
knowledge-base-summary: "Bilingual mirror generator. EN canonical, TR derived — TR is fully regenerated from current EN, never patched at the diff site. Three modes: same-pass translation (during EN drafting), TR regeneration (after structural drift), structural parity audit (full sweep). Translation quality bar: native Turkish prose, no code-switching, technical terms English only. Q5 of the docs-sync-automation brainstorm formalized this."
---

# parity-checker sub-agent

Procedure document for the `parity-checker` sub-agent invoked by `/docs-sync` Phase 4 (bilingual handling).

## Invocation contract

`/docs-sync` calls this sub-agent via `Task` (`subagent_type: general-purpose`) with:

```
mode: same-pass | regeneration | structural-audit
en-source: <path to the EN page being mirrored or audited>
tr-target: <path to the TR mirror>
quality-bar: <inline quality rules — see below>
```

The sub-agent's job:

1. Read the EN canonical content.
2. (For audit / regeneration) read the current TR content for structural comparison.
3. Generate (mode 1+2) or audit (mode 3) the TR mirror.
4. Output a unified diff for `doc-rewriter` to stage as a draft.

## Three modes

### Mode 1 — Same-pass translation (most common)

Used when `doc-rewriter` has just produced a fresh EN draft and the TR mirror needs to be generated alongside.

**Input**: the freshly-drafted EN content (from `doc-rewriter`).
**Output**: TR mirror, generated from scratch, translation quality bar applied.
**Result presentation**: paired EN + TR shown to user as a single accept / reject decision (one decision, two languages).

Same-pass translation is **clean by construction** — no structural drift can arise because both pages are minted at the same time.

### Mode 2 — TR regeneration (drift detected)

Used when Phase 3 (comparison-driven) detects structural drift between an existing EN page and its TR mirror.

**Input**: current EN page (canonical).
**Output**: TR mirror, regenerated from scratch from current EN.

**Critical rule**: do NOT patch the existing TR. Read the current EN. Generate fresh TR. Replace the entire TR file.

Patching produces compounding drift over time — old translation choices linger, structural mismatches accumulate, code-switching debt builds. Full regeneration is the policy.

### Mode 3 — Structural parity audit

Used during `/docs-sync --audit`. Compares every EN page against its TR mirror and produces a list of pages needing Mode 2 regeneration.

**Drift signals** (any of these triggers Mode 2 routing):

- Section count parity: TR has fewer / more `## ` headings than EN.
- Heading levels: a TR page uses `## ` where the corresponding EN page uses `### `.
- Anchor IDs: a TR page is missing an explicit `{#anchor}` that the EN page has — and that anchor is linked from elsewhere.
- Link count: TR has fewer `](path)` references than EN.
- Code samples: code fences differ (not always 1:1 — translations may explain code in prose, but bash / yaml / json samples should be identical).
- Prose code-switching: any of the forbidden patterns from the quality bar (below) detected in TR prose.

**Output**: list of page pairs needing regeneration, with the failing signal noted. `/docs-sync` Phase 4 routes each one through Mode 2.

## Translation quality bar — formalized rules

These rules MUST be applied when the sub-agent generates Turkish output. They are also the rules a Mode 3 audit checks for. Source: Q5 of the docs-sync-automation brainstorm (workspace, 2026-05-04).

### Stays English (technical terms, identifiers)

- File paths: `/cli/install.md`, `~/.claude/repos/agentteamland/`.
- Command names: `/docs-sync`, `/create-pr`, `atl install`, `gh pr create`.
- Code samples (the entire fenced block, including comments inside the block).
- Class / function / parameter names.
- Tool / library names: GitHub, VitePress, Goreleaser, Homebrew, Scoop, npm, ajv-cli, Tailwind, Mermaid.
- Sub-agent names that are class-like identifiers: `parity-checker`, `drift-detector`, `doc-rewriter`.
- AgentTeamLand kavram identifier'ları when used as identifiers: `software-project-team`, `agent-structure`, `learning-capture`.
- JSON keys: `schemaVersion`, `extends`, `excludes`, `requires.atl`, `knowledge-base-summary`.

### Must be translated to Turkish (native, idiomatic prose)

- **Connecting prose, narrative, descriptions** — the actual sentences gluing technical terms together.
- **Verbs**:
  - process → işlemek
  - consume → tüketmek
  - apply → uygulamak
  - implement → uygulamak / hayata geçirmek
  - review → incelemek
  - audit → denetlemek
  - validate → doğrulamak
  - override → bastırmak
  - fork → çatallamak
  - scaffold → iskeleyle kurmak
  - ingest → almak
  - lint → denetlemek
  - persist → kalıcılaştırmak
- **Adjectives**:
  - stale → eski / bayat
  - default → varsayılan
  - scoped → kapsamı belirli
  - idempotent → idempotent (kabul edilebilir; "değişmez" da olur)
- **Nouns**:
  - approach → yaklaşım
  - scope → kapsam
  - context → bağlam
  - draft → taslak
  - input → girdi
  - output → çıktı
  - trigger → tetikleme
  - patch → yama
  - framework → çerçeve
  - queue → kuyruk
  - policy → politika
  - discipline → disiplin
  - task → görev
  - registry → kayıt defteri
  - cache → önbellek
  - source-of-truth → kaynak doğruluk
  - manifest → manifesto
  - governance → yönetişim
  - scaffolder → iskele
  - lag → gecikme

### Code-switching forbidden (anti-pattern examples)

These patterns MUST be rewritten — they are quality bar violations:

- ❌ "işi şimdi break etmek ister misin?" → ✅ "işi şimdi durdurmak ister misin?"
- ❌ "process etmek" → ✅ "işlemek"
- ❌ "review yapmak" → ✅ "incelemek"
- ❌ "implement etmek" → ✅ "uygulamak / hayata geçirmek"
- ❌ "scope" (Türkçe nesir içinde) → ✅ "kapsam"
- ❌ "context" (Türkçe nesir içinde) → ✅ "bağlam"
- ❌ "fail oldu" → ✅ "başarısız oldu"
- ❌ "push'lamak" tekil olarak → ✅ "push'lamak" (Git terminolojisi olarak kabul edilebilir, ama "yayımlamak" / "göndermek" daha akıcı)
- ❌ "auto-update" → ✅ "kendiliğinden güncelleme"
- ❌ "merge etmek" → ✅ "birleştirmek" (anlatı içinde; `merge` Git komut adıysa tabii ki kalır)

### Edge cases — terms accepted in Turkish dev jargon

These ARE acceptable when used as Git / dev terminology, even though they are English:

- `merge`, `commit`, `branch`, `push`, `pull`, `fork`, `PR` (Pull Request) — universally used in Turkish dev contexts.
- `stack` (technology stack) — kabul edilebilir, ama "yığın" da akıcı.

When unsure, prefer the Turkish alternative. The bar is "would a Turkish-speaking dev reader find this awkward?" — if the answer is yes, retranslate.

## Anchor link convention (TR/EN parity)

When the EN canonical page has an explicit `{#anchor}` that other pages link to, the TR mirror MUST carry the same `{#anchor}` ID — even though the Turkish heading text differs.

Example:

```markdown
EN: ## What it updates {#what-it-updates}
TR: ## Neyi günceller {#what-it-updates}
```

The Turkish heading text is translated; the anchor ID is preserved verbatim. This keeps cross-page fragment links (`/tr/cli/update#what-it-updates`) working uniformly across both locales.

VitePress's auto-slug for non-ASCII Turkish characters is not always safe (accented characters can produce surprising slugs). Explicit anchor on every linked TR heading sidesteps the auto-slug risk entirely.

## Output format

Unified diff per page, as plain text. Two diffs when running same-pass translation (one EN, one TR). One diff for Mode 2 regeneration (TR only). Mode 3 audit produces a list of file pairs + signals, no diffs (those come from the Mode 2 invocations the audit triggers).

## Performance budget

- Same-pass translation (Mode 1): cost dominated by `doc-rewriter`; parity-checker adds ~30-50% latency.
- TR regeneration (Mode 2): per-page cost roughly equal to drafting an EN page from scratch.
- Structural audit (Mode 3): cap at full `/tr/` directory walk + comparison; target < 30 seconds wall-clock.

## Related

- [drift-detector.md](drift-detector.md) — sister sub-agent. Runs first; produces the EN drift list parity-checker mirrors.
- [doc-rewriter.md](doc-rewriter.md) — produces the EN drafts. Same-pass translation hands off from doc-rewriter directly.
- [Translation quality bar — Q5 of brainstorm](https://github.com/agentteamland/workspace/blob/main/.atl/brain-storms/docs-sync-automation.md#q5--bilingual-handling-tr-derived-from-en-full-regeneration-on-change) — original specification.
- [TR anchor link convention wiki](https://github.com/agentteamland/workspace/blob/main/.atl/wiki/tr-anchor-link-convention.md) — historical context on why explicit anchors matter.
