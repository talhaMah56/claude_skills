# Curation Log — Where These Skills Came From, and What Was Done to Them

*Covers the curation work of 2026-08-20 → 2026-08-24. All counts were verified against the filesystem on 2026-08-24 by a dedicated audit pass (§9); re-verify before relying on them if you've since added or removed skills.*

Written so you can trace any skill back to its origin without re-reading the whole process that produced this collection.

**Final counts: 145 skills in one flat directory, plus four support dirs (`shared-references/`, `tools/`, `mcp-servers/`, `_research-agora-plugins/`).** The `relevant/` + `other/` split described below was a sorting device used during the work; it was collapsed into one flat directory at the end (§8b) — that flat directory is what this repository now is. See [README.md](README.md) for install and usage.

---

## 0. About this repository

This collection started as a curation exercise inside a larger local workspace: six external skill repositories were cloned alongside a base marketplace repo, pooled into 605 candidate skills, then classified, pruned, merged, repaired, and extended down to the 145 in this repository (full account below). The source clones and the original workspace scaffolding are **not** part of this repository — they were the raw material, not the output. Section §1's table below is where each surviving skill actually came from, with links back to the originals.

## 1. Where things came from

Seven folders at the repo root supplied candidates — six git clones plus `plugins/`, which is **not** a clone but this repo's own tracked content:

| Folder | Source | Contribution to the pool |
|---|---|---|
| `plugins/` | this repo's own tracked dir (secondsky/claude-skills content) | **185** SKILL.md across 144 plugin dirs |
| `Auto-claude-code-research-in-sleep/` | wanshuiyin/Auto-claude-code-research-in-sleep ("ARIS") | **82** (after excluding three variant packs, below) |
| `awesome-agentic-ai/` | adriannoes/awesome-agentic-ai | **328** under `cursor-claude-codex/skills/`, itself mirroring several other collections |
| `research-agora/` | rpatrik96/research-agora | **4** — but these are *plugin bundles*, not skills (see note below) |
| `research-skills/` | saidwivedi/research-skills | **3** |
| `awesome-python/` | vinta/awesome-python | **3** — pooled, classified into `other/`, then dropped in the §2 prune |
| `claude-skills/` | secondsky/claude-skills (second clone of this repo) | **0 unique** — excluded, see below |

**Pool total: 185 + 82 + 328 + 4 + 3 + 3 = 605 unique skill units.**

**On `claude-skills/`** — it is a full clone of *this same repo* at the same commit. Its `plugins/` subdir is genuinely byte-identical to `plugins/` (verified with `diff -r`, exit 0, plus per-file checksums — the original pass only compared filename lists, so this was re-verified during the audit). It holds 187 SKILL.md total; the 2 outside `plugins/` are `.agents/skills/grill-me/SKILL.md` (a 7-line stub that just delegates to `/grilling`) and a `templates/skill-skeleton/` placeholder — neither is a real skill. **Do not delete this clone yet**: because 95 tracked files are deleted in the root working tree (§0), `claude-skills/` is currently the only on-disk copy of the repo's own `docs/`, `scripts/`, `schemas/`, and `templates/`.

**On `research-agora/`** — its 4 entries (`discover`, `toolkit`, `verify`, `write`) are plugin index/README pages, **not skills**. Their `SKILL.md` files have **no YAML frontmatter at all** (they start with `# Discover` etc.). They were copied into `relevant/` anyway. Consequence: Claude Code cannot discover or invoke them as skills — see §7.

**Excluded before counting anything**: three variant packs nested inside `Auto-claude-code-research-in-sleep/skills/` — `skills-codex/` (82 skills, the full canonical set repackaged for Codex CLI), `skills-codex-claude-review/` (8) and `skills-codex-gemini-review/` (15). All are re-packagings of the canonical `skills/` copies for other tools, so only the canonical set was counted.

**Source clones are otherwise untouched**, with one exception found during the audit: `awesome-agentic-ai/` has three files deleted from its working tree — `.../anthropic-cybersecurity-skills/skills/detecting-fileless-malware-techniques/SKILL.md` and two PDFs under `papers/`. That SKILL.md was in the pruned set anyway, so nothing kept was affected; `git -C awesome-agentic-ai restore .` recovers them.

### Licensing

All six sources are permissively licensed (MIT, except `awesome-python` which is CC-BY-SA-4.0 for its list content). Attribution is preserved inside the copied SKILL.md files where the originals carried it. If you ever publish this collection, the CC-BY-SA content is the one to check — but all 3 `awesome-python` entries were dropped in the prune, so nothing from it survives in `skills/`.

---

## 2. Relevant vs. other — how the split was made

Classification targeted your domain (deep learning, geoscience datasets, medical data, image preprocessing, foundation models, big-ML training) plus general PhD research workflow:

- **Whole-source overrides**: everything from `Auto-claude-code-research-in-sleep/`, `research-agora/`, and `research-skills/` went to `relevant/`. These three are overwhelmingly research/paper-pipeline tooling, so they were moved wholesale rather than classified one-by-one. Caveat: "overwhelmingly" is not "entirely" — a few ARIS skills are generic or off-domain (`pixel-art`, `mermaid-diagram`, `render-html`, `interview-cheatsheet`, `feishu-notify`) and arguably belong in `other/`.
- **Whole-collection overrides**: within `awesome-agentic-ai/`, the `anthropic-cybersecurity-skills` (108), `alirezarezvani-skills`, `bug-hunter`, `business-automation`, `matt-pocock`, and `taste-skills` sub-collections went straight to `other/` — confirmed by inspection as entirely off-domain.
- **Keyword classification** for everything else (`plugins/` and the remaining `awesome-agentic-ai/` skills), tuned after checking for false positives.

**Result: 95 → `relevant/`, 510 → `other/`.** You then asked to move `autoresearch` (a meta-skill for improving *other* Claude skills) from `other/` into `relevant/`, making it **96 / 509**.

### The pruning pass on `other/`

You asked to drop everything in `other/` with no bearing on your work, workflow, or code — not just off-domain content, but anything tied to a stack you don't use (React/Vue/Nuxt/Vercel/Cloudflare, mobile app dev, WordPress/WooCommerce, REST-API-as-a-product design, SEO/business/marketing).

**`other/` went 509 → 96 in this pass. `relevant/` was untouched by it.**

**Dropped: 413 skills** — **250** from `awesome-agentic-ai/`, **160** from `plugins/`, and all **3** from `awesome-python/`.

**Kept: 96** — generic coding/agent-workflow skills that help regardless of domain: debugging, TDD, code review, git worktrees, spec-driven development, MCP orchestration, and skill-authoring tools (relevant because you are building this collection).

Two caveats on "generic": a dozen of the survivors are written for a specific author's personal setup and name their own tooling inline (`vps-server-management`, `deep-research`, `brain-to-docs`, `distribute-skill-to-all-agents`, etc.). And `deep-research` in particular is a **DeepAPI-only** skill needing `DEEPAPI_API_KEY` and paid credits — it slipped through a prune rule that was meant to catch exactly that (§7).

### Where the survivors came from

| Source | → `relevant/` | → `other/` |
|---|---|---|
| `Auto-claude-code-research-in-sleep/` | **78** (+ `shared-references/`, §4) | 0 |
| `awesome-agentic-ai/` | **2** (`autoresearch`, `research-paper-writing`) | **50** |
| `plugins/` | **5** (`ml-model-training`, `ml-pipeline-automation`, `model-deployment`, `recommendation-engine`, `recommendation-system`) | **20** |
| `research-agora/` | **4** (but see §1 — no frontmatter) | 0 |
| `research-skills/` | **3** | 0 |
| newly written for this collection | 0 | **2** (`technical-diagrams`, `data-visualization`) |
| **Total** | **92** | **72** |

*(The `other/` column reflects the state before the second prune below, which took it to 51.)*

### Second prune of `other/`: 72 → 51

A later pass removed 21 more on a narrower rule than §2's — **not "off-domain", but "cannot ever serve research, paper-writing, or coding work."** Diagram and figure tooling was explicitly protected.

**11 removed — structurally cannot function here:**

| Skill | Why |
|---|---|
| `push-skill-to-github`, `vps-server-management`, `brain-to-docs`, `interview-style-doc-building` | Hard-wired to another author's private repo, servers, and personal doc workflow |
| `deep-research` | DeepAPI — a paid third-party product, not reachable |
| `distribute-skill-to-all-agents`, `delegating-to-agents`, `multi-ai-consultant`, `codex-goal-loop` | All assume Codex / Pi / Hermes / Gemini alongside Claude; you have only Claude |
| `harness` | Written entirely in Korean |
| `markdown-rendering` | Fixes a rendering quirk specific to the `cmux` terminal multiplexer |

**10 removed — bound to a methodology or team structure that does not apply:**

`auditar`, `revisar-pr`, `evals`, `validar`, `nova-feature`, `kickoff` are hard-wired to spec-driven-development artifacts (`specs/NNNN-*/spec.md`, `AC-N` identifiers, `SPEC_DEVIATION` markers, `docs/engineering/TESTING.md`) that only exist if you adopt the whole SDD methodology. `metricas` (team Lead Time / Throughput), `roadmap` (product roadmap with owner/value/effort), `integracoes` (survey the team's Jira/Confluence/Notion), and `delegate-my-work` (interview *employees* about office workflows) are team- and product-management artifacts.

**Deliberately kept from that same SDD pack** because they stand alone and do serve research work: `diagramar` (Mermaid C4 architecture diagrams), `mapear` (map an unfamiliar codebase → architecture assessment), `clarificar` (relentless requirements interview), `camada-agentica` (generate `CLAUDE.md` / `settings.json` / subagents), `setup-ci` (CI pipeline), `handoff-session-state`.

**Reference repair:** deleting those 10 left 9 dangling `/slash-command` pointers inside the 6 kept SDD skills, plus one in `research-prompt` (which pointed at `deep-research` for execution). All were rewritten to stand alone — e.g. `mapear`'s "suggest `/roadmap` to prioritize the mapped debts" now points at the surviving `/tech-debt`, and `research-prompt` now says to hand the prompt to any deep-research AI. Verified: **0 dangling references, 0 duplicate skill names** across all 143 skills.

All 21 were, at the time, recoverable from the source clone (`awesome-agentic-ai/cursor-claude-codex/skills/`) — that clone is no longer part of this workspace; re-clone [adriannoes/awesome-agentic-ai](https://github.com/adriannoes/awesome-agentic-ai) if you need one back.

---

## 3. Consolidations — merges, dedups, and deletions

Five operations: two true merges (3.1, 3.2), one deletion of a deprecated stub (3.3), one same-name dedup (3.4), and one from-scratch replacement of 25 boilerplate skills with 2 real ones (3.5).

Clusters that were **read and deliberately cleared as distinct** (not merged): the patent chain (`invention-structuring` → `prior-art-search` → `patent-novelty-check` → `claims-drafting` → `specification-writing` → `patent-review`), the proof chain (`proof-writer` / `proof-checker` / `proof-orchestrator`), the idea chain (`idea-creator` / `idea-discovery` / `idea-discovery-robot`), and the literature-search set (`arxiv` / `alphaxiv` / `deepxiv` / `openalex` / `semantic-scholar` / `exa-search` / `gemini-search`). Each names the others in its own description as a distinct stage or a distinct index. **Honest caveat**: "cleared as distinct" means they do different jobs, not that all are *useful to you* — three of the literature-search skills are dead without API keys you don't have (§7).

### 3.1 `auto-review-loop` ← `auto-review-loop-llm` + `auto-review-loop-minimax`

- **Why**: the same review-loop mechanic three times, differing only in which model answers as reviewer (Codex / any OpenAI-compatible API / MiniMax).
- **What changed**: added a **"Manual-Backend Provider Presets"** section to `auto-review-loop` listing 8 providers (OpenAI, DeepSeek, MiniMax, Kimi/Moonshot, ZhiPu GLM, SiliconFlow, 阿里云百炼, 零一万物) with a curl fallback and trigger-phrase aliases for the two deleted skill names, so the pre-existing `manual` backend now covers what they did.
- **Originals**: `Auto-claude-code-research-in-sleep/skills/auto-review-loop-llm/` and `.../auto-review-loop-minimax/` — both still present in the source clone.
- **Now at**: `skills/relevant/auto-review-loop/`

### 3.2 `paper-illustration` ← `paper-illustration-image2`

- **Why**: same job (AI-generated paper figures), two renderer backends — Gemini API vs. a local Codex app-server bridge. The deleted skill's own description called itself "a separate experimental alternative to `paper-illustration`."
- **What changed**: added an **"Alternate Renderer: codex-image2"** section with the preflight/generate/finalize helper commands.
- **Bug caught and fixed**: the first merge pass silently dropped the bundled helper `scripts/paper_illustration_image2.py`. Recovered from the source clone into `skills/relevant/paper-illustration/scripts/`.
- **Originals**: `Auto-claude-code-research-in-sleep/skills/paper-illustration-image2/` — still present in the source clone.
- **Now at**: `skills/relevant/paper-illustration/`
- **Cross-references updated**: `paper-writing/SKILL.md` referenced `/paper-illustration-image2` in **5 places** (lines 32, 324, 326, 330, 331); `grant-proposal/SKILL.md` referenced `/auto-review-loop-llm` in **1 place** (line 472). All updated to the merged skills' new invocation syntax.
- **Also fixed during the audit**: three *further* dangling pointers were found inside `shared-references/` and repaired — `external-cadence.md:140` listed both deleted `auto-review-loop-*` skills as live, and `integration-contract.md:334,449` pointed at the deleted `skills/paper-illustration-image2/` path.

### 3.3 `paper-poster` — deleted (not a merge)

Its own description read: `DEPRECATED — superseded by /paper-poster-html. Kept only as a redirect for muscle memory; do not use for new posters.` Removing a dead stub, nothing merged.

- **Original**: `Auto-claude-code-research-in-sleep/skills/paper-poster/` (still in the source clone). **Surviving skill**: `skills/relevant/paper-poster-html/`.

### 3.4 `systematic-debugging` — same-name dedup

- **Why**: same skill name from two sources with genuinely different content.
- **Correction to an earlier claim**: the two are **near-identical**, not "fuller vs. thinner" — 297 vs. 296 lines, `diff` = 49 lines. The `plugins/` copy was kept because its frontmatter description is fuller and its examples use runnable commands. **It was not a clean superset**: the discarded `awesome-agentic-ai/` copy had an extra ~11-line section and shipped 10 companion files (`root-cause-tracing.md`, `defense-in-depth.md`, `condition-based-waiting.md`, and others) that did not come across. Recover from the original if you want them.
- **Originals**: `plugins/systematic-debugging/skills/systematic-debugging/` and `awesome-agentic-ai/cursor-claude-codex/skills/systematic-debugging/` (both intact).
- **Now at**: `skills/other/systematic-debugging/`

### 3.5 `technical-diagrams` + `data-visualization` ← 25 boilerplate `visual-content` skills

- **Why**: all 25 skills in `awesome-agentic-ai`'s `visual-content` collection were the same auto-generated template with only the tool name swapped — no real Mermaid syntax, no real DOT syntax, nothing but filler ("Provides step-by-step guidance… follows industry best practices"). Verified by reading several side by side.
- **What changed**: wrote two replacements from scratch — `technical-diagrams` (real Mermaid/Graphviz/D2/PlantUML/ASCII syntax + a format-routing table) and `data-visualization` (Chart.js/Plotly patterns + chart-type selection).
- **Overlap note**: `skills/relevant/mermaid-diagram/` (from ARIS, 419 lines) already covers Mermaid in more depth than `technical-diagrams` (179 lines) does. The new skill earns its place for Graphviz/D2/PlantUML and the routing table; for Mermaid specifically, prefer `mermaid-diagram`.
- **Originals** (all 25, intact in the clone) under `awesome-agentic-ai/cursor-claude-codex/skills/visual-content/`: `api-flow-diagram-creator`, `architecture-diagram-creator`, `ascii-art-diagram-creator`, `chart-js-config-creator`, `d2-diagram-creator`, `data-visualization-helper`, `database-schema-visualizer`, `graphviz-dot-generator`, `infographic-outline-creator`, `mermaid-class-diagram-generator`, `mermaid-er-diagram-creator`, `mermaid-flowchart-generator`, `mermaid-gantt-chart-generator`, `mermaid-sequence-diagram-creator`, `mermaid-state-diagram-creator`, `mindmap-generator`, `network-diagram-generator`, `org-chart-creator`, `plantuml-diagram-generator`, `plotly-chart-generator`, `presentation-slide-outliner`, `process-flow-generator`, `svg-icon-generator`, `technical-diagram-analyzer`, `user-journey-mapper`.

### Net effect of the five operations

- **`relevant/` 96 → 92 skills** — 4 directories removed (`auto-review-loop-llm`, `auto-review-loop-minimax`, `paper-illustration-image2`, `paper-poster`). The 93rd directory on disk is `shared-references/`, added separately in §4; it is a dependency folder, not a skill.
- **`other/` 96 → 72 skills** — 25 `visual-content` + 1 duplicate `systematic-debugging` removed, 2 new skills added.
- **Cumulative removals from `other/`: 439** (413 pruned in §2 + 26 here).

---

## 4. Dependencies that had to be repaired

### Fixed: `shared-references/`

**57 of the 92 `relevant/` skills** reference `shared-references/<file>.md` (60 of the 82 in the source clone). The folder holds **31 files** — reviewer-routing rules, patent format templates, output protocols, integration contracts. It has no `SKILL.md`, so the original copy step correctly skipped it as "not a skill" — meaning those 57 skills would have broken on their first reference.

**Fixed**: copied `Auto-claude-code-research-in-sleep/skills/shared-references/` → `skills/relevant/shared-references/` (31 files, verified identical to source), preserving the relative path so `../shared-references/…` resolves from any skill folder.

Caveat: roughly half the references use a bare `shared-references/…` form inherited from upstream rather than `../shared-references/…`. Those don't resolve as literal relative paths from inside a skill folder — Claude will generally find the file anyway, but it isn't a literal path resolution.

### Not fixed: the external `tools/` helper scripts

**30 helper scripts** (24 `.py` + 6 `.sh`) live in `Auto-claude-code-research-in-sleep/tools/` and are **not** wired up. **34 of the 92 `relevant/` skills** invoke at least one — the heaviest used are `review_gate.py`, `extract_paper_style.py`, `save_trace.sh`, `forensics_gate.py`, `paper_illustration_image2.py`, `install_aris.sh`.

The cheap fix is a one-time `~/.aris/repo` file containing the absolute path to the clone — most skills' resolution chains check that location as a fallback. **But 9 of the 34 do not have that fallback** and will fail regardless: `citation-audit`, `experiment-audit`, `kill-argument`, `meta-optimize`, `novelty-check`, `paper-claim-audit`, `rebuttal`, `research-review`, `training-check`. Those need either a real ARIS project install (`tools/install_aris.sh`) or a manual copy of the helpers into the project.

---

## 5. Removing the Codex / second-model dependency

You have only Claude Code — no Codex CLI, no other model API. **45 of the 92 `relevant/` skills** called `mcp__codex__codex` as an independent second agent, mostly for adversarial review of Claude's own work.

### The pattern

A `self` reviewer backend, documented canonically in `auto-review-loop/SKILL.md` under **"Self-Review Backend (No Second Model)"**: re-read artifacts fresh from disk (not from memory of writing them), apply the same rubric that would have gone to Codex, argue against the work rather than confirming it, and flag anything that can't be judged impartially. Same output contract as before. In `auto-review-loop` the `self` backend is fully wired — it appears in the `REVIEWER_BACKEND` constant, the Phase A routing, and the stop-gate, not just in a documentation section.

### Breakdown of the 45

| Outcome | Count | Detail |
|---|---|---|
| Genuinely converted to self-review | **40** | A real review step now runs with Claude as reviewer. |
| Only had an unused `allowed-tools` grant stripped | **3** | `research-refine-pipeline`, `research-wiki`, `writing-systems-papers` — these listed `mcp__codex__codex` in frontmatter but **never actually called it**. Nothing was converted because there was nothing to convert. (An earlier version of this file wrongly counted these among the converted.) |
| Flagged as genuinely not convertible | **1** | `integrity-forensics` — delegates to a SHA-pinned, tamper-resistant upstream project whose design principle is "vendor nothing, fork nothing, never rewrite", and whose adjudicators are deliberately GPT-family *for* cross-family independence. Self-review would defeat its stated purpose. A note at the top of the file says so and points to the nearest substitutes (`paper-claim-audit`, `citation-audit`, `proof-checker`, human proofreading) — none of which replicate its 46-pattern coverage. |
| Left alone (not a review step) | **1** | `paper-illustration` — its Codex reference is the `codex-image2` **image-rendering** bridge, a capability Claude structurally lacks. See §7: this skill is unusable for you anyway, for a different reason. |

**Codex/manual backends are kept as a selectable option in about 27 of the 40** (`auto-review-loop`, `auto-paper-improvement-loop`, `experiment-audit`, `proof-checker`, `rebuttal`, `meta-optimize`, `research-review`, `idea-creator`, `grant-proposal`, `research-refine`, …). In the rest, the Codex path was removed outright rather than kept as a branch — so "they're still there if you get an API key" is true for most, not all.

Two special cases worth knowing:

- **`meta-apply`** — its whole design enforces cross-family verification before landing corpus patches; self-review structurally defeats that guarantee. Rather than pretending otherwise, it now carries an explicit "Self-Review Fallback" section saying so and elevating *your own* diff review as the real backstop.
- **`auto-paper-improvement-loop`** — a background agent updated its description and Constants to promise self-review, then died on an API session limit before touching the actual Round 1 / Round 2 review mechanics. Finished by hand: the Reviewer Independence Protocol and both review steps now branch by backend.

### Known leftovers (not cleaned up)

- **13 files under `skills/relevant/` still contain `mcp__codex__codex`.** Eight are legitimate: 6 converted skills where it is one branch of an explicit backend switch (`auto-review-loop`, `auto-paper-improvement-loop`, `experiment-audit`, `meta-optimize`, `proof-checker`, `rebuttal`), plus `integrity-forensics` and `paper-illustration` as described above. **The other 5 are `shared-references/` docs that the conversion pass never touched** — `reviewer-routing.md`, `reviewer-independence.md`, `acceptance-gate.md`, `review-tracing.md`, `fan-out-pattern.md`. `reviewer-routing.md` still routes everything to Codex with no `self` entry, and `acceptance-gate.md` still states that certain gates may never be self-judged — which directly contradicts what the converted skills now do. **If a skill's behaviour ever looks inconsistent with §5, this is the likely cause.**
- **`proof-orchestrator`** depends on a second model via a different route (GPT Pro handoff, DeepSeek via `mcp__llm_chat__chat`), so it fell outside the `mcp__codex__codex` sweep and was never converted. Its documented `local-executor-fallback` runs Claude-only.
- **Three skills** (`experiment-bridge`, `paper-compile`, `research-pipeline`) still mention `/codex:rescue` as an optional debugging escalation. Not review steps, and all degrade gracefully to Claude-only diagnosis — left in place deliberately.
- A handful of converted skills still contain prose asserting cross-family review as a fact (e.g. `citation-audit`'s "each layer has cross-family review"). Cosmetic, but misleading if read literally.

---

## 6. Directory naming, and a collision that was fixed

Copied skill folders are named after the SKILL.md `name:` frontmatter value. Where two skills from different sources shared a `name:`, the copy script appended `--<source>` to the **folder** name to avoid overwriting.

**This did not actually deduplicate anything.** Claude Code resolves a skill by its frontmatter `name:`, not by folder name — so two folders with different names but the same `name:` still collide at invocation time.

**Found and fixed during the audit:** `handoff--agent-orchestration/` and `handoff--awesome-agentic-ai/` both declared `name: handoff`. Resolved by giving them distinct, descriptive identities:

| Was | Now | What it does |
|---|---|---|
| `handoff--agent-orchestration/` (`name: handoff`) | `handoff-conversation/` (`name: handoff-conversation`) | Compacts the current conversation into a paste-ready handoff message |
| `handoff--awesome-agentic-ai/` (`name: handoff`) | `handoff-session-state/` (`name: handoff-session-state`) | Records/resumes session state via `docs/STATE.md` (SDD pipeline) |

Two other `--` folders had no live collision left (their collision partners were pruned in §2), so they were simplified back: `grill-me--thinking-and-docs/` → `grill-me/`, `teach--thinking-and-docs/` → `teach/`.

**Verified after the fix: zero duplicate `name:` values across all 164 skills, and every folder name now matches its frontmatter `name:`.**

---

## 7. Blockers other than Codex — skills that still won't work for you

The §5 pass only removed *Codex* dependencies. Several skills are dead-on-arrival for a Claude-only setup for unrelated reasons. This was not previously documented.

**Hard-blocked (no Claude-only fallback):**

| Skill | Needs |
|---|---|
| `gemini-search` | Gemini CLI + `GEMINI_API_KEY` |
| `exa-search` | `exa-py` + `EXA_API_KEY` (its own text says there is no fallback) |
| `qzcli` | Qizhi (启智) platform account + its CLI |
| `paper-illustration` | **Both** renderers are unavailable: default needs `GEMINI_API_KEY`, alternate needs Codex. Use `/figure-spec` or `/mermaid-diagram` instead. |
| `other/deep-research` | `DEEPAPI_API_KEY` + paid credits (slipped through the §2 prune) |

**Blocked without an account you may not have:** `vast-gpu`, `serverless-modal`, `run-experiment`, `monitor-experiment`, `experiment-bridge` (GPU compute providers — Vast.ai / Modal); `overleaf-sync` and the Overleaf paths inside `paper-plan` / `paper-write` / `paper-slides` / `resubmit-pipeline`.

**Optional, degrades gracefully:** anything referencing `feishu.json` (Feishu/Lark notifications — explicitly skipped when the config is absent), and W&B usage in `training-check` / `run-experiment`.

**Not discoverable as skills at all:** `discover`, `toolkit`, `verify`, `write` (the four `research-agora` entries) have no YAML frontmatter — they are plugin index pages. Either add frontmatter, install them the way research-agora intends (`/plugin install <name>@research-agora`), or drop them.

---

## 8. Outstanding

### Installing these skills

Superseded by [README.md](README.md) — this repository is now the flat, install-ready output of the curation process, so the install steps live there rather than being duplicated here.

### Still open

- ~~9 skills lack the `~/.aris/repo` fallback~~ — corrected in §8b: only `meta-optimize` actually needs it; the other 8 need nothing extra.
- **`shared-references/` was never Codex-converted** and contradicts §5 in places (§5 leftovers).
- **No inventory of the 413 pruned skills exists**, and the six source clones that held them are no longer on disk anywhere — recovery now means re-cloning the original repos listed in §1 and finding the skill by name. If that matters to you, generate a manifest before this becomes a problem again.
- **The four `research-agora` entries** need frontmatter or removal (§7) — still unresolved; they ship in this repo's `_research-agora-plugins/` precisely because they aren't real skills.

---

## 8b. Recovery pass — what the pruning missed

A later audit re-scanned all seven source folders against the curated collection, asking the opposite question from §2: *what did we drop that we should not have?* Six scanners plus a domain gap-critic. It surfaced two broken-infrastructure findings that mattered more than any missing skill.

### Fixed: 27 skills were half-broken

27 of the installed skills shell out to helper scripts — `research_wiki.py` (21 references), `extract_paper_style.py` (14), `run_state.py` (7), plus `arxiv_fetch.py`, `openalex_fetch.py`, `semantic_scholar_fetch.py`, `verify_papers.py`, `review_gate.py`, `save_trace.sh` and more. **None of those files existed anywhere under `skills/`.** They live in `Auto-claude-code-research-in-sleep/tools/` (42 files) and were never copied, because §4's dependency audit only looked for `shared-references`. Skills resolve `.aris/tools/X.py` → `tools/X.py` → `$ARIS_REPO/tools/X.py` and then hard-error.

So `arxiv`, `openalex`, `semantic-scholar`, `deepxiv`, `research-wiki`, `research-lit`, `wiki-enrich`, `research-pipeline`, `paper-write`, `paper-plan`, `integrity-forensics`, `experiment-queue` and ~15 others were failing at their first helper call.

**Fixed**: copied `tools/` into `skills/tools/`, making the collection self-contained. Pointing `~/.aris/repo` at `skills/` now resolves `$ARIS_REPO/tools/*`. All 42 are Python 3 stdlib or free public APIs — no keys (except `exa_search.py`, which is unusable here anyway).

**Correction to the "9 skills need a real ARIS install" claim above**: re-checked and it was overstated. `tools/install_aris.ps1` does something unrelated to those 9 — it creates per-project junctions for a different, project-local install pattern, and isn't needed once skills are installed flat into `~/.claude/skills/`. Of the 9: 7 (`citation-audit`, `experiment-audit`, `kill-argument`, `novelty-check`, `paper-claim-audit`, `rebuttal`, `research-review`) call no external script at all — they only write to a `.aris/` working directory that's created on first run. `training-check` resolves `watchdog.py` via a relative path already satisfied by the flat layout. Only `meta-optimize` genuinely needs the `~/.aris/repo` pointer above, which the fix in this section already provides.

Worth knowing independently: `watchdog.py` (485 lines) is an unattended monitor that registers training runs by tmux/screen session + GPU id and flags DEAD/STALLED/IDLE — useful for overnight foundation-model runs.

### Fixed: §5's core limitation was overstated

§5 concluded that with no second model, Claude self-review was the only option. **That was wrong.** `Auto-claude-code-research-in-sleep/mcp-servers/manual-review/` is a human-in-the-loop MCP server — pure Python stdlib, no pip install, no API key, works on Windows. It opens a local browser page; you paste the review prompt into any free model's web UI (ChatGPT free tier, DeepSeek, Gemini, Kimi, Qwen) and paste the response back. It classifies the reviewer's model family so the pipeline can verify genuine cross-family independence.

Seven skills already carry the `— reviewer: manual` wiring and `mcp__manual_review__*` in their `allowed-tools` — only the server implementing those tool names was missing. Installing it upgrades `auto-review-loop`, `auto-paper-improvement-loop`, `research-review`, `experiment-audit`, `proof-checker`, `rebuttal`, `idea-creator` and `meta-optimize` from self-judgement to real independent review, at zero cost.

**Fixed**: copied to `skills/mcp-servers/manual-review/`, registration documented in the README. This does not invalidate §5 — self-review remains the default and the honest tradeoff still applies when you don't want the copy-paste — but "no independence is possible" was too strong.

### Six skills added

Two recovered from source, four written from scratch. The scan confirmed that **none of the ~460 dropped skills covered the user's actual research domain** — the sources are weighted toward web development and research *workflow*, with nothing on geospatial formats, medical imaging, image tiling, or cluster scheduling.

| Skill | Origin | Why |
|---|---|---|
| `geospatial-raster-io` | written | rasterio/xarray/rioxarray, nodata poisoning normalization stats, CRS + resampling (nearest for labels), affine carried through tiling, Sentinel-2/Landsat specifics |
| `medical-imaging-io` | written | DICOM/NIfTI, series sorting, the HU rescale trap, CT windowing, MRI normalization differences, RAS/LPS orientation flips, isotropic resampling, MONAI pipelines, PHI de-identification |
| `image-preprocessing-and-tiling` | written | feathered tile reassembly, train-set-only normalization stats, albumentations mask handling, per-domain augmentation validity, dataloader throughput |
| `slurm-jobs` | written | sbatch directives, multi-node torchrun, job arrays, atomic checkpointing + preemption requeue |
| `diagnosing-bugs` | `awesome-agentic-ai/…/matt-pocock/engineering/` | repro-loop-first discipline; its non-determinism strategy (raise reproduction rate rather than chase a clean repro) is absent from the installed `systematic-debugging` |
| `skill-security-auditor` | `awesome-agentic-ai/…/alirezarezvani-skills/` | scans third-party skills for injection/exfiltration/prompt-injection before install — directly relevant given ~600 skills were pulled from untrusted clones |

The four written skills were each reviewed by a separate adversarial agent against the same anti-boilerplate standard that got 25 filler skills deleted in §3.5. Those reviewers verified API signatures against the real installed libraries and executed the tiling code; they fixed 9, 7, 13 and 6 issues respectively. `skill-security-auditor` shipped without the `scripts/skill_security_auditor.py` its Quick Start invokes, so that section was rewritten as runnable greps.

**Collection is now 145 skills** (139 + 6), plus four support directories: `shared-references/` (31), `tools/` (42), `mcp-servers/`, `_research-agora-plugins/`.

---

## 9. About this document

The first version of this changelog was written from memory of the work and contained **64 factual errors and omissions**, found by a dedicated audit pass (5 parallel verifiers + 2 adversarial critics, each checking claims against the actual filesystem rather than against the narrative).

Corrections applied here include: the headline count (93 → 92 skills, since `shared-references/` is not a skill); `shared-references` file count (28 → 31); how many skills depend on it ("nearly all 82" → 57); the prune breakdown (was arithmetically impossible at 326+3+130=459, actually 250+160+3=413); a garbled "96 → 96" line; the merge net-effect arithmetic that silently netted a §4 addition against §3 deletions; the `systematic-debugging` characterization (near-identical, not "fuller vs. thinner", and it dropped 10 companion files); the unverified "byte-identical" claim (since actually verified); and the conversion count (43 → 40 converted + 3 that had nothing to convert).

Three real bugs were also found and **fixed** during the audit, not merely documented: the dangling `shared-references/` pointers to deleted skills (§3.2), the duplicate `name: handoff` collision (§6), and the four folder/frontmatter name mismatches (§6).
