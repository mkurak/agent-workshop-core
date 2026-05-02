---
name: wiki
description: "Project knowledge base — living, cross-referenced, always current. Initialize, ingest new knowledge, query existing, lint for staleness. Powered by Karpathy's LLM Wiki pattern."
argument-hint: "<init|ingest|query|lint> [topic or question]"
---

# /wiki Skill

## Purpose

Maintains a living knowledge base for the project. Unlike memory (append-only historical notes) or docs (static decision records), the wiki is **actively maintained** — pages are updated when knowledge changes, cross-references are kept current, stale information is cleaned up.

The wiki answers: "What is the current truth about X in this project?"

## Location

```
.claude/wiki/
├── index.md                    ← Auto-maintained table of contents
├── {topic-1}.md                ← Knowledge pages (kebab-case)
├── {topic-2}.md
└── ...
```

## Four Modes

### `init` Mode

**Usage:** `/wiki init` (no argument)

Scaffolds `.claude/wiki/` in the current project. Safe to re-run — it's a no-op if the wiki is already initialized.

**Process:**

1. Check for `.claude/wiki/` directory.
   - Does not exist → create it.
   - Exists with `index.md` containing the initialized template → **idempotent no-op**, print `wiki: already initialized (N pages)` and exit.
   - Exists but empty or missing `index.md` → treat as not-yet-initialized and continue.
2. Create `.claude/wiki/index.md` with this template:

   ```markdown
   # Project Wiki

   > Current truth about this project — maintained automatically from `<!-- learning -->` markers via `/save-learnings`, or manually via `/wiki ingest`.

   ## Pages

   (none yet — pages appear here as knowledge accumulates)

   ## Health

   - Last ingest: never
   - Last lint: never

   ## How this works

   - **Content flows in** from inline learning markers captured at session end (`atl learning-capture`), plus any manual `/save-learnings` or `/wiki ingest` invocation.
   - **Pages are topic-based** (one concept per page) and reflect current truth — old facts get replaced, not appended.
   - **Run `/wiki query <question>`** to search wiki content.
   - **Run `/wiki lint`** periodically to catch stale / orphan / contradicting pages.
   ```

3. Report one-line status: `wiki initialized at .claude/wiki/ (ready for ingest)`

**When init fires automatically:**

- First session of a project where the user has `atl setup-hooks` installed but the project has no `.claude/wiki/` yet — the `SessionStart` hook invokes `/wiki init` as part of the wiki-bootstrap step.
- Manual: user runs `/wiki init` directly.

**What init does NOT do:**

- It does not ingest existing knowledge sources. Use `/wiki ingest` for that after init.
- It does not create topic pages. Pages are created by `ingest` or `save-learnings` when content exists.

---

### `ingest` Mode

**Usage:** `/wiki ingest` (no argument needed)

Scans all project knowledge sources and updates wiki pages:

**Sources scanned:**
1. `<!-- learning -->` markers in the current session transcript — primary source, see [learning-capture rule](../../rules/learning-capture.md)
2. `.claude/agent-memory/*.md` — agent learnings
3. `.claude/journal/*.md` — inter-agent notes
4. `.claude/docs/*.md` — finalized decisions from brainstorms
5. `.claude/brain-storms/*.md` (completed only) — decision context
6. Recent conversation context — what was just discussed/built

**Process:**
1. Read all sources
2. For each piece of knowledge, determine the topic (e.g., "caching", "authentication", "order-management")
3. If wiki page exists for that topic → **update** (merge new info, resolve contradictions, keep latest truth)
4. If no wiki page exists → **create** new page
5. Update cross-references (backlinks between related pages)
6. Update `index.md` with new/changed pages

**Page format:**

```markdown
# {Topic Title}

> Last updated: {date}
> Sources: [agent-memory](../agent-memory/api-agent-memory.md), [brainstorm](../brain-storms/auth-design.md)

## Summary
{2-3 sentence overview of this topic in the project}

## Current State
{What is true RIGHT NOW — not history, not plans, just current reality}

## Key Decisions
{Important decisions made about this topic, with brief reasoning}

## Patterns & Rules
{Established patterns, conventions, rules for this topic}

## Known Issues
{Current problems or limitations}

## Related
- [{related-topic-1}]({related-topic-1}.md)
- [{related-topic-2}]({related-topic-2}.md)
```

**Important:** Wiki pages reflect **current truth.** If old memory says "we use pattern X" but later memory says "pattern X caused problems, switched to Y" — the wiki page says "we use Y" (not both).

---

### `query` Mode

**Usage:** `/wiki query how does caching work in this project?`

1. Read `index.md` to find relevant pages
2. Read those pages
3. Synthesize an answer from wiki content
4. Cite which wiki pages the answer came from

This is useful when:
- New developer (or new Claude session) needs to understand a topic
- You forgot how something was decided/implemented
- You want a quick refresher before making changes

---

### `lint` Mode

**Usage:** `/wiki lint`

Health check for the entire wiki:

**Checks performed:**
1. **Stale pages** — last updated > 30 days ago, source files have changed since → flag for review
2. **Contradictions** — two pages say conflicting things → flag for resolution
3. **Orphan pages** — page exists but no other page references it → suggest connections
4. **Missing pages** — topic referenced in another page but no page exists → suggest creation
5. **Duplicate topics** — two pages cover the same topic → suggest merge
6. **Index sync** — index.md matches actual files in wiki/ → fix if out of sync

**Output:**
```
Wiki Health Report:
──────────────────────────
📊 Total pages: 12
✅ Healthy: 9
⚠️  Stale (>30 days): 2 (caching-patterns.md, email-setup.md)
❌ Contradiction found: auth.md says "15min token" but jwt-config.md says "30min token"
🔗 Orphan: database-indexes.md (no incoming links)
📝 Missing: "rate-limiting" referenced in api-endpoints.md but no page exists
──────────────────────────

Fixing automatically...
✅ index.md synced
✅ Created rate-limiting.md (stub)
⚠️  Review needed: auth.md vs jwt-config.md contradiction
⚠️  Review needed: 2 stale pages
```

Auto-fixable issues are fixed silently. Contradictions and stale content are reported for human review.

---

## Integration with Other Systems

### Learning markers → /save-learnings → Wiki

The normal flow when `atl setup-hooks` is installed:

1. Claude drops `<!-- learning topic=... -->` markers inline during the conversation (per [learning-capture](../../rules/learning-capture.md) rule)
2. Session end / PreCompact → `atl learning-capture` scans transcript for markers
3. If markers found → `/save-learnings` runs on the marked regions
4. `/save-learnings` updates agent-memory (append), journal, and wiki pages (replace/update)

Example propagation:

```
<!-- learning topic: redis-cache; body: TTL should be 30 min, not 15 -->
  → agent-memory: append historical note with date
  → wiki/redis-cache.md: UPDATE "TTL is 30 minutes" (replace old "15 minutes")
  → journal/{date}_{agent}.md: cross-agent summary entry
```

Without markers (or without hooks), manual `/wiki ingest` and manual `/save-learnings` still work.

### Agent Startup → Wiki

Agents read relevant wiki pages at session start (per knowledge-system rule). Agent doesn't read ALL pages — only those related to its domain:
- API Agent → pages about api patterns, database, auth, caching
- Flutter Agent → pages about ui patterns, state management, navigation
- Selection is based on page topics matching agent's responsibility area

### /brainstorm done → Wiki

When a brainstorm completes, its decisions are ingested into the wiki automatically.

---

## Wiki Page Lifecycle

```
New knowledge discovered (conversation, brainstorm, learning)
    ↓
Determine topic
    ↓
Page exists?
├── Yes → Update page (merge, resolve contradictions, update date)
└── No → Create new page (from template)
    ↓
Update cross-references (backlinks)
    ↓
Update index.md
```

## Important Rules

1. **Wiki = current truth.** Not history, not plans. What is true RIGHT NOW.
2. **Update, don't append.** If a fact changes, the old version is replaced, not kept alongside.
3. **Cross-reference always.** Every page should link to related pages. Orphans are flagged by lint.
4. **Auto-maintained.** Humans rarely edit wiki directly. It's maintained by /save-learnings, /brainstorm done, and /wiki ingest.
5. **Agent-readable.** Pages are structured for both human and AI consumption — clear sections, no ambiguity.
6. **Topic-based, not date-based.** Unlike journal (date-based) or memory (date-based), wiki is organized by topic. One page per concept.
7. **Lint regularly.** Run `/wiki lint` periodically (monthly or when something feels off).
