---
knowledge-base-summary: "Applies marker bodies and drift items to doc pages. Produces unified diffs as drafts. Reads target page + marker context + drift item; writes a draft that respects the page's existing style + section structure. The user-facing accept/reject loop sees its output. Hand-off rule: drafts ride along with the same PR (under /create-pr) or get a fresh PR (manual mode)."
---

# doc-rewriter sub-agent

Procedure document for the `doc-rewriter` sub-agent invoked by `/docs-sync` Phase 1c (marker drain) and Phase 2/3 (drift items).

## Invocation contract

`/docs-sync` calls this sub-agent via `Task` (`subagent_type: general-purpose`) with:

```
target-page: <path/to/page.md>
inputs:
  markers: [{topic, kind, body, doc-impact}, ...]   # Phase 1c
  drift-items: [{subcheck, suggested-fix, ...}, ...] # Phase 2/3
existing-content: <full content of target-page>
related-context: <links to related pages, glossary, code excerpts>
```

The sub-agent's job:

1. Read the target page's current content.
2. Decide where the change lands within the existing structure.
3. Author the new / replaced section content in the page's existing voice.
4. Output a unified diff (the draft).

The sub-agent does NOT stage the diff. `/docs-sync` Phase 1d does the staging through the user's accept / reject gate.

## Authoring rules

### Respect the page's existing voice

Every page has a tone — terse reference, narrative tutorial, step-by-step procedural. Match it. Do not introduce new voices into a single page.

If a marker's `body` is in narrative voice and the target page is reference style, condense the body to fit the reference page's tone — and put the narrative version in a related wiki page (a side effect, not the doc-sync's primary output).

### Respect the page's existing structure

If the page has sections `## Why`, `## Flow`, `## Failure modes`, the new content lands in the appropriate section. Do not invent new sections unless the change genuinely needs one.

When a change requires a new section, place it in the canonical position relative to neighbors:

- Identity / overview → top.
- Steps / flow → middle.
- Failure modes / edge cases → near end.
- Related / Source → bottom.

### Surgical changes only

Touch only what the marker / drift item requires. Do not "improve" adjacent prose, fix unrelated typos, or rework section ordering as a side effect.

The single exception: the [stub-policy-doc-impact-semantics](https://github.com/agentteamland/workspace/blob/main/.claude/wiki/stub-policy-doc-impact-semantics.md) post-Phase-2 policy. If a marker carries `doc-impact: readme` AND the target README is a stub (per uniform stub policy), the sub-agent should re-route the change to the docs-site equivalent and leave the README untouched. The README is documentation of the stub policy itself, not a place for evolving content.

### Convergence with related markers

When multiple markers in the same drain target the same page, collapse them into a single coherent draft. Do not produce three separate drafts that each touch overlapping prose — the user gets confused, the diff reviewer rejects the noise.

A simple rule: group markers by target page; rewrite once with all grouped markers as inputs.

### Drift item disambiguation

Drift items carry `suggested-fix` strings, but the sub-agent should NOT apply the suggestion verbatim. The suggestion is a hint; the actual fix is the sub-agent's responsibility. Use:

- The drift item's `source-truth.value` as the canonical truth to encode.
- The page's surrounding context to phrase the fix in the page's voice.

If a drift item's suggestion contradicts another drift item or a marker body, prefer the source-of-truth value over either suggestion. Note the contradiction in a comment in the draft for review.

## Output format

Unified diff, ready to be staged via `git apply`. Per-target-page, one diff. The diff includes:

- The target page's path (relative to the docs / repo root).
- The diff hunks, with sufficient context (3 lines before / after).
- A 1-2 sentence prefix comment describing the change cluster (which markers / drift items it addresses).

Example:

```
# Cluster: marker auth-refresh (doc-impact: docs) + drift item version-pin
--- a/site/cli/install.md
+++ b/site/cli/install.md
@@ -10,7 +10,7 @@
 ::: code-group
 ```bash [macOS / Linux]
-brew install agentteamland/tap/atl@1.1.3
+brew install agentteamland/tap/atl@1.1.4
 ```
```

The 1-2 sentence prefix is essential context for the user's accept / reject decision and for the reviewer reading the PR diff later.

## Bilingual handoff

When the target page has a TR mirror (most pages under `site/` do), the sub-agent must signal to `/docs-sync` Phase 4 that the EN draft is ready for `parity-checker` Mode 1 (same-pass translation).

The signal is part of the `Task` tool's response payload: `tr-mirror-needed: true|false` + the path of the TR mirror. `/docs-sync` uses this to spawn the parity-checker invocation in the same batch.

Cost-wise, the parallel invocation cuts wall-clock about in half — both the EN draft and the TR mirror are ready by the time the user is presented the accept / reject prompt.

## Hand-off rule (drafts → branch)

`doc-rewriter` does NOT commit. `/docs-sync` Phase 5 stages and commits.

Two flows:

- **Under `/create-pr` Step 4.5**: drafts stage on the active feature branch. Commit message follows: `docs: <type> drift-fix from <PR title>`. The drafts ride along in the same PR — review chain (Step 5) sees them.
- **Manual mode**: drafts stage to a new `docs/sync-<YYYY-MM-DD>` branch in the docs repo (and / or peer repos when README touches happened). New PR per repo, each follows team-repo-maintenance.

Never auto-merge. Per [PR merge discipline](../../rules/team-repo-maintenance.md#pr-merge-discipline-absolute-no-exceptions), the merge belongs to the human reviewer.

## Common pitfalls

### Hallucinated context

The sub-agent must NOT invent content the marker body / drift item did not provide. If a marker says "API key handling changed" but does not say HOW, the draft should reflect that — leave a `<!-- TODO: marker body needs detail -->` placeholder in the draft, and route the marker back for clarification rather than guessing.

### Tone drift

Reference pages in clipped phrasing should not become long narrative pages. Each draft is reviewed for tone preservation before staging.

### Untargeted changes

Every line in the diff must trace to a specific marker / drift item input. If the sub-agent finds itself "improving" something not in the inputs, stop and remove the improvement. Save it for a separate marker.

### Wrong target page

A marker with `doc-impact: docs` may target the wrong page if the topic is ambiguous. The sub-agent should preserve the routing decision but flag uncertain routings:

```
# Cluster: marker auth-refresh (doc-impact: docs)
# NOTE: routed to /cli/install.md because the marker body mentions install workflows.
#       If a different target was intended, reject and re-route via marker re-tag.
```

## Performance budget

- Single page draft: target < 5 seconds wall-clock per cluster.
- Parallel cluster invocations: 3 clusters × 5 seconds ≈ 7-10 seconds (parallelism gives ~2-3× speedup).
- The whole `/docs-sync` Phase 1+2 budget is ~20 seconds for typical PR diffs (1-3 clusters).

## Related

- [drift-detector.md](drift-detector.md) — produces the drift items this sub-agent consumes.
- [parity-checker.md](parity-checker.md) — receives the EN draft for TR mirror generation.
- [stub-policy-doc-impact-semantics wiki](https://github.com/agentteamland/workspace/blob/main/.claude/wiki/stub-policy-doc-impact-semantics.md) — `doc-impact: readme` routing policy after Phase 2 stub conversion.
- [PR merge discipline](../../rules/team-repo-maintenance.md#pr-merge-discipline-absolute-no-exceptions) — the never-merge rule the auto-staging respects.
