# Not essential: 118 skills that are not installed by default

These are kept for reference and as a pool to pull from. **None of them are needed for the 27 skills in [`../essential/`](../essential/) to work.** Every reference from an essential skill to one of these is an optional pointer, an opt-in branch, or explicitly non-blocking. That was checked by reading each call site (see [CURATION.md §10](../CURATION.md#10-essential-vs-not-essential-2026-10-07)).

They fall into two groups:

- **Cut for balance (48).** These have real value but overlap with an essential skill, are rarely needed, or depend on tools or accounts this setup lacks. Add one back if your work changes (see [Adding a skill back](#adding-a-skill-back)).
- **No value (70).** These are blocked, off-domain, filler, duplicates, or tied to someone else's setup.

Most of the 48 below mirror ARIS pipeline stages, so the `paper-*`, `experiment-*` and `idea-*` names here pair up with the essential ones.

## Cut for balance (48)

| Skill | Why it's not in the essential set |
|---|---|
| `ablation-planner` | `experiment-plan` already plans ablations, so this would be a second skill for the same job. |
| `arxiv` | `research-lit` calls the arXiv helper directly and `alphaxiv` reads single papers, so a separate download skill isn't needed. |
| `auto-paper-improvement-loop` | A heavy 655-line polishing loop. The essential review and audit skills cover the checks that matter. |
| `auto-review-loop` | 1,194 lines with a 710-character description, and its default and self-review routes are broken as written. `research-review` gives the critique more cheaply. |
| `autoresearch` | Tuning skills isn't recurring research work, and the installed `skill-creator` covers skill evals. |
| `brainstorming` | It forces a chain into `writing-plans` and the subagent workflow. `grill-me` covers plan stress-testing at no context cost. |
| `experiment-bridge` | An orchestrator that needs `run-experiment`, `monitor-experiment` and more. Claude can implement a plan directly and use `experiment-queue` for batches. |
| `figure-spec` | Its SVG can't be turned into a PDF or preview without `rsvg-convert` or `cairosvg`, and its labels can't hold math. TikZ is the diagram route for papers. |
| `finishing-a-development-branch` | Only the cut superpowers execution chain needs it. Claude handles merge or PR choices when asked. |
| `folder-specific-claude-and-agents-md` | Written around another person ("David") and their folders. Built-in `/init` covers CLAUDE.md files. |
| `formula-derivation` | Theory derivation isn't a recurring need in applied deep-learning work. |
| `idea-discovery` | Pulls in `research-refine-pipeline` and a gate script that's blocked here. `idea-creator` plus `novelty-check` do the core job. |
| `kill-argument` | Aimed mainly at theory papers. `research-review` already gives a hostile critique with a real second model. |
| `loop-library` | Designing agent loops is rare, and the built-in loop and schedule tools run loops. |
| `mermaid-diagram` | Its required render and check steps need Node, which isn't installed, and TikZ is the paper-diagram route. |
| `monitor-experiment` | Built for screen, Vast and Modal, and it doesn't read the queue state. Claude can check logs directly. |
| `overleaf-sync` | Needs Overleaf's paid Git integration and `rsync`, which is missing here. |
| `paper-write` | A second drafting skill. `research-paper-writing` covers drafting and `citation-audit` catches made-up references. |
| `paper-writing` | A 910-line orchestrator that needs `paper-write`, the improvement loop and more. The essential stages can be run one at a time. |
| `plain-writing` | Another person's personal style, applied to all prose, and it clashes with the paper and slide rules. |
| `proof-checker` | Formal proof audits are rare for applied deep-learning papers. |
| `proof-writer` | Writing formal theorem proofs is rare for applied deep-learning papers. |
| `receiving-code-review` | Rarely triggered. The built-in `/code-review` plus normal judgment cover it. |
| `requesting-code-review` | Built-in `/code-review` and `engineering:code-review` already give independent review. |
| `research-pipeline` | An end-to-end orchestrator that drags in four more heavy skills: too many slots for one job. |
| `research-refine` | Overlaps `idea-creator`, `research-collaborator` and `experiment-plan`, which works from your prompt when no proposal file exists. |
| `research-refine-pipeline` | A thin wrapper over `research-refine` and `experiment-plan` that only the cut `idea-discovery` needs. |
| `research-wiki` | Needs setup in every project and a working `~/.aris/repo` pointer. Nice to have, not essential. |
| `resubmit-pipeline` | Needs `rsync`, several cut skills, and `integrity-forensics` turned off. The essential audit and compile skills cover a resubmission. |
| `result-to-claim` | `experiment-audit` sets claim limits and `paper-claim-audit` checks the numbers, so its job is mostly covered. |
| `results-to-slides` | Bans the interpretation and next-steps slides that lab reviews rely on, requires image galleries from someone else's workflow, and its diagram step needs Node. |
| `run-experiment` | Its free-GPU rule and launcher don't suit a shared server. `experiment-queue` covers batches, and Claude can launch single runs. |
| `semantic-scholar` | Every keyless call was rate-limited, and `research-lit` already uses Semantic Scholar for verification. |
| `skill-security-auditor` | Only needed when adding untrusted skills, which isn't routine, and its runner script is missing. |
| `slides-polish` | An optional follow-up to `paper-slides` whose main path needs LibreOffice, which isn't installed. |
| `slurm-jobs` | No SLURM use was found in the current projects. **Add it back if you get cluster access.** It's a good skill for multi-node `torchrun`, job arrays and preemption-safe checkpointing. |
| `subagent-driven-development` | Its helper scripts are missing and it requires more cut skills. The built-in Agent and Workflow tools cover delegation. |
| `system-profile` | A generic list of profilers Claude already reaches for. |
| `systematic-debugging` | Same job as `diagnosing-bugs`, which is the stronger of the two. |
| `teach` | A learning-workspace tool. It's niche and not part of research or engineering output. |
| `tech-debt` | Mostly the same as the installed `engineering:tech-debt`. |
| `test-driven-development` | Fires on almost every coding turn and tells Claude to delete exploratory research code. `diagnosing-bugs` and `verification-before-completion` already cover regression tests. |
| `training-check` | Its scheduled checks only run while a Claude session is open and need W&B online, so they aren't reliable for week-long runs. |
| `unknowns-discovery` | Overlaps `grill-me` for surfacing open questions before work starts. |
| `vast-gpu` | Renting GPUs is occasional when you have your own server and GPU. |
| `wiki-enrich` | Only works on top of `research-wiki`, which is cut. |
| `writing-plans` | Part of the cut superpowers chain, and built-in plan mode covers planning. |
| `writing-skills` | Skill authoring is covered by the installed `skill-creator` and isn't core work. |

## No value (70)

| Skill | Why |
|---|---|
| `agent-self-scheduling` | Mostly about Codex, Pi, Hermes and cmux, with Linux-only cron/systemd. The built-in `loop` and `schedule` cover scheduling. |
| `analyze-results` | A 47-line generic checklist (table, mean ± std, trends) that Claude already follows. |
| `architecture-patterns` | Textbook Clean/Hexagonal/DDD taught through an e-commerce backend, with buggy examples. `engineering:architecture` covers it. |
| `camada-agentica` | Reads spec-driven-development (SDD) docs that don't exist here. Built-ins `/init` and `update-config` cover the rest. |
| `claims-drafting` | Patent claim drafting. Patents aren't part of this workflow. |
| `clarificar` | Nearly word for word the `grill-me` prompt, plus output wired to SDD spec files. |
| `claude-code-bash-patterns` | Teaches hook wiring and settings keys that don't exist in current Claude Code, so hooks built from it never fire. |
| `claude-hook-writer` | Its hook snippets parse `.input.*`, but the real payload is `tool_input`. Built-in `update-config` has the correct schema. |
| `code-review` | An older bundled copy of three standalone skills, and its name collides with the built-in `/code-review`. |
| `comm-lit-review` | Literature review for communications and networking venues. Off-domain; `research-lit` does the same job generally. |
| `data-visualization` | 77 lines of generic chart advice. The built-in `dataviz` skill and `paper-figure` cover plotting. |
| `deepxiv` | Needs a third-party `deepxiv` CLI that isn't installed. `alphaxiv` already does progressive paper reading keylessly. |
| `dependency-upgrade` | Targets npm/Bun/pnpm/Yarn with no pip, uv or conda coverage, and parts need a Socket account. |
| `diagramar` | Reads SDD discovery docs that don't exist, and its required validator script is missing. |
| `dispatching-parallel-agents` | Restates how Claude Code's Agent tool is already documented to be used. |
| `dse-loop` | Design-space exploration for computer architecture and EDA (gem5, yosys, verilator). It can't drive GPU training jobs. |
| `effective-agent-skills` | Restates Anthropic's skill-authoring guide, which `skill-creator` already covers. |
| `embodiment-description` | Patent specification drafting. Patents aren't part of this workflow. |
| `exa-search` | Needs a paid Exa API key with no fallback. Built-in WebSearch and WebFetch do the same job. |
| `executing-plans` | Its own text says to use `subagent-driven-development` instead whenever subagents exist, which is always true in Claude Code. |
| `feature-dev` | Relies on three custom subagents whose definitions aren't shipped. |
| `feishu-notify` | Feishu/Lark webhooks (a region-locked platform). Claude Code's push notifications cover alerts. |
| `figure-description` | Patent-drawing bookkeeping. Patents aren't part of this workflow. |
| `gemini-search` | Needs the Gemini CLI plus a key, and it asks a second model to recall papers from memory, which its own text warns can hallucinate. |
| `github-project-automation` | JavaScript-first templates written for another author (`reviewers: secondsky`, npm defaults, JS-only CodeQL). |
| `handoff-session-state` | Same job as `handoff-conversation` but tied to SDD `docs/STATE.md` files. |
| `idea-discovery-robot` | A robotics and embodied-AI variant of idea discovery (CoRL/RSS/ICRA, sim2real). Off-domain. |
| `integrity-forensics` | Requires GPT-family auditors through Codex and forbids substituting another reviewer. Not convertible to a Claude-only setup. |
| `interview-cheatsheet` | Produces Chinese-primary interview-prep notes. Not research or engineering work. |
| `invention-structuring` | Patent invention disclosure. Patents aren't part of this workflow. |
| `jurisdiction-format` | Patent filing-format compilation (CN/US/EP), with errors in its EP layout. |
| `logging-best-practices` | Generic Node/Express web-service logging advice Claude already follows. |
| `mapear` | A 26-line prompt that fills SDD templates which don't exist. |
| `mcp-dynamic-orchestrator` | The orchestrator it documents isn't shipped: its scripts import a missing `../src/`. |
| `mcp-management` | Its scripts are missing and its main path is the Gemini CLI. Built-in `claude mcp` and deferred tool loading cover the job. |
| `meta-apply` | Only lands patches from `meta-optimize`, which can't run here. |
| `meta-optimize` | ARIS maintainer infrastructure that needs usage-logging hooks this repo doesn't ship. |
| `ml-model-training` | Tutorial-level tabular sklearn/MLP code Claude writes unaided. |
| `ml-pipeline-automation` | Enterprise MLOps (Airflow, Kubeflow on Kubernetes). Airflow doesn't run natively on Windows. |
| `model-deployment` | Generic FastAPI/Docker/K8s serving boilerplate for a tabular model. Its drift detector is literally `pass`. |
| `openalex` | Doesn't deliver the affiliation and funding data it advertises. `research-lit` already queries OpenAlex. |
| `paper-illustration` | Both renderers are blocked: Gemini needs an API key, the alternate needs Codex. |
| `paper-talk` | `paper-slides` already ships the same chain as a recommended follow-up, and its export check needs LibreOffice. |
| `patent-novelty-check` | The legal patent-novelty test. `novelty-check` answers the research question. |
| `patent-pipeline` | Orchestrates the 10-skill patent chain. Patents aren't part of this workflow. |
| `patent-review` | Patent-examiner critique. Patents aren't part of this workflow. |
| `pixel-art` | Reproduces another project's README mascot style, and its preview step uses the macOS `open` command. |
| `plan-interview` | Its command and agent components are missing, and the question banks target enterprise product work. |
| `prior-art-search` | Its patent-database and Scholar searches failed when tested. `research-lit` is the working literature search. |
| `proof-orchestrator` | Built around GPT Pro and DeepSeek escalation routes that aren't available here. |
| `qzcli` | The Qizhi (启智) GPU platform, which needs an institutional account. |
| `read-all-adrs` | An unfinished personal stub ("TODO(David)"). |
| `recommendation-engine` | E-commerce recommenders. Off-domain, and its Quick Start imports modules that don't exist. |
| `recommendation-system` | E-commerce recommendation serving with Redis and A/B tests. Off-domain, and several of its fixes are broken. |
| `render-html` | Converts ARIS reports to HTML. Optional everywhere it's called, and the VS Code Markdown preview reads them fine. |
| `research-prompt` | A 43-line brief template for handing research to another product's deep-research mode. |
| `root-cause-tracing` | One idea ("trace back to the trigger") that `diagnosing-bugs` already contains. Its bisection script isn't bundled. |
| `sequential-thinking` | Needs a sequential-thinking MCP server that isn't configured, and duplicates native extended thinking. |
| `serverless-modal` | Its core training wrapper uses an outdated Modal API, and a local GPU covers its stated niche. |
| `setup-ci` | Reads gate commands from SDD docs (`docs/engineering/TESTING.md`, `AC-N` specs) that don't exist. |
| `specification-writing` | Patent specification drafting. Patents aren't part of this workflow. |
| `technical-diagrams` | Beginner syntax Claude already writes, and the Graphviz/D2/PlantUML renderers it exists for aren't installed. |
| `technical-specification` | A static web-service spec template (React/Node/Postgres) with no requirements process. |
| `test-quality-analysis` | Textbook test-smell advice, written around bun/vitest. |
| `token-usage` | Its script crashes on real transcripts (encoding error). Built-in `/usage` shows cost and usage. |
| `update-changelog` | No tagged releases in any current repo, and Claude writes changelogs unaided. |
| `using-git-worktrees` | Hands off to the built-in `EnterWorktree`/`ExitWorktree` tools, which it says to prefer anyway. |
| `using-superpowers` | Forces a skill check before every response, which is noisy with many skills installed. Native skill discovery already does this. |
| `writing-guidelines` | Just fetches Vercel's product-docs handbook at run time and applies it. |
| `writing-systems-papers` | A blueprint for systems venues (OSDI/SOSP/NSDI). Off-domain for ML venues. |

## Adding a skill back

Copy the folder into your skills directory. Nothing else changes:

```sh
cp -r ./not-essential/<skill> ~/.claude/skills/
```

To make it part of the repo's essential set, move it with `git mv not-essential/<skill> essential/<skill>`. Skills that read `../shared-references/` only resolve it from inside `essential/`, or once installed next to it in `~/.claude/skills/`.
