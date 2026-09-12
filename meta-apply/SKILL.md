---
name: meta-apply
description: "Privileged applier that LANDS meta-optimize / corpus-audit patches the user approved — the ONLY skill permitted to mutate the skill corpus from a self-modification proposal, with a reviewer jury (self-review, with a much weaker guarantee, when no other model is configured — see below) and human approval at landing. Use when the user says \"meta apply\", \"/meta-apply\", \"land the staged patches\", \"应用优化\", after a /meta-optimize run."
argument-hint: "[patch-number-or-all]"
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob
---

# Meta-Apply: the privileged landing gate for self-modification patches

This skill exists to enforce a **privilege boundary**. Producers like
[`/meta-optimize`](../meta-optimize/SKILL.md) (and, later, `corpus-audit`) are
**read-only** — no `Write`/`Edit`, no apply step; they can only *stage* candidate patches
under `.aris/meta/pending/`. **This skill is the only place a staged patch becomes a real
change to the corpus.** Splitting "propose" from "land" across two skills with different
tool grants is what makes "a loop cannot apply its own patch" structural rather than a
sentence the producer is asked to obey.

It is **human-invoked only.** It runs when the user explicitly types `/meta-apply` after
reading the producer's REPORT. That invocation IS the landing authorization (the locked
2026-05-30 decision: the human stays at the landing gate).

## The acquittal is generated HERE, not relayed

The earlier design trusted a `jury_verdict: pass` field in the producer's manifest. That
is **forgeable** — the producer writes that field, so a buggy/misaligned producer could
stage a killed patch labelled "pass." **So this skill does not trust any producer-written
verdict.** It runs the jury **itself, at landing time, on the actual staged
diff** (fresh thread when an external model is available, paths-only). The verdict is
produced where it is consumed, by the privileged human-invoked skill — nothing to forge.

### Self-Review Fallback (No Second Model) — read before relying on this

This skill was built around a **structural** guarantee: the model that authored a corpus
patch must never be the one that acquits it (`assert_cross_family` raises and refuses on a
same-family pairing). With no Codex CLI or other model/API configured, there is no second
family to check against, and that guarantee **cannot be met** — Claude reviewing a patch
to Claude's own skill definitions is exactly the self-modification-without-oversight
scenario this skill exists to prevent. This is a sharper version of the tradeoff in
auto-review-loop's "Self-Review Backend (No Second Model)": everywhere else in this
collection self-review is a weaker quality check; here it means the loop can no longer
structurally be stopped from self-acquitting its own edits to itself.

Given that, when `self` is the only backend available: run the self-review below and
record `identity_assurance: self_review_no_independence`, `independence_verified: false`
in place of the cross-family check — but treat a `self`-reviewed PASS as a **draft
recommendation for the human**, not as satisfying Rule 3. Before landing a `self`-reviewed
patch, read the staged `.diff` yourself (the human) rather than trusting the verdict alone;
this skill cannot manufacture the independence it was designed to require. If a genuinely
different model/API becomes available later, prefer it for this skill over any other in the
collection — the cost of self-acquitting corpus mutation is the highest here.

## The non-negotiable rules (jury-at-landing, reject-default)

For each staged patch the user asks to land, in order — any failure ⇒ skip & report,
never silently apply:

1. **The human named THIS patch.** Apply only patches the user listed (`/meta-apply 1,3`
   or `all`); default to applying nothing.
2. **Fresh jury PASS, obtained now.** With an external model configured (`— reviewer:
   codex` or `manual`), run it fresh (not a reply/continued thread; `reasoning effort:
   ultra`/`xhigh`, read-only sandbox, paths-only per
   [`reviewer-independence.md`](../shared-references/reviewer-independence.md)) on the
   staged `.diff` + its target. With no external model configured (default), re-read the
   diff and target fresh from disk and answer the same prompt yourself — see "Self-Review
   Fallback" above; this is a materially weaker check. Either way, ask: *does this change
   improve the harness without regressions; PASS or KILL + one-line reason.* Include the
   scope-limits block from
   [`review-scope-limits.md`](../shared-references/review-scope-limits.md) in that prompt:
   this jury judges ARIS's own mechanism, so an over-defensive KILL permanently blocks a
   good patch. Note the block bans *proposing new* hash binding — it is not a reason to
   KILL a patch that touches the existing provenance stamp. **KILL ⇒ refuse.** The human
   cannot override a KILL — they may only pick among PASSED survivors. (A loop can
   DRIVE; only an independent acquittal — or, under `self`, the human's own reading of the
   diff — can ACQUIT.)
3. **Author ≠ reviewer family, when there is a reviewer family to check.** The author is
   the producer's executor model; the reviewer is the external model that just judged it.
   Run `provenance.py assert_cross_family` — if it raises (same family / unknown), refuse.
   (With an external reviewer configured this always holds: producer=Claude, jury=codex or
   another family — the check is the structural backstop.) **Under `self`, skip this
   check** (there is no second family) and instead record
   `identity_assurance: self_review_no_independence` — see "Self-Review Fallback" above for
   why that materially changes what a PASS means here.

## Workflow

### Step 0: Load staging + resolve the helper

```bash
PENDING=".aris/meta/pending"
[ -d "$PENDING" ] || { echo "Nothing staged. Run /meta-optimize first."; exit 0; }
echo "Staged:"; cat "$PENDING/manifest.jsonl"
```

Resolve `provenance.py` via the 4-layer chain in
[`integration-contract.md`](../shared-references/integration-contract.md) §2
(`.aris/tools/` → `tools/` → `$ARIS_REPO/tools/` → `$ARIS_REPO/tools/` via
`~/.aris/repo`).

### Step 1: Jury-at-landing for each requested patch

For every patch the user asked to land, read its staged `.diff` and target, then run the
fresh jury per Rule 2 — an external reviewer if one is configured, otherwise Claude
self-review (weaker; see "Self-Review Fallback" above) — paths-only, no producer
reasoning, no prior-round context. Record `{patch, jury_verdict, jury_thread_id,
one_line_reason, identity_assurance}` (`jury_thread_id` is `null` and `identity_assurance`
is `self_review_no_independence` under `self`). Print a one-line result per patch (`PASS →
eligible` / `KILL → refused: <reason>`), and for a `self`-reviewed PASS, tell the human
explicitly that this patch was not independently checked.

> The producer may have written an *advisory* pre-screen into the manifest to help the
> human read the REPORT — **ignore it for the landing decision.** Only this fresh verdict
> counts.

### Step 2: Land the survivors (Write/Edit only — never Bash)

For each patch that PASSED Step 1 **and** was named by the user:

1. **Back up** the target to `.aris/meta/backups/<date>/<target>` (use the **Write** tool
   to copy contents; corpus paths are not Bash-writable when `corpus_write_guard` is
   active — and the applier should use Write/Edit for corpus mutation anyway).
2. **Apply** the diff by **Edit/Write** on the target corpus file.
3. **Stamp provenance** on the changed file:
   ```bash
   python3 "$PROVENANCE" stamp "$TARGET" --author "$AUTHOR" \
     --reviewer "$JURY_MODEL" --verdict-id "$JURY_THREAD_ID"
   ```
   `stamp()` re-asserts cross-family and refuses on same-family — the structural backstop
   at the moment the authorization record is written. Under `self`, pass `--reviewer self
   --identity-assurance self_review_no_independence` instead (skip the cross-family
   assertion, which would otherwise correctly refuse a same-family stamp) so the receipt
   honestly records that no independent model acquitted this change. The stamp is a
   **process receipt** (who authored, who acquitted-at-landing, content hash) — NOT a claim
   the change is correct, and under `self` it is not even a claim of independence.
4. **Log** to `.aris/meta/optimizations.jsonl`:
   `{ts, patch, target, author_model, reviewer_model, jury_thread_id, identity_assurance,
   applied: true}` (`reviewer_model: "self"`, `jury_thread_id: null`, `identity_assurance:
   "self_review_no_independence"` under `self`).

### Step 3: Report

Per patch: `LANDED <target>` (+ backup path + provenance sidecar) or
`REFUSED <patch>: <reason>`. Remove landed patches from `.aris/meta/pending/`. Remind the
user a landed patch is revertable from its backup, and to test the changed skill next run.

## Provenance is a receipt, not an acquittal of correctness

A stamp records that a change passed *a process* (cross-model jury at landing + human
landing), not that it is *correct*. To prevent "approved-but-wrong with a stamp that
vouches for it" (false-authority laundering — worse than no stamp, because a later
auto-curator reads it as evidence):

- The stamp carries `verdict_id` (auditable review) + `content_hash` (a later hand-edit
  invalidates it).
- **Recommended (not yet built):** a TTL forcing re-review of long-lived auto-authored
  artifacts, and a behavioral auditor that REVOKES a stamp when a landed skill misbehaves.
  Track as follow-up; never treat a stamp as permanent truth.

## Key Rules

- **Human-invoked only.** Never run as a side-effect of another skill or a hook.
- **Jury-at-landing, reject-default, no override.** The binding verdict is produced HERE
  on the staged diff; never trust a producer-written verdict; the human picks among
  survivors, never resurrects a KILL.
- **Cross-family or refuse — when a second family exists.** `assert_cross_family` must not
  raise. A `deterministic:<verifier>` reviewer is valid per skill-governance.md. Under
  `self` (no external model configured) this check is skipped by necessity, not by choice
  — see "Self-Review Fallback" above — and the human's own reading of the diff is the
  actual backstop for that patch.
- **Corpus mutation goes through Write/Edit** (reviewable, attributable), not Bash. The
  `corpus_write_guard` hook (if installed) additionally denies Bash corpus writes — it
  does NOT gate Write/Edit, so it does not by itself stop this skill from editing the
  corpus; the jury-at-landing + stamp discipline above is what governs Write/Edit
  mutations (that discipline is procedure, not a hook-enforced mechanism).
- **Back up before every mutation.** Reversible by construction.
- **Only land staged patches.** Applies what producers staged in `.aris/meta/pending/`;
  invents nothing of its own.

## Review Tracing

Save each landing-jury call's trace (external reviewer or `self`) per
[`review-tracing.md`](../shared-references/review-tracing.md) to
`.aris/traces/meta-apply/<date>_run<NN>/` — the acquittal that landed a corpus change must
be forensically recoverable. A `self` round has no external thread; trace it with
`reviewer_backend: self`, `identity_assurance: self_review_no_independence`, and the full
self-review text, so a later audit can see at a glance that this landing had no
independent check.
