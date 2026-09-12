---
name: novelty-check
description: Verify research idea novelty against recent literature. Use when user says "查新", "novelty check", "有没有人做过", "check novelty", or wants to verify a research idea is novel before implementing.
argument-hint: "[method-or-idea-description]"
allowed-tools: WebSearch, WebFetch, Grep, Read, Glob
---

# Novelty Check Skill

Check whether a proposed method/idea has already been done in the literature: **$ARGUMENTS**

## Constants

- No external reviewer model is required — Phase C is Claude self-review (see below). If a genuinely different model/API is available, it can be substituted for Phase C by hand; this skill does not assume one.

## Instructions

Given a method description, systematically verify its novelty:

### Phase A: Extract Key Claims
1. Read the user's method description
2. Identify 3-5 core technical claims that would need to be novel:
   - What is the method?
   - What problem does it solve?
   - What is the mechanism?
   - What makes it different from obvious baselines?

### Phase B: Multi-Source Literature Search
For EACH core claim, search using ALL available sources:

1. **Web Search** (via `WebSearch`):
   - Search arXiv, Google Scholar, Semantic Scholar
   - Use specific technical terms from the claim
   - Try at least 3 different query formulations per claim
   - Include year filters for 2024-2026

2. **Known paper databases**: Check against:
   - ICLR 2025/2026, NeurIPS 2025, ICML 2025/2026
   - Recent arXiv preprints (2025-2026)

3. **Read abstracts**: For each potentially overlapping paper, WebFetch its abstract and related work section

### Phase C: Self-Review Verification
With no Codex CLI or other model/API configured, this phase is Claude
self-review rather than a second model's independent read — see
auto-review-loop's "Self-Review Backend (No Second Model)" for the method and
its honest independence tradeoff (it cannot catch a novelty miss Claude is
systematically blind to the way a different model might).

Write a dossier file such as `NOVELTY_DOSSIER.md` (or a project-local
equivalent) containing the method description, core claims, and all candidate
papers found in Phase B — this keeps the record durable and forces a genuine
re-read rather than a judgment from memory. Then, **re-open the dossier as if
reading a stranger's submission** (not from memory of writing it) and answer,
in writing, for each core claim: "Is this method novel? What is the closest
prior work? What is the delta?" — actively arguing for the strongest case that
it is NOT novel before concluding otherwise, and flagging any claim Claude is
too invested in to judge impartially.

### Phase D: Novelty Report
Output a structured report:

```markdown
## Novelty Check Report

### Proposed Method
[1-2 sentence description]

### Core Claims
1. [Claim 1] — Novelty: HIGH/MEDIUM/LOW — Closest: [paper]
2. [Claim 2] — Novelty: HIGH/MEDIUM/LOW — Closest: [paper]
...

### Closest Prior Work
| Paper | Year | Venue | Overlap | Key Difference |
|-------|------|-------|---------|----------------|

### Overall Novelty Assessment
- Score: X/10
- Recommendation: PROCEED / PROCEED WITH CAUTION / ABANDON
- Key differentiator: [what makes this unique, if anything]
- Risk: [what a reviewer would cite as prior work]

### Suggested Positioning
[How to frame the contribution to maximize novelty perception]
```

### Important Rules
- Be BRUTALLY honest — false novelty claims waste months of research time
- "Applying X to Y" is NOT novel unless the application reveals surprising insights
- Check both the method AND the experimental setting for novelty
- If the method is not novel but the FINDING would be, say so explicitly
- Always check the most recent 6 months of arXiv — the field moves fast
- **Anti-hallucination for Closest Prior Work.** Every paper in the prior-work table must pass pre-search verification via `verify_papers.py` (canonical name resolved per [`shared-references/integration-contract.md`](../shared-references/integration-contract.md) §2; 3-layer arXiv / CrossRef / Semantic Scholar fallback inside the helper itself). Policy D1 (primary + degraded-output fallback): if the helper is unresolved **or** its invocation fails, tag candidate entries `[UNVERIFIED]` and surface the uncertainty rather than dropping them. Never fabricate arXiv IDs, DOIs, or titles from memory. Full protocol in [`shared-references/citation-discipline.md`](../shared-references/citation-discipline.md) § Pre-Search Verification Protocol.

## Review Tracing

Phase C's self-review has no MCP thread to trace. Record `reviewer_backend: self` and `independence_verified: false` alongside the novelty verdict in the report, so a later reader knows this round wasn't independently checked. If a genuinely different model is substituted for Phase C by hand, save its trace following `shared-references/review-tracing.md` (Policy C — forensic; never silently skip) via `save_trace.sh` (resolved per the chain in `shared-references/integration-contract.md` §2), respecting `--- trace:` (default: `full`).
