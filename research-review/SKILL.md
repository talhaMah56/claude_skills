---
name: research-review
description: Get a deep critical review of research from a reviewer backend (Claude self-review by default, or Codex/manual if configured). Use when user says "review my research", "help me review", "get external review", or wants critical feedback on research ideas, papers, or experimental results.
argument-hint: "[topic-or-scope]"
allowed-tools: Bash(*), Read, Grep, Glob, Write, Edit, mcp__manual_review__review, mcp__manual_review__review_reply
---

# Research Review via External Reviewer Backend (ultra reasoning)

> 🔒 **Do not wrap this skill in `/loop`, `/schedule`, or `CronCreate`.** It is
> verdict-bearing — it produces a cross-model review verdict, multi-round with
> reviewer thread continuity. An external timer re-fires the verdict on
> wall-clock time and breaks the reviewer's round-to-round memory: zero new
> signal, full token cost. Schedule the *external wait that precedes it* (work
> ready → then review once), not the verdict. See
> [`shared-references/external-cadence.md`](../shared-references/external-cadence.md).

Get a multi-round critical review of research work from the selected external reviewer backend with maximum reasoning depth.

## Constants

- **REVIEWER_BACKEND = `self`** — Default when no Codex CLI or other model/API is configured: Claude reviews its own work (no `REVIEWER_MODEL` needed) at the same `ultra`-equivalent depth. Override with `— reviewer: manual` for Manual Review MCP if a genuinely different model is available — that backend uses a model the user chooses, but it must be a recognized model from a different family (OpenAI, Anthropic, Google, DeepSeek, Moonshot/Kimi, Qwen). See auto-review-loop's "Self-Review Backend (No Second Model)" for what `self` does and the independence tradeoff it carries.

## Reviewer Calling Convention

When calling the reviewer, branch on REVIEWER_BACKEND:

**If REVIEWER_BACKEND = `self`** (default — no Codex CLI or other model/API configured):
  No MCP thread is involved. Re-read the briefing fresh from disk (not from
  memory of writing it) and answer the same prompt a reviewer would get,
  actively arguing against the work's own claims before accepting them, and
  flagging any part Claude cannot judge impartially — see auto-review-loop's
  "Self-Review Backend (No Second Model)" for the full method.

**If REVIEWER_BACKEND = `manual`:**
  Use `mcp__manual_review__review` for new review threads with:
    prompt: [the same review prompt used under `self` above]
    config: {"model_reasoning_effort": "xhigh", "executor_model": "<actual executor model>", "require_reviewer_model": true}
  Save the returned `threadId`.
  Use `mcp__manual_review__review_reply` for follow-up rounds with:
    threadId: [saved manual-review threadId]
    prompt: [follow-up prompt]
    config: {"model_reasoning_effort": "xhigh", "executor_model": "<actual executor model>", "require_reviewer_model": true}

Content fidelity: the manual reviewer should see the same substantive review
brief a `self`-backend read would use. If the manual UI supports file upload / attachment,
reuse the same brief file; otherwise paste the brief contents inline because
remote web UIs cannot read your local filesystem paths. Review tracing applies
to every backend (see *Review Tracing* below).

## Context: $ARGUMENTS

## Prerequisites

- **None, by default.** With no Codex CLI or other model/API configured, `REVIEWER_BACKEND = self` needs nothing beyond Claude Code itself.
- **Optional — Manual Review MCP**, for a genuinely independent review when another model is available. Configure per its own setup docs; then pass `— reviewer: manual`.

## Workflow

### Step 1: Gather Research Context
Before calling the external reviewer, compile a comprehensive briefing:
1. Read project narrative documents (e.g., STORY.md, README.md, paper drafts)
2. Read any memory/notes files for key findings and experiment history
3. Identify: core claims, methodology, key results, known weaknesses

### Step 2: Initial Review (Round 1)
Send a detailed prompt with ultra-equivalent reasoning depth, using the
selected backend. Write the full briefing to `RESEARCH_REVIEW_REQUEST.md`
first either way — it's the durable artifact self-review re-reads fresh.

*For `self` backend (default):* re-read `RESEARCH_REVIEW_REQUEST.md` from
disk (not from memory of writing it) and answer the prompt below directly —
see auto-review-loop's "Self-Review Backend" for the method:

```
Read the review brief at <absolute path to RESEARCH_REVIEW_REQUEST.md>.
Notes made while producing the work are not evidence beyond the files they
cite, so verify the referenced artifacts before judging.
Please act as a senior ML reviewer (NeurIPS/ICML level). Start from the
assumption that the work is broken somewhere — your job is to find where.
Be adversarial. Trust nothing the work's own narrative tells you — verify
everything yourself. Identify:
1. Logical gaps or unjustified claims
2. Missing experiments that would strengthen the story
3. Narrative weaknesses
4. Whether the contribution is sufficient for a top venue
Please be brutally honest. Flag any point Claude cannot judge impartially
because it produced the work being reviewed.
```

The review brief should contain the full research context, the specific
questions, and the primary artifact / raw-result paths the reviewer should
inspect.

*For manual backend:* use `mcp__manual_review__review` with the same brief
contents. If the manual-review UI supports attachments, attach
`RESEARCH_REVIEW_REQUEST.md`; otherwise paste the brief inline. Save the
returned `threadId`.

### Step 3: Iterative Dialogue (Rounds 2-N)
For `self` backend (default): write an updated brief such as
`RESEARCH_REVIEW_ROUND_2.md`, then re-read it fresh from disk and answer
directly — no threadId, so explicitly re-state the prior round's unresolved
weaknesses in the prompt rather than relying on thread memory:

```text
Read the updated review brief at <absolute path to
RESEARCH_REVIEW_ROUND_2.md>.
Focus on unresolved weaknesses from the prior round and whether the revision
actually fixed them, not just reworded around them.
```

For `manual` backend: use `mcp__manual_review__review_reply` with the same
`threadId`. Attach that same updated brief if possible; otherwise paste it
inline.

For each round:
1. **Respond** to criticisms with evidence/counterarguments
2. **Ask targeted follow-ups** on the most actionable points
3. **Request specific deliverables**: experiment designs, paper outlines, claims matrices

Key follow-up patterns:
- "If we reframe X as Y, does that change your assessment?"
- "What's the minimum experiment to satisfy concern Z?"
- "Please design the minimal additional experiment package (highest acceptance lift per GPU week)"
- "Please write a mock NeurIPS/ICML review with scores"
- "Give me a results-to-claims matrix for possible experimental outcomes"

### Step 4: Convergence
Stop iterating when:
- Both sides agree on the core claims and their evidence requirements
- A concrete experiment plan is established
- The narrative structure is settled

### Step 5: Document Everything
Save the full interaction and conclusions to a review document in the project root:
- Round-by-round summary of criticisms and responses
- Final consensus on claims, narrative, and experiments
- Claims matrix (what claims are allowed under each possible outcome)
- Prioritized TODO list with estimated compute costs
- Paper outline if discussed

Update project memory/notes with key review conclusions.

> **Composed mode** — if invoked with `— composed: <canonical-report-path>` (an
> orchestrator like `/idea-discovery` passes this), do **not** write a standalone review
> `.md` in the project root. The raw conversation is already persisted to `.aris/traces/…`
> (see *Review Tracing* below — that audit copy is kept in every mode); fold the review
> *conclusions* (consensus, claims matrix, prioritized TODOs) into the orchestrator's
> canonical report and cite the trace path there. **Default (no `— composed:` directive):
> behave exactly as above — write the standalone review document.** Never infer composed
> mode from a report file merely existing. Full rules:
> [`shared-references/output-composition.md`](../shared-references/output-composition.md).

## Key Rules

- Under `self` (default), review at the same depth every round implies: re-read fresh, argue against the work, flag impartiality gaps — see auto-review-loop's "Self-Review Backend". It is a real check but not an independent one; prefer `manual` with a genuinely different model whenever one is available.
- For `manual`, use the identity-bearing config from the Reviewer Calling Convention above (`model: gpt-5.6-sol` + `ultra`/`xhigh` reasoning is the Codex-specific pin from before this conversion and no longer applies under `self`)
- Put comprehensive context in the review brief either way — self-review reads local files directly; manual reviewers usually cannot, so attach or paste the same brief there.
- Be honest about weaknesses — hiding them leads to worse feedback
- Push back on criticisms you disagree with, but accept valid ones
- Focus on ACTIONABLE feedback — "what experiment would fix this?"
- Document the threadId for potential future resumption (`manual` backend only)
- The review document should be self-contained (readable without the conversation)

## Prompt Templates

### For initial review:
"I'm going to present a complete ML research project for your critical review. Please act as a senior ML reviewer (NeurIPS/ICML level)..."

### For experiment design:
"Please design the minimal additional experiment package that gives the highest acceptance lift per GPU week. Our compute: [describe]. Be very specific about configurations."

### For paper structure:
"Please turn this into a concrete paper outline with section-by-section claims and figure plan."

### For claims matrix:
"Please give me a results-to-claims matrix: what claim is allowed under each possible outcome of experiments X and Y?"

### For mock review:
"Please write a mock NeurIPS review with: Summary, Strengths, Weaknesses, Questions for Authors, Score, Confidence, and What Would Move Toward Accept."

## Review Tracing

A `self` round has no MCP thread to trace — instead record `reviewer_backend: self` and `independence_verified: false` alongside the round's conclusions in the review document. After each `mcp__manual_review__review` or `mcp__manual_review__review_reply` call, save the trace following `shared-references/review-tracing.md` (Policy C — forensic; never silently skip). Use `save_trace.sh` (resolved per the chain in `shared-references/integration-contract.md` §2) or write files directly to `.aris/traces/<skill>/<date>_run<NN>/`. Respect the `--- trace:` parameter (default: `full`).
  A verdict-bearing manual response MUST begin with
  `Reviewer-Model: <exact-model-id>` — pass the model THIS session is actually
  running as in `executor_model`. Missing, unknown, or same-family identity
  cannot acquit; emit `REVIEW_UNAVAILABLE` rather than guessing. If the executor
  model cannot be named, manual review's cross-family claim is unprovable — say
  so in the report instead of asserting it.
