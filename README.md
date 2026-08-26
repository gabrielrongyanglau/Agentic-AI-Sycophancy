# agentic-sycophancy-data-collection

Google Colab notebook and supporting files implementing the final Stage-1 harvest of a two-stage design for a taxonomy-building study of sycophancy in agentic AI systems. The workflow originated as an adaptation of the TRACE framework data collection pipeline and was revised across two pilot runs. Stage 2 (false-positive screening and multi-label coding) is handled separately; this notebook also constructs its sampling frame.

## Nature of the corpus

The resulting dataset is a **keyword-enriched candidate corpus**: every deduplicated hit proceeds to comment retrieval and thread construction. This design is appropriate for discovery, taxonomy development, and empirical mapping of sycophancy phenomena. It is **not a prevalence sample** and cannot by itself support claims about how often sycophancy occurs in GitHub discussions, or comparative prevalence claims across repository categories or dimensions.

## Contents

| File | Purpose |
| --- | --- |
| `Sycophancy_GitHub_Data_Collection.ipynb` | The data collection notebook, intended to run in Google Colab. |
| `requirements.txt` | Python dependencies used by the notebook. |
| `.gitignore` | Excludes raw data files and secrets from the repository. |

Raw GitHub issue/comment text is not included in this repository. The notebook regenerates it from the GitHub API.

## Repository corpus

The corpus contains 80 repositories across six functional categories, expanded from 30 (pilot 1) and 55 (pilot 2) to strengthen browser, research, memory, and multi-agent coverage. Inclusion criteria: an active public issue tracker with substantive discussion, direct agent functionality (tool use, planning, memory, browser interaction, coding, or multi-agent coordination), and substantial issue volume. Demonstration and sample-collection repositories are excluded.

| Category | n | Repositories |
| --- | --- | --- |
| Agent SDKs and orchestration | 13 | `microsoft/agent-framework`, `microsoft/semantic-kernel`, `langchain-ai/langgraph`, `openai/openai-agents-python`, `google/adk-python`, `langchain-ai/langchain`, `Significant-Gravitas/AutoGPT`, `pydantic/pydantic-ai`, `agno-agi/agno`, `mastra-ai/mastra`, `huggingface/smolagents`, `deepset-ai/haystack`, `FlowiseAI/Flowise` |
| Multi-agent collaboration | 14 | `microsoft/autogen`, `crewAIInc/crewAI`, `camel-ai/camel`, `FoundationAgents/MetaGPT`, `OpenBMB/ChatDev`, `agentscope-ai/agentscope`, `agentscope-ai/AgentTeams`, `TransformerOptimus/SuperAGI`, `VoltAgent/voltagent`, `openai/swarm`, `langroid/langroid`, `awslabs/agent-squad`, `kyegomez/swarms`, `microsoft/TinyTroupe` |
| Coding and software-engineering agents | 12 | `OpenHands/OpenHands`, `cline/cline`, `openai/codex`, `google-gemini/gemini-cli`, `anthropics/claude-code`, `SWE-agent/SWE-agent`, `Aider-AI/aider`, `continuedev/continue`, `TabbyML/tabby`, `RooCodeInc/Roo-Code`, `getcursor/cursor`, `plandex-ai/plandex` |
| Browser, research, and action agents | 19 | `browser-use/browser-use`, `FoundationAgents/OpenManus`, `assafelovic/gpt-researcher`, `langchain-ai/open_deep_research`, `SamuelSchmidgall/AgentLaboratory`, `Skyvern-AI/skyvern`, `nanobrowser/nanobrowser`, `vercel-labs/agent-browser`, `browser-use/web-ui`, `run-llama/llama_index`, `browserbase/stagehand`, `lavague-ai/LaVague`, `microsoft/OmniParser`, `trycua/cua`, `simular-ai/Agent-S`, `stanford-oval/storm`, `dzhng/deep-research`, `camel-ai/owl`, `bytedance/deer-flow` |
| Memory and personalisation infrastructure | 11 | `mem0ai/mem0`, `letta-ai/letta`, `getzep/graphiti`, `langchain-ai/langmem`, `langgenius/dify`, `getzep/zep`, `topoteretes/cognee`, `supermemoryai/supermemory`, `kingjulio8238/memary`, `memodb-io/memobase`, `basicmachines-co/basic-memory` |
| Evaluation, observability, and control | 11 | `promptfoo/promptfoo`, `confident-ai/deepeval`, `langfuse/langfuse`, `Arize-ai/phoenix`, `AgentOps-AI/agentops`, `openai/evals`, `vibrantlabsai/ragas`, `guardrails-ai/guardrails`, `NVIDIA-NeMo/Guardrails`, `Giskard-AI/giskard`, `comet-ml/opik` |

Because GitHub's search API can silently return zero results for renamed or moved repositories, the notebook resolves every path to its current canonical location via the GitHub API before searching and prints a resolution report: repositories supplied, successfully resolved, renamed (with canonical paths), and skipped (with HTTP status codes), plus the final number actually searched. The Methods should report the resolved count, not the supplied count of 80, and the search log records exactly which repositories were searched.

## Search strategy

### Eligibility window

Eligibility is based on **issue creation date**: only issues created between `START_DATE` (2021-01-01) and `END_DATE` (2026-06-30) are collected, using GitHub's `created:` qualifier. The window opens in 2021 because several agent frameworks and coding agents generated issue discussion before 2023. Both dates are parameters; any change should be reported.

### Search scope

Queries use `in:title,body,comments`, so an issue qualifies if any lexicon term appears in its title, body, **or any comment**. This matters because developers frequently describe the sycophancy problem only in comments under a technically-titled issue.

### Bundled queries and the two-layer lexicon

The lexicon contains 103 terms across eight dimensions (Direct Sycophancy and Agreement; Opinion Conformity and Belief Reinforcement; Capitulation and Answer Flipping; Evidence and Reasoning Conformity; Agentic Action and Plan Sycophancy; Inter-Agent Deference and Consensus; Personalisation and Memory Sycophancy; Evaluator and Judge Conformity), each split into a **core** layer (44 high-precision terms) and an **exploratory** layer (59 broader discovery terms).

Rather than one query per term, terms are searched as OR-bundles per dimension and layer, greedily packed so every query string stays under GitHub's length limit, using the advanced issues search syntax:

```text
repo:OWNER/REPO is:issue in:title,body,comments created:2021-01-01..2026-06-30 ("sycophancy" OR "sycophantic" OR "pandering" OR ...)
```

With 80 repositories this yields roughly 2,100 base bundle queries. Every retrieved issue records which bundle dimensions and layers matched it (`all_bundle_dimensions`, `all_bundle_levels`).

### Local term attribution

Bundle matching establishes that an issue matched somewhere in title, body, or comments, but not which term. After thread construction, every lexicon term is matched locally against the full thread text (case-insensitive substring, typographic apostrophes normalised), producing term-, dimension-, and level-resolved attribution per thread (`matched_terms_local`, `matched_dimensions_local`, `matched_levels_local`, `n_matched_terms`). A thread with no local match usually reflects tokenisation differences or comments deleted between search and retrieval; these are flagged, not dropped.

### Exhaustive retrieval and the 1,000-result cap

The notebook retrieves all pages for each query. When a query's `total_count` exceeds GitHub's 1,000-result cap, the date window is split in half recursively until every sub-window fits; queries at or above 900 results are additionally flagged `near_cap`. Every executed query is written to `github_sycophancy_search_log.csv` recording the bundle, window, `total_count`, results retrieved, split/collected action, and truncation status.

### Deduplication

Duplicates arise across bundles and adjacent date windows. Issues are deduplicated on repository and issue number after bundle membership is aggregated, so multi-bundle hits are preserved.

## Two-stage design and sampling frame

This repository implements Stage 1: a high-recall candidate harvest. The notebook also constructs the Stage-2 sampling frame:

The sampling frame never reduces the primary datasets: **all unique candidate issues, all comments from all eligible candidate issues, and all issue threads are always collected and saved in full.** The frame only adds selection flags for an optional balanced coding sample.

With `MAX_PER_REPO = None` (the default), no repository cap is applied: every thread is flagged `selected_for_coding` and the reserve pool is empty. With an integer cap (e.g. 40), a seeded, stratified selection is produced instead:

1. every thread from repositories at or under the cap is selected;
2. repositories over the cap are downsampled to the cap, stratified proportionally by locally-attributed primary dimension (seed 42);
3. any dimension with fewer than 30 selected threads (multi-label membership) is topped up from unselected threads where available;
4. unselected threads are flagged as a reserve pool for sensitivity analysis; no rows are dropped;
5. each repository's share of the selected set is reported against an advisory 20% cap (reported, never enforced).

Zero-comment issues are labelled (`zero_comments`) rather than excluded. The final-check cell prints the reporting funnel (raw bundle hits, unique issues, zero-comment issues, locally-attributed threads, selected threads, reserve pool) for direct transfer into the Methods.

## Reproducibility settings

| Setting | Value |
| --- | --- |
| `START_DATE` (issue creation) | 2021-01-01 |
| `END_DATE` (issue creation) | 2026-06-30 |
| Search scope | `in:title,body,comments` |
| `SEARCH_SLEEP_SECONDS` | 2 |
| `MAX_QUERY_CHARS` | 250 |
| Result cap handling | Recursive date-window splitting at 1,000; near-cap flag at 900 |
| `MAX_PER_REPO` (sampling frame, optional) | None (no cap; set an integer for a balanced coding sample) |
| `MIN_PER_DIMENSION` (sampling frame) | 30 |
| `MAX_REPO_SHARE` (advisory) | 20% |
| `RANDOM_SEED` | 42 |

Note on runtime: comment-scope matching substantially increases hit volume, so run the dry-run cell first (roughly 2,100 count-only requests, about 1.5 hours) to obtain exact per-bundle totals before the full harvest. The full search stage is expected to take several hours including pagination and window splits; comment retrieval for a 2,000-4,000 issue corpus adds several more, checkpointing to disk every 200 issues and resuming automatically across sessions. A GitHub personal access token is required in practice.

## Requirements

The notebook is designed to run in Google Colab. Local execution is possible if the Colab-specific cells (`google.colab.userdata`, `google.colab.files`) are adapted.

```text
requests
pandas
tqdm
openpyxl
```

## Setup

1. Open `Sycophancy_GitHub_Data_Collection.ipynb` in Google Colab.
2. Add `GITHUB_TOKEN` (a GitHub personal access token) through Colab Secrets, with notebook access enabled.
3. Run the cells in order; set `RUN_DRY_RUN = True` first if you want exact corpus size estimates before committing.

## What the notebook produces

| File | Content |
| --- | --- |
| `github_sycophancy_candidate_issues.csv` / `_clean.xlsx` | All candidate issues after deduplication, with repository category, issue author and role, and all matching bundle dimensions and levels. |
| `github_sycophancy_search_log.csv` | One row per executed query: bundle, window, total count, results retrieved, split/collected action, truncation and near-cap flags. |
| `github_sycophancy_issue_comments.csv` / `.xlsx` | All public comments for the unique candidate issues, with author, role, and timestamps, in order. Checkpointed every 200 issues; resumes automatically. |
| `github_sycophancy_issue_threads.csv` / `.xlsx` | One structured thread per issue. `full_text` interleaves the issue and its comments chronologically with `[ISSUE | author (role) | timestamp]` and `[COMMENT n | author (role) | timestamp]` markers, plus local term/dimension/level attribution columns. |
| `github_sycophancy_sampling_frame.csv` / `.xlsx` | The full thread corpus with seeded `selected_for_coding` and `reserve_pool` flags implementing repository caps, dimension floors, and the advisory share check for Stage-2 screening and coding. |
| `github_sycophancy_dry_run_counts.csv` | (Optional) Per-bundle total counts from the dry run. |

## Citation

Code published alongside a manuscript on sycophancy in agentic AI systems. Please cite the manuscript when using this notebook in derived work.

## License

MIT License. See `LICENSE` for details.
