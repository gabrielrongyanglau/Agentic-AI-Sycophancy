# agentic-sycophancy-data-collection

Google Colab notebook and supporting files used to collect GitHub issue-thread data for a study on sycophancy in agentic AI systems. The workflow adapts the data collection pipeline from the TRACE framework study on ethical reliability in agentic AI. Coding of the collected threads is handled separately and is not part of this repository.

The notebook searches public GitHub issues created within a fixed date window across a balanced corpus of 30 agentic AI repositories, retrieves all results per query (splitting date windows where GitHub's result cap is exceeded), removes duplicate issues, retrieves public issue comments for all unique candidate issues, and constructs one structured thread text per issue with author, role, and timestamp markers.

## Nature of the corpus

The resulting dataset is a **keyword-enriched candidate corpus**: every deduplicated keyword hit proceeds to comment retrieval and thread construction. This design is appropriate for discovery, taxonomy development, and qualitative analysis of sycophancy phenomena. It is **not a prevalence sample** and cannot by itself support claims about how often sycophancy occurs in GitHub discussions, or comparative prevalence claims across repository categories or facets. Such claims would require an independent denominator, for example a random or stratified background sample of issues from the same repositories.

## Contents

| File | Purpose |
| --- | --- |
| `Sycophancy_GitHub_Data_Collection.ipynb` | The data collection notebook, intended to run in Google Colab. |
| `requirements.txt` | Python dependencies used by the notebook. |
| `.gitignore` | Excludes raw data files and secrets from the repository. |

Raw GitHub issue/comment text is not included in this repository because it contains near-raw public developer discussion. The notebook regenerates it from the GitHub API.

## Repository corpus

The corpus contains 55 repositories across six functional categories (8-10 per category), expanded from the original 30 after a pilot run showed uneven coverage.

| Category | Repositories |
| --- | --- |
| Agent SDKs and orchestration (10) | `microsoft/agent-framework`, `microsoft/semantic-kernel`, `langchain-ai/langgraph`, `openai/openai-agents-python`, `google/adk-python`, `langchain-ai/langchain`, `Significant-Gravitas/AutoGPT`, `pydantic/pydantic-ai`, `agno-agi/agno`, `mastra-ai/mastra` |
| Multi-agent collaboration (9) | `microsoft/autogen`, `crewAIInc/crewAI`, `camel-ai/camel`, `FoundationAgents/MetaGPT`, `OpenBMB/ChatDev`, `agentscope-ai/agentscope`, `agentscope-ai/AgentTeams`, `TransformerOptimus/SuperAGI`, `VoltAgent/voltagent` |
| Coding and software-engineering agents (9) | `OpenHands/OpenHands`, `cline/cline`, `openai/codex`, `google-gemini/gemini-cli`, `anthropics/claude-code`, `SWE-agent/SWE-agent`, `aider-ai/aider`, `continuedev/continue`, `TabbyML/tabby` |
| Browser, research, and action agents (10) | `browser-use/browser-use`, `FoundationAgents/OpenManus`, `assafelovic/gpt-researcher`, `langchain-ai/open_deep_research`, `SamuelSchmidgall/AgentLaboratory`, `Skyvern-AI/skyvern`, `nanobrowser/nanobrowser`, `vercel-labs/agent-browser`, `browser-use/web-ui`, `run-llama/llama_index` |
| Memory and personalisation infrastructure (8) | `mem0ai/mem0`, `letta-ai/letta`, `getzep/graphiti`, `langchain-ai/langmem`, `langgenius/dify`, `getzep/zep`, `topoteretes/cognee`, `supermemoryai/supermemory` |
| Evaluation, observability, and control (9) | `promptfoo/promptfoo`, `confident-ai/deepeval`, `langfuse/langfuse`, `Arize-ai/phoenix`, `AgentOps-AI/agentops`, `openai/evals`, `explodinggradients/ragas`, `guardrails-ai/guardrails`, `NVIDIA/NeMo-Guardrails` |

Only repositories with substantive public issue trackers are included; demonstration and sample-collection repositories were excluded. A note on Dify: although Dify is primarily an application-development and workflow platform, it is retained in the memory and personalisation category because it represents stateful, personalised agent deployment at scale. Analysts who prefer a stricter reading can recategorise or exclude it using the `repo_category` column.

Because GitHub's search API can silently return zero results for renamed or moved repositories, the notebook resolves every path to its current canonical location via the GitHub API before searching, and reports any path that cannot be resolved (any unresolved path is skipped, and the search log records exactly which repositories were searched).

## Search strategy

### Date window

Only issues **created** between `START_DATE` (2023-01-01) and `END_DATE` (2026-06-30) are collected, using GitHub's `created:` qualifier. A fixed window makes the corpus reproducible: rerunning the notebook later returns the same issue population, up to post-hoc deletions and edits on GitHub. The window covers the period from the post-ChatGPT emergence of mainstream agentic AI frameworks to shortly before collection. Both dates are parameters and can be changed in the settings cell; any change should be reported.

### Search lexicon

The lexicon contains 112 terms across eight dimensions that separate primary sycophancy from adjacent phenomena, defined in the `SYCOPHANCY_TERMS` cell: Direct Sycophancy and Agreement; Opinion Conformity and Belief Reinforcement; Capitulation and Answer Flipping; Evidence and Reasoning Conformity; Agentic Action and Plan Sycophancy; Inter-Agent Deference and Consensus; Personalisation and Memory Sycophancy; and Evaluator and Judge Conformity.

Each term carries a diagnostic level, recorded per issue in the outputs (`search_term_level`, `all_search_term_levels`) so analyses can report how many candidates came from core versus exploratory searches:

| Level | n | Role |
| --- | --- | --- |
| `core` | 34 | High-precision, direct sycophancy vocabulary (e.g. "sycophancy", "pandering", "changes answer after pushback"). |
| `behavioural` | 55 | Agentic manifestations in actions, plans, retrieval, and compliance (e.g. "fails to challenge", "accepts a bad plan", "blindly follows the supervisor"). |
| `exploratory` | 23 | Broader discovery terms (e.g. "echo chamber", "false consensus", "evaluator bias"). Hits are candidates only. |

Following a pilot run of the earlier 30-repository design, highly ambiguous terms were removed: "cherry-pick" (which returned predominantly git cherry-pick discussion; replaced by "cherry-picks evidence"), standalone "defers to", standalone "over-validation", "lenient", "judge bias", "position bias", "verbosity bias", and "reward hacking" (broader evaluation phenomena, not necessarily sycophancy; replaced by evaluator-conformity phrasings such as "judge favours its own answer" and "critic fails to challenge the answer").

For each repository and search term, issues are retrieved using queries of the form `repo:{repository} is:issue "{search_term}" created:{start}..{end}`. With 55 repositories this yields up to 6,160 repository-term searches before window splitting. Most multi-word phrases return zero results for most repositories, and a zero-result query costs a single request. A pilot run also indicated that GitHub issue search matched issue titles and bodies rather than comment text, so hits represent issues framed by the reporter in sycophancy-relevant terms; this should be stated in the Methods.

An optional dry-run cell queries only the `total_count` for every repo-term pair (one request each) and saves them to `github_sycophancy_dry_run_counts.csv`, giving an exact pre-deduplication corpus size estimate before committing to full collection.

### Exhaustive retrieval and the 1,000-result cap

GitHub search returns at most 1,000 results per query. The notebook retrieves **all** pages for each query. When a query's `total_count` exceeds the cap, the date window is split in half recursively until every sub-window fits under the cap; only a single-day window that still exceeds the cap would be truncated, and this is flagged. Every executed query is written to a search log (`github_sycophancy_search_log.csv`) recording the window, `total_count`, results retrieved, whether the window was split, and truncation status, so search completeness is fully auditable.

### Deduplication

Duplicates arise across search terms and across adjacent date windows. Before removal, every search term, facet, and term type that retrieved a given issue is aggregated into `all_search_terms`, `all_search_dimensions`, and `all_search_term_types`, so multi-term hits are preserved.

## Two-stage design and sampling frame

This repository implements Stage 1 of a two-stage design: a high-recall candidate harvest. Stage 2 (false-positive screening and multi-label coding) is handled separately, but this notebook constructs its sampling frame:

1. every thread from repositories with at most 75 candidate issues is selected;
2. repositories over the cap are downsampled to 75, stratified proportionally by search dimension (seed 42), so prolific repositories such as `anthropics/claude-code` cannot dominate the coded corpus;
3. any dimension with fewer than 25 selected threads (multi-label membership via `all_search_dimensions`) is topped up from unselected threads where available.

Selection is recorded as a `selected_for_coding` flag in `github_sycophancy_sampling_frame.csv`; no rows are dropped, so the full candidate corpus remains available. Zero-comment issues are labelled (`zero_comments`) rather than excluded, so screening can treat report-only threads separately from discussed threads.

The final-check cell prints the reporting funnel (raw keyword hits, unique issues, zero-comment issues, selected threads) for direct transfer into the Methods.

## Reproducibility settings

| Setting | Value |
| --- | --- |
| `START_DATE` | 2023-01-01 |
| `END_DATE` | 2026-06-30 |
| `SEARCH_SLEEP_SECONDS` | 2 |
| Result cap handling | Recursive date-window splitting at 1,000 results; near-cap flag at 900 |
| `MAX_PER_REPO` (sampling frame) | 75 |
| `MIN_PER_DIMENSION` (sampling frame) | 25 |
| `RANDOM_SEED` | 42 |

Note on runtime: 6,160 base queries plus pagination and window splits, at a 2-second sleep per search request (the authenticated search limit is 30 requests per minute), means the search stage takes roughly four to six hours. Comment retrieval checkpoints every 200 issues and resumes automatically, so the run can span sessions. A GitHub personal access token is required in practice.

## Requirements

The notebook is designed to run in Google Colab. Local execution is possible if the Colab-specific cells (`google.colab.userdata`, `google.colab.files`) are adapted. Python dependencies are listed in `requirements.txt`.

```text
requests
pandas
tqdm
openpyxl
```

## Setup

1. Open `Sycophancy_GitHub_Data_Collection.ipynb` in Google Colab.
2. Add `GITHUB_TOKEN` — a GitHub personal access token — through Colab Secrets (the key icon on the left sidebar).
3. Run the cells in order.

## What the notebook produces

| File | Content |
| --- | --- |
| `github_sycophancy_candidate_issues.csv` / `_clean.xlsx` | All candidate issues after deduplication, with repository category, issue author and role, and all matching search terms, facets, and term types. |
| `github_sycophancy_search_log.csv` | One row per executed search query: window, total count, results retrieved, split/collected action, truncation flag. |
| `github_sycophancy_issue_comments.csv` / `.xlsx` | All public comments retrieved for the unique candidate issues, with author, role, and timestamps. This file is the authoritative source for comment-level metadata. Comment retrieval checkpoints to this file every 200 issues and resumes automatically if the session is interrupted. |
| `github_sycophancy_issue_threads.csv` / `.xlsx` | One structured thread per issue. The `full_text` field interleaves the issue and its comments in chronological order with markers of the form `[ISSUE | author (role) | timestamp]` and `[COMMENT n | author (role) | timestamp]`. |
| `github_sycophancy_sampling_frame.csv` / `.xlsx` | The full thread corpus with a seeded `selected_for_coding` flag implementing repository caps and dimension floors for Stage-2 screening and coding. |

## Citation

Code published alongside a manuscript on sycophancy in agentic AI systems. Please cite the manuscript when using this notebook in derived work.

## License

MIT License. See `LICENSE` for details.
