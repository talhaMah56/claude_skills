---
name: patent-review
description: "Get a patent-examiner-perspective review of a patent application. Use when user says \"专利审查\", \"patent review\", \"审查意见\", \"examiner review\", or wants critical feedback on patent claims and specification."
argument-hint: "[patent-directory-or-scope]"
allowed-tools: Bash(*), Read, Grep, Glob, Write, Edit
---

# Patent Examiner Review: Self-Review (No Second Model)

Get a multi-round patent-examiner-perspective review of the patent application based on: **$ARGUMENTS**

Adapted from `/research-review`. The reviewer persona is a patent examiner, not a paper reviewer.

No external model is available, so this skill's examiner review is performed by Claude itself rather than delegated to Codex: Claude re-reads the claims and specification fresh from disk (not from memory of drafting them) and adopts an adversarial patent-examiner persona. It follows the same discipline as `auto-review-loop`'s "Self-Review Backend (No Second Model)" — a real check that actively argues against the application and flags spots it can't evaluate impartially, but with no independence guarantee against blind spots a genuinely different model would catch.

## Constants

- `REVIEW_ROUNDS = 2` — Number of review rounds
- `EXAMINER_PERSONA = "patent-examiner"` — Self-review persona Claude adopts (no external model)

## Inputs

1. `patent/CLAIMS.md` — all drafted claims
2. `patent/specification/` — all specification sections
3. `patent/figures/numeral_index.md` — reference numeral mapping
4. `patent/PRIOR_ART_REPORT.md` — known prior art
5. `patent/INVENTION_DISCLOSURE.md` — invention structure

## Workflow

### Step 1: Gather Patent Context

Before the self-review examiner pass, compile a comprehensive briefing:
1. Read all claims (independent + dependent)
2. Read specification sections (at least summary and detailed description)
3. Read prior art report for context
4. Identify: core inventive concept, claim scope, known prior art, target jurisdiction

### Step 2: Round 1 — Full Examiner Review

No second model is available, so Claude performs this round itself: re-read the claims and specification fresh from the actual files (not from memory of drafting them), adopt a senior patent examiner persona at the [USPTO/CNIPA/EPO], and issue a detailed office action against:

CLAIMS: [all claims]
SPECIFICATION SUMMARY: [key sections: title, technical field, background, summary, abstract]
PRIOR ART KNOWN: [prior art references]
PATENTABILITY STANDARDS TO APPLY: [US: 35 USC 101/102/103/112 | CN: Articles 22, 26 | EP: Articles 54, 56, 83, 84]

Issue an office action covering:

1. CLAIM CLARITY (112(b)/Art 84):
   - Are all terms definite?
   - Any indefinite functional language?
   - Antecedent basis issues?

2. WRITTEN DESCRIPTION (112(a)/Art 83 first para):
   - Does the spec support ALL claim scope?
   - Any claim elements without spec support?

3. ENABLEMENT (112(a)/Art 83):
   - Can a POSITA practice the invention?
   - Any missing algorithm/structure for functional claims?

4. NOVELTY (102/Art 54):
   - Would any known reference anticipate any claim?
   - Identify the closest single reference.

5. NON-OBVIOUSNESS (103/Art 56):
   - Would any combination render claims obvious?
   - What is the motivation to combine?

6. CLAIM SCOPE:
   - Are independent claims broad enough to be commercially valuable?
   - Do dependent claims provide meaningful fallback positions?
   - Any claims that are too broad (likely rejected) or too narrow (not valuable)?

7. SPECIFICATION QUALITY:
   - Language issues (subjective terms, relative terms, result-to-be-achieved)
   - Reference numeral consistency
   - Missing embodiments

Format the output as a formal office action with:
- GROUNDS OF REJECTION for each issue (cite statute)
- SUGGESTED AMENDMENTS for each issue
- OVERALL PATENTABILITY SCORE: 1-10

Be rigorous and specific, and actively argue against the application rather than confirming it — this is meant to be a real, adversarial examination, not a rubber stamp. Flag any section you cannot evaluate impartially because you just drafted it.

### Step 3: Implement Fixes (Round 1)

Based on the examiner's office action:

1. **CRITICAL issues** (102 rejection, 112 indefiniteness, missing enablement):
   - Must be fixed before proceeding
   - Amend claims or add specification support

2. **MAJOR issues** (103 obviousness, weak claim scope, missing support):
   - Should be fixed or argued
   - Consider claim amendments or specification additions

3. **MINOR issues** (language quality, numeral consistency, formatting):
   - Fix if time permits
   - Document in output for later cleanup

For each fix:
- Show the specific change (old claim -> new claim)
- Explain how the fix addresses the examiner's concern

### Step 4: Round 2 — Follow-Up Review

Claude re-examines its own Round 1 office action against the revised application, again re-reading the current files fresh rather than relying on memory of the fixes just applied:

CHANGES MADE: [list of all changes with rationale]
REVISED CLAIMS: [updated claims]
REVISED SPECIFICATION EXCERPTS: [changed sections]

Re-examine:
1. Are the previous rejections overcome?
2. Are there new issues introduced by the amendments?
3. What is the updated patentability score?
4. Any remaining grounds for rejection?

Because the same session both fixed the issues and now checks them, this round is especially prone to confirmation bias ("of course I fixed it") — actively look for cases where a fix is cosmetic rather than substantive, and say so plainly rather than defaulting to a clean pass. This is the self-review discipline from `auto-review-loop`'s "Self-Review Backend (No Second Model)": a real check, but with no independence guarantee.

### Step 5: Generate Improvement Report

Write `patent/PATENT_REVIEW.md`:

```markdown
## Patent Review Report

### Application Summary
[Title, claims count, jurisdiction]

### Review Round 1
#### Office Action Summary
[Key findings from examiner]

#### Issues Found
| # | Type | Severity | Claim/Section | Issue | Citation | Fix Applied |
|---|------|----------|--------------|-------|----------|-------------|
| 1 | Clarity | CRITICAL | Claim 3 | Indefinite term "rapid" | 112(b) | Defined in spec |
| 2 | Novelty | MAJOR | Claim 1 | Ref X anticipates element C | 102 | Amended claim |

#### Score After Round 1: [X]/10

### Review Round 2
#### Follow-Up Assessment
[Are previous rejections overcome?]

#### Remaining Issues
[Any issues still outstanding]

#### Score After Round 2: [X]/10

### Recommendations
[Final recommendations before proceeding to jurisdiction formatting]
- [ ] All CRITICAL issues resolved
- [ ] All MAJOR issues resolved or argued
- [ ] Specification supports all claim amendments
- [ ] Ready for jurisdiction formatting
```

## Key Rules

- The reviewer persona must be a patent examiner, not a paper reviewer or academic.
- Always apply maximum analysis depth -- do not shortcut the self-review passes.
- Address CRITICAL and MAJOR issues before proceeding to the next phase.
- Document all changes in the review report for traceability.
- If the patentability score is below 5/10 after Round 2, recommend significant rework before filing.
- The review is advisory -- actual prosecution may proceed differently.
- Both rounds are self-review, not independent review -- note this in the output rather than implying an external examiner was consulted.
