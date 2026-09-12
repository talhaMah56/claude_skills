# Claude Skills — curated for DL / geoscience / medical-imaging research

A flat, install-ready collection of **145 Claude Code skills** for a deep-learning research workflow: literature search, experiment planning, ML engineering, geospatial and medical image preprocessing, paper writing, review, and submission. Pulled from six open-source skill collections, aggressively pruned down to what's actually useful for this domain, patched where things were broken, and extended with domain skills that didn't exist anywhere yet.

If you're reading this on GitHub: clone it, run the install command in §1, and you have all of this in Claude Code. Full step-by-step reasoning for every decision below — what got kept, what got cut, what was found broken and fixed — lives in [CURATION.md](CURATION.md).

---

## Where these skills came from

Six repositories were pulled and pooled (**605 skills** to start), then classified and pruned down to what actually serves this domain, then merged/deduped/repaired, then extended. See [CURATION.md](CURATION.md) for the full account of that process — every merge, every drop, every fix, with reasons.

| Source | License | What was taken |
|---|---|---|
| [wanshuiyin/Auto-claude-code-research-in-sleep](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep) ("ARIS") | MIT | The backbone of this collection — **78 skills**: literature search, idea discovery, experiment planning/auditing, the full paper-writing pipeline, patents, review/rebuttal loops. Plus its 42 helper scripts (`tools/`), its 31 shared protocol docs (`shared-references/`), and its human-in-the-loop cross-model review server (`mcp-servers/manual-review/`). |
| [secondsky/claude-skills](https://github.com/secondsky/claude-skills) | MIT | **~25 skills** from its plugin marketplace — `ml-model-training`, `ml-pipeline-automation`, `model-deployment`, `recommendation-engine`/`-system`, plus generic engineering practice skills (debugging, code review, dependency upgrades, logging, architecture patterns). |
| [adriannoes/awesome-agentic-ai](https://github.com/adriannoes/awesome-agentic-ai) | MIT | **~52 skills** — research-paper-writing, spec-driven-development tooling (`clarificar`, `mapear`, `diagramar`, `camada-agentica`, `setup-ci`), agent-workflow discipline (writing-plans, dispatching-parallel-agents, verification-before-completion), plus `diagnosing-bugs` and `skill-security-auditor` recovered on a later pass. The bulk of this repo (cybersecurity/pentest, business automation, frontend/design, TypeScript-specific tooling — ~275 skills) was off-domain and dropped. |
| [rpatrik96/research-agora](https://github.com/rpatrik96/research-agora) | MIT | 4 research-assistant plugin bundles (`discover`, `toolkit`, `verify`, `write`) — install these as Claude Code *plugins*, not as skills; see §5. |
| [saidwivedi/research-skills](https://github.com/saidwivedi/research-skills) | MIT | 3 skills — `research-collaborator`, `results-to-slides`, `token-usage`. |
| [vinta/awesome-python](https://github.com/vinta/awesome-python) | CC-BY-SA-4.0 | Considered, nothing kept — its only "skills" were meta-tools for maintaining that repo's own README list. |

Six skills have no upstream source — written from scratch for this collection because nothing in any of the above covered them: `geospatial-raster-io`, `medical-imaging-io`, `image-preprocessing-and-tiling`, `slurm-jobs` (new), plus `diagnosing-bugs` and `skill-security-auditor` (recovered from `awesome-agentic-ai` on a later audit pass, see below).

**What "curated" actually involved** (full detail in CURATION.md):
- **Classified** all 605 against this specific research domain, then pruned to ~190, then further pruned to what genuinely helps research/paper/code work (145 final).
- **Merged** overlapping skills where two were doing the same job (e.g. three review-loop variants differing only by reviewer backend → one skill with a provider-preset table).
- **Deduplicated** by frontmatter name where two sources shipped a skill under the same identifier — Claude Code resolves by `name:`, not folder name, so a naming collision silently breaks one of the two.
- **Fixed dependencies**: 27 skills called helper scripts that were never copied from the source repo — now they are (`tools/`); a shared-protocols folder 57 skills read from was missing the same way — now fixed (`shared-references/`).
- **Removed the Codex dependency**: 45 skills called out to Codex CLI as an independent reviewer. All were converted to also support Claude self-review or a free human-in-the-loop cross-model server (`mcp-servers/manual-review/`) — see §1.3 and CURATION.md §5/§8b.
- **Authored** the six skills nothing upstream covered, then reviewed each with a separate adversarial pass checking real library APIs and executing the example code.

---

## 1. Install

Claude Code scans `~/.claude/skills/<skill-name>/SKILL.md` — one level deep, flat. This repo is already laid out to match. From inside your clone of this repo:

```sh
mkdir -p ~/.claude/skills
cp -r ./*/ ~/.claude/skills/
```

**Use `cp -r` on whole folders — never a `find … -name SKILL.md` copy.** Many skills ship `scripts/`, `references/`, and `assets/` alongside their SKILL.md, and 57 of them read from `shared-references/`. A SKILL.md-only copy silently breaks all of it. `shared-references/` must stay a sibling of the skills — that's what `../shared-references/…` resolves to. (`_research-agora-plugins/` is nested two levels deep on purpose, so the copy above and Claude's own scanner both skip it — see §5.)

### Second step: the ARIS helper scripts (required — 27 skills are dead without them)

27 of these skills shell out to Python/shell helpers (`research_wiki.py`, `extract_paper_style.py`, `run_state.py`, `arxiv_fetch.py`, `verify_papers.py`, …). Those helpers ship **inside this repo** at `tools/`, so point the resolver at your clone:

```sh
mkdir -p ~/.aris
echo "$(pwd)" > ~/.aris/repo
```

Run that from inside the cloned repo directory — it records the absolute path so `$ARIS_REPO/tools/<script>.py` resolves from any project you're working in later. All 42 helpers are Python 3 stdlib or free public APIs (arXiv, OpenAlex, Semantic Scholar) — **no API keys** (the sole exception is `tools/exa_search.py`, which needs an Exa key; ignore it).

Without this, `arxiv`, `openalex`, `semantic-scholar`, `research-wiki`, `research-lit`, `wiki-enrich`, `paper-write`, `paper-plan`, `experiment-queue` and ~18 others fail at their first helper call.

You do **not** need to run `tools/install_aris.ps1` / `.sh` — that installs a different, per-project junction pattern that doesn't apply once you've done the flat install above.

### Third step (optional but high value): real cross-model review, free

Self-review (Claude checking its own work) is the default here since most people running this won't have a second model API key — but it has no independence guarantee. `mcp-servers/manual-review/` closes that gap for free: a human-in-the-loop MCP server that opens a local browser page, you paste the review prompt into any free model's web UI (ChatGPT free tier, DeepSeek, Gemini, Kimi, Qwen), and paste the response back. It classifies the reviewer's model family so the pipeline can verify it actually differed from Claude.

```sh
claude mcp add manual-review -s user -- python3 "$(pwd)/mcp-servers/manual-review/server.py"
```

(On Windows, if `claude` isn't on PATH in your terminal, add the same entry directly to the `mcpServers` object in `~/.claude.json` instead, with the full path to your `python.exe` and the full path to `server.py`. Restart Claude Code fully after either method — MCP servers only load at startup.)

Pure Python stdlib, no pip install required. Then invoke any review skill with `— reviewer: manual`:

```
/auto-review-loop "my paper" — reviewer: manual
/proof-checker "theory.tex" — reviewer: manual
```

This upgrades eight skills — `auto-review-loop`, `auto-paper-improvement-loop`, `research-review`, `experiment-audit`, `proof-checker`, `rebuttal`, `idea-creator`, `meta-optimize` — from Claude judging its own work to a genuinely independent second opinion. For anything you're about to submit, it's worth the copy-paste.

### Verify it worked

Start Claude Code and run `/context` — installed skills show up there. Or just ask for something a skill covers and see if it fires.

---

## 2. How you actually use them

**You mostly don't invoke them by name.** Claude keeps every installed skill's `name` + `description` in context and picks the matching one from how you phrase a request. Saying *"check whether this idea has been done before"* is enough to trigger `novelty-check`; you don't type `/novelty-check`.

Two consequences worth knowing:

- **Phrasing matters more than memorising names.** Describe the task; the descriptions were written to match natural phrasing.
- **You can still force one** by typing `/skill-name` when you want a specific one, or when Claude picks the wrong neighbour (the `paper-*` family overlaps a lot).

**Standing cost:** all 145 descriptions sit in context every turn — roughly 8–12K tokens. That's fine on a large context window, but it does make skill selection noisier. If Claude starts picking odd skills, install a subset instead (§6).

---

## 3. Mapped to your workflow

### Your data — the domain layer
These six were written or recovered specifically for this collection because **no source repo covered them**:

- `geospatial-raster-io` — GeoTIFF/NetCDF/Zarr via rasterio + xarray/rioxarray, windowed reads, CRS and reprojection (nearest for labels, bilinear for continuous — using bilinear on a class raster invents classes that don't exist), affine transforms carried through tiling, burning shapefiles into label masks, Sentinel-2/Landsat scale factors and cloud masks. Leads with the nodata trap: a `-9999` sentinel silently poisons your mean/std and every downstream normalization.
- `medical-imaging-io` — DICOM/NIfTI via pydicom/nibabel/SimpleITK, correct series sorting (why `InstanceNumber` alone is unreliable), the `RescaleSlope`/`RescaleIntercept` trap (raw pixels are *not* Hounsfield Units), CT windowing presets, why MRI needs entirely different normalization, RAS vs LPS orientation flips that are invisible on a brain scan but destroy laterality labels, isotropic resampling, MONAI transform pipelines, and PHI de-identification including burned-in pixel annotations and defacing.
- `image-preprocessing-and-tiling` — tiling big scenes, feathered/weighted reassembly instead of naive overwrite, train-set-only normalization stats (and re-applying them identically at inference — a top source of train/serve skew), albumentations with correct mask handling, augmentations that are wrong per domain, foreground-biased sampling for sparse targets, dataloader throughput, and a real verification section.
- `slurm-jobs` — sbatch directives that actually matter, multi-node `torchrun` under SLURM, job arrays for sweeps, and the checkpointing/requeue discipline that separates losing a week from not (atomic writes, saving optimizer+scheduler+scaler+RNG state, `--signal=B:USR1@120` preemption traps).
- `diagnosing-bugs` — build a tight deterministic repro loop *before* hypothesizing; the non-determinism section (raise reproduction rate rather than chase a clean repro) maps directly onto flaky dataloaders and losses that NaN one run in twenty.
- `skill-security-auditor` — scan a third-party skill for injection/exfiltration/prompt-injection before installing it. Relevant because this whole collection was pulled from six untrusted clones — run it on anything else you add.

### Finding and framing work
`arxiv` · `semantic-scholar` · `openalex` · `alphaxiv` · `deepxiv` — different literature indexes; arXiv preprints vs published venues vs open citation graphs. `research-lit` and `comm-lit-review` run structured surveys. `idea-creator` / `idea-discovery` generate and rank directions, `novelty-check` verifies nobody's done it, `research-wiki` + `wiki-enrich` build a durable per-paper knowledge base.

> *"Survey recent work on self-supervised pretraining for multispectral satellite imagery, then check whether my angle is novel."*

### Planning and running experiments
`experiment-plan` → `ablation-planner` (design the ablations reviewers will demand) → `run-experiment` / `experiment-queue` → `monitor-experiment` → `training-check` (catches NaN, divergence, plateau) → `analyze-results` → `experiment-audit` (adversarial integrity check: fake ground truth, normalization tricks, phantom results) → `result-to-claim` (turns results into paper-ready claims).

`experiment-bridge` connects a validated idea to a concrete experimental setup. `system-profile` benchmarks your hardware.

> *"Plan the ablations for this architecture, then audit whether the results actually support what I want to claim."*

### ML engineering
`ml-model-training` (PyTorch/TF/Keras training loops, hyperparameters), `ml-pipeline-automation` (MLOps, MLflow, experiment tracking), `model-deployment`, `recommendation-engine` / `recommendation-system`.

### Writing the paper
`paper-writing` is the full pipeline — narrative report → submission-ready PDF. Or run the stages: `paper-plan` (outline) → `paper-figure` (plots from results) → `paper-write` (LaTeX prose) → `paper-compile` (build + fix LaTeX errors).

Figures: `figure-spec` for deterministic architecture/pipeline diagrams (JSON → editable SVG — **best default**), `mermaid-diagram` for flowcharts, `technical-diagrams` for Graphviz/D2/PlantUML, `data-visualization` for charts. Avoid `paper-illustration` (§5).

Correctness: `paper-claim-audit` (do the numbers in the text match the result files?), `citation-audit` (are the references real and correctly used?), `proof-writer` / `proof-checker` / `proof-orchestrator` for theory, `formula-derivation`.

> *"Draft the methods section from PAPER_PLAN.md, then audit every number in it against results/."*

### Review before you submit
`auto-review-loop` — the core one. Review → implement fixes → re-review, up to 4 rounds, until it scores ≥6/10 with a ready/almost verdict. `auto-paper-improvement-loop` does the same for writing quality specifically. `kill-argument` runs an adversarial attack on your framing (best for theory/scope claims). `research-review` and `paper-claim-audit` are narrower checks.

By default these run in **self-review mode** — Claude re-reads its own output fresh and argues against it. Real, but no independence guarantee unless you set up the free manual-review server (§1.3) — see CURATION.md §5/§8b.

### After submission
`rebuttal` (parse reviews → coverage-checked response under venue char limits), `resubmit-pipeline`, `research-refine`.

### Talks and posters
`paper-slides` (Beamer + PPTX with speaker notes) → `slides-polish` (per-page layout fixes). `paper-talk` is the end-to-end version. `paper-poster-html` builds conference posters as HTML/CSS → print-ready PDF. `results-to-slides` for quick result decks.

### Funding and IP
`grant-proposal` (KAKENHI/NSF/NSFC/ERC/DFG/SNSF/ARC/NWO formats). Patents: `invention-structuring` → `prior-art-search` → `patent-novelty-check` → `claims-drafting` → `specification-writing` → `patent-review`.

### Writing code (any project)
`systematic-debugging`, `diagnosing-bugs`, `root-cause-tracing`, `test-driven-development`, `test-quality-analysis`, `code-review`, `requesting-code-review` / `receiving-code-review`, `tech-debt`, `dependency-upgrade`, `logging-best-practices`, `architecture-patterns`, `using-git-worktrees`, `verification-before-completion`.

### Working with Claude itself
`effective-agent-skills` + `writing-skills` (how to write good skills — directly relevant to maintaining this collection), `skill-security-auditor`, `claude-hook-writer`, `claude-code-bash-patterns`, `mcp-management`, `folder-specific-claude-and-agents-md` (per-directory CLAUDE.md), `handoff-conversation` (compact a long session into a paste-ready handoff), `unknowns-discovery`, `grill-me` / `clarificar` (interrogate a vague plan until it's concrete), `autoresearch` (auto-tune a skill's prompt against binary evals).

---

## 4. A realistic end-to-end run

```
1.  "Find recent work on X and check if my angle is novel"     → arxiv, research-lit, novelty-check
2.  "Design experiments + the ablations reviewers will want"   → experiment-plan, ablation-planner
3.  … run training …                                            → run-experiment, training-check
4.  "Did the results actually hold up?"                         → analyze-results, experiment-audit
5.  "Turn these into paper claims"                              → result-to-claim
6.  "Write the paper"                                           → paper-writing (or the stages)
7.  "Make the architecture figure"                              → figure-spec
8.  "Check every number and citation"                           → paper-claim-audit, citation-audit
9.  "Review it like a NeurIPS reviewer and fix what's broken"   → auto-review-loop
10. "Make the slides and poster"                                → paper-talk, paper-poster-html
```

Steps 4, 8 and 9 are where the real value is — they catch the things that get papers rejected.

---

## 5. What won't work, and why

| Skill | Blocker |
|---|---|
| `gemini-search` | Needs Gemini CLI + `GEMINI_API_KEY` |
| `exa-search` | Needs `exa-py` + `EXA_API_KEY`; no fallback |
| `qzcli` | Needs a Qizhi (启智) platform account |
| `paper-illustration` | **Both** renderers need something you may not have — default needs `GEMINI_API_KEY`, alternate needs Codex CLI. **Use `figure-spec` or `mermaid-diagram` instead.** |
| `integrity-forensics` | Requires a GPT-family model for genuine cross-family audit; cannot be converted to self-review without defeating its purpose |
| `vast-gpu`, `serverless-modal`, `run-experiment`, `monitor-experiment` | Need a Vast.ai / Modal account (only the remote-compute paths) |
| `overleaf-sync` | Needs an Overleaf account |
| `feishu-notify` | Feishu/Lark webhook — silently skipped when absent, harmless |

**`_research-agora-plugins/`** — `discover`, `toolkit`, `verify`, `write` are Claude Code **plugins**, not skills: they ship `commands/` and `agents/`, and their SKILL.md is a README index. They will not work if copied into `~/.claude/skills/`. Install them the intended way instead:

```
/plugin marketplace add rpatrik96/research-agora
/plugin install verify@research-agora
```

`verify` in particular is worth it — it adds `/paper-references`, `/paper-verify-experiments`, `/pre-submission-audit`, and a statistical-validator agent.

---

## 6. If it gets noisy

145 skills is a lot of description text competing for attention. If Claude starts picking odd skills, install a subset instead — copy only the folders you want, from inside your clone:

```sh
for s in arxiv semantic-scholar novelty-check experiment-plan ablation-planner \
         training-check analyze-results experiment-audit result-to-claim \
         paper-plan paper-write paper-figure paper-compile figure-spec \
         paper-claim-audit citation-audit auto-review-loop kill-argument \
         paper-slides paper-poster-html rebuttal grant-proposal \
         ml-model-training systematic-debugging diagnosing-bugs \
         technical-diagrams data-visualization handoff-conversation \
         geospatial-raster-io medical-imaging-io image-preprocessing-and-tiling \
         slurm-jobs shared-references tools; do
  cp -r "./$s" ~/.claude/skills/
done
```

That's the ~32 that carry most of the value for this domain. Add more as you find gaps — and keep `shared-references` and `tools` in the list, they are not optional.

---

## License

The bulk of this collection is redistributed under its original **MIT** licenses (see the source table above); attribution is preserved in each SKILL.md where the original carried it. The six domain skills authored for this collection (`geospatial-raster-io`, `medical-imaging-io`, `image-preprocessing-and-tiling`, `slurm-jobs`) and this documentation are original work. No content from the CC-BY-SA-licensed `awesome-python` survived the curation pass.
