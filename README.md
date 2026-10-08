# Claude Skills: curated for DL / geoscience / medical-imaging research

A small, install-ready set of **27 essential Claude Code skills** for a deep-learning research workflow. It covers literature, ideas, experiments, satellite and medical image data, paper writing, integrity checks, review, rebuttal, and talks, plus a lean core for debugging and agent work.

They were chosen from **145** skills, which were themselves pruned from 605 across six open-source collections. The other 118 are kept in [`not-essential/`](not-essential/), with a one-line reason for each.

The goal is balance. Every installed skill's description sits in Claude's context on every turn and competes to be picked, so each one here had to earn its slot. Nothing in `essential/` needs anything from `not-essential/`.

```
essential/                 27 skills + shared-references/ (install this)
not-essential/             118 skills, kept for reference (README lists why each one was cut)
tools/                     ARIS helper scripts several essential skills call
mcp-servers/manual-review/ free human-in-the-loop cross-model review server
_research-agora-plugins/   4 Claude Code plugins (not skills)
CURATION.md                full history: sources, prunes, fixes, and how the essential set was chosen
```

---

## 1. Install

Claude Code reads `~/.claude/skills/<skill-name>/SKILL.md`, one level deep. From inside your clone:

```sh
mkdir -p ~/.claude/skills
cp -r ./essential/*/ ~/.claude/skills/
```

This also copies `shared-references/`, which 15 of the 27 skills read as `../shared-references/…`, so it has to sit next to them. Copy whole folders, never just the SKILL.md files. Several skills ship `scripts/` or `references/`, for example `experiment-queue` and `research-collaborator`.

If you previously installed the full 145-skill collection, remove the extra folders from `~/.claude/skills/` first. Otherwise the cut skills stay loaded.

### Point the helper resolver at this repo

`research-lit`, `paper-plan`, `paper-slides`, `grant-proposal`, `alphaxiv` and `idea-creator` call Python helpers in `tools/` (`arxiv_fetch.py`, `verify_papers.py`, `extract_paper_style.py`, `research_wiki.py`, …). They find them through `~/.aris/repo`, which must contain the absolute path of this repo as plain UTF-8 text. Run from the repo root:

```sh
# Git Bash / macOS / Linux
mkdir -p ~/.aris && printf '%s' "$(pwd)" > ~/.aris/repo
```

```powershell
# PowerShell. Don't use `echo >`: Windows PowerShell writes UTF-16, which the skills can't read.
New-Item -ItemType Directory -Force "$HOME\.aris" | Out-Null
[IO.File]::WriteAllText("$HOME\.aris\repo", (Get-Location).Path)
```

Without it, `research-lit` still searches but tags every paper `[UNVERIFIED]`, the `— style-ref:` option of `paper-plan` / `paper-slides` / `grant-proposal` stops with an error, and the optional research-wiki ingest is skipped. All helpers are Python 3 standard library or free public APIs, with no keys.

### Optional: real cross-model review, free

`research-review`, `rebuttal`, `experiment-audit` and `idea-creator` default to Claude reviewing its own work. Add `— reviewer: manual` to get a genuinely independent second opinion instead. The `manual-review` server opens a local page; you paste the prompt into any free chatbot (ChatGPT free tier, DeepSeek, Gemini, Kimi, Qwen) and paste the answer back.

```sh
claude mcp add manual-review -s user -- python3 "$(pwd)/mcp-servers/manual-review/server.py"
```

On Windows, if `claude` isn't on PATH, add the same entry to `mcpServers` in `~/.claude.json`, with full paths to `python.exe` and `server.py`. Restart Claude Code afterwards; MCP servers only load at startup.

---

## 2. The 27 skills, mapped to the workflow

You mostly don't call these by name. Claude picks a skill from how you phrase the request, so "check whether this idea has been done" triggers `novelty-check`. Two are manual-only: `/grill-me` and `/handoff-conversation`.

**Literature**
- `research-lit`: a multi-source survey that checks your own library first, then verifies each paper against arXiv, CrossRef and Semantic Scholar, and tags anything it can't verify.
- `alphaxiv`: "explain this paper". It reads a keyless overview and steps up to the full text or LaTeX source only when needed.

**Ideas**
- `idea-creator`: finds gaps from several angles, argues against its own ideas, and writes a ranked report that also lists the rejected ones.
- `novelty-check`: splits an idea into claims, searches each one several ways, and argues against its own novelty case.
- `research-collaborator`: requires a kill criterion and a written prediction before each run, and carries a catalogue of silent deep-learning bugs, including segmentation and ViT ones.

**Experiments and compute**
- `experiment-plan`: claim-driven experiment blocks with pass/fail criteria, ablations, and stop/go milestones.
- `experiment-queue`: an SSH job queue for multi-seed and multi-config grids on a shared GPU server. It fills GPUs as they free up, retries out-of-memory jobs, chains phases, and survives SSH drops. Write manifests in the phases format from `scripts/build_manifest.py`.
- `experiment-audit`: catches fake ground truth, scores normalized by the model's own output, and phantom numbers in evaluation code.

**Your data**
- `geospatial-raster-io`: GeoTIFF, NetCDF and Zarr traps, including nodata poisoning your statistics, nearest-neighbour resampling for labels, the Sentinel-2 offset, Landsat scale factors and QA bits, and GDAL on Windows.
- `medical-imaging-io`: DICOM and NIfTI, the Hounsfield-unit rescale trap, slice ordering, RAS vs LPS orientation, MONAI transform order, and safe de-identification.
- `image-preprocessing-and-tiling`: tiling and blended stitching, train-only normalization statistics, albumentations mask handling, and Windows DataLoader seeding.

**Writing**
- `paper-plan`: an outline built around a claims-evidence matrix, with venue page rules.
- `research-paper-writing`: section templates for CV and MICCAI-style papers, with a rejection-risk self-review.
- `paper-figure`: one reproducible script per figure, vector PDFs at column width, and captions checked against the data.
- `paper-compile`: latexmk plus venue page counts, font embedding, anonymity scans, and a leftover-marker sweep.

**Integrity and review**
- `paper-claim-audit`: traces every number in the paper back to the raw result files.
- `citation-audit`: confirms each reference is real and actually supports the sentence citing it.
- `research-review`: a tough venue-style critique that can use the manual-review server as a real second model.

**After submission, talks, and funding**
- `rebuttal`: breaks reviews into single issues, keeps every fact traceable, and fits the reply within the venue's character limit.
- `paper-slides`: a Beamer PDF plus an editable PPTX with speaker notes and a talk script.
- `paper-poster-html`: an HTML/CSS conference poster with measured layout checks. It needs a one-time free Playwright install.
- `grant-proposal`: agency-specific structure and a panel-style review. Name the agency, such as NSF or ERC, to avoid the Japanese default.

**Engineering and agent work**
- `diagnosing-bugs`: a reliable reproduction before any hypothesis, then ranked testable hypotheses, bisection, and handling for flaky failures.
- `web-debug-search`: a targeted search for install and version errors (CUDA, PyTorch, GDAL, MONAI) that checks whether a fix actually shipped.
- `verification-before-completion`: run the check fresh and read the output before claiming anything works.
- `/grill-me`: questions a plan one decision at a time until it holds up.
- `/handoff-conversation`: writes a paste-ready handoff so a fresh session can continue the work.

### A typical run

```
"Find recent work on X and check if my angle is novel"     → research-lit, novelty-check
"Plan the experiments and ablations"                        → experiment-plan
"Run this 3-seed × 4-config grid on the server"             → experiment-queue
"Did the results actually hold up?"                         → experiment-audit
"Outline the paper, make the figures, draft the methods"    → paper-plan, paper-figure, research-paper-writing
"Check every number and citation, then review it"           → paper-claim-audit, citation-audit, research-review
"Compile it for submission"                                 → paper-compile
"Write the rebuttal" / "make the slides and poster"         → rebuttal, paper-slides, paper-poster-html
```

---

## 3. Not included, and when to add something back

[`not-essential/README.md`](not-essential/README.md) lists all 118 with a reason each, split into **cut for balance** (real value, but overlapping, rare, or blocked here) and **no value** (blocked, off-domain, filler, or tied to someone else's setup). The cuts most likely to matter:

- **`slurm-jobs`**: add it if you get HPC cluster access.
- **`slides-polish`**: add it once LibreOffice is installed.
- **`mermaid-diagram`**: add it once Node is installed.
- **`paper-writing`**: the end-to-end paper pipeline. It also needs `paper-write` and `auto-paper-improvement-loop`.
- **`proof-writer` / `proof-checker`**: add them for theory-heavy papers.
- **`vast-gpu`**: add it if you rent GPUs.

To add one, run `cp -r ./not-essential/<skill> ~/.claude/skills/`.

Also deliberately absent: patents (the 10-skill chain is in `not-essential/`), and anything that needs Codex, Gemini or Exa API keys. `_research-agora-plugins/` holds four Claude Code **plugins**, not skills. Install those with `/plugin marketplace add rpatrik96/research-agora`, then for example `/plugin install verify@research-agora`.

---

## Where these skills came from

| Source | License | Contribution |
|---|---|---|
| [wanshuiyin/Auto-claude-code-research-in-sleep](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep) ("ARIS") | MIT | The backbone: literature, ideas, experiments, the paper pipeline, review and rebuttal, plus `tools/`, `shared-references/` and `mcp-servers/manual-review/`. 18 of the 27 essential skills come from here. |
| [secondsky/claude-skills](https://github.com/secondsky/claude-skills) | MIT | Generic ML and engineering skills. None made the essential set. |
| [adriannoes/awesome-agentic-ai](https://github.com/adriannoes/awesome-agentic-ai) | MIT | `research-paper-writing`, `verification-before-completion`, `diagnosing-bugs`, `grill-me`, `handoff-conversation`, and agent-workflow skills. |
| [rpatrik96/research-agora](https://github.com/rpatrik96/research-agora) | MIT | 4 plugin bundles in `_research-agora-plugins/`. |
| [saidwivedi/research-skills](https://github.com/saidwivedi/research-skills) | MIT | `research-collaborator`. |

`geospatial-raster-io`, `medical-imaging-io`, `image-preprocessing-and-tiling` and `slurm-jobs` were written from scratch for this collection, and each was checked against the real library APIs. [CURATION.md](CURATION.md) has the full account of every merge, drop and fix, and how the essential set was chosen (§10).

## License

Redistributed under the original **MIT** licenses of the sources above. Attribution is preserved in each SKILL.md where the original carried it. The four skills written for this collection and this documentation are original work. Nothing from the CC-BY-SA-licensed `awesome-python` survived curation.
