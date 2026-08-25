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

The corpus contains 30 repositories, five in each of six functional categories.

| Category | Repositories |
| --- | --- |
| Agent SDKs and orchestration | `microsoft/agent-framework`, `microsoft/semantic-kernel`, `langchain-ai/langgraph`, `openai/openai-agents-python`, `google/adk-python` |
| Multi-agent collaboration | `microsoft/autogen`, `crewAIInc/crewAI`, `camel-ai/camel`, `FoundationAgents/MetaGPT`, `OpenBMB/ChatDev` |
| Coding and software-engineering agents | `OpenHands/OpenHands`, `cline/cline`, `openai/codex`, `google-gemini/gemini-cli`, `aaif-goose/goose` |
| Browser, research, and action agents | `browser-use/browser-use`, `FoundationAgents/OpenManus`, `assafelovic/gpt-researcher`, `langchain-ai/open_deep_research`, `SamuelSchmidgall/AgentLaboratory` |
| Memory and personalisation infrastructure | `mem0ai/mem0`, `letta-ai/letta`, `getzep/graphiti`, `langchain-ai/langmem`, `langgenius/dify` |
| Evaluation, observability, and control | `promptfoo/promptfoo`, `confident-ai/deepeval`, `langfuse/langfuse`, `Arize-ai/phoenix`, `AgentOps-AI/agentops` |

A note on Dify: although Dify is primarily an application-development and workflow platform, it is retained in the memory and personalisation category because it represents stateful, personalised agent deployment at scale, with built-in conversation memory, variable/state management, and user-facing personalisation features. Its issue tracker is therefore a substantial source of memory- and personalisation-related agent discussion. Analysts who prefer a stricter reading can recategorise or exclude it using the `repo_category` column.

Because GitHub's search API can silently return zero results for renamed or moved repositories, the notebook resolves every path to its current canonical location via the GitHub API before searching, and reports any path that cannot be resolved.

## Search strategy

### Date window

Only issues **created** between `START_DATE` (2023-01-01) and `END_DATE` (2026-06-30) are collected, using GitHub's `created:` qualifier. A fixed window makes the corpus reproducible: rerunning the notebook later returns the same issue population, up to post-hoc deletions and edits on GitHub. The window covers the period from the post-ChatGPT emergence of mainstream agentic AI frameworks to shortly before collection. Both dates are parameters and can be changed in the settings cell; any change should be reported.

### Search lexicon

The lexicon contains 106 terms across eight facets, defined in the `SYCOPHANCY_TERMS` cell: Agreement and Opinion Conformity; Capitulation and Answer Flipping; Flattery and Excessive Validation; Confirmation-Biased Evidence; Agentic Action Sycophancy; Inter-Agent Deference and Consensus; Evaluator and Judge Bias; and Personalisation and Memory Sycophancy.

Terms range from direct sycophancy vocabulary ("sycophantic", "blind agreement", "excessive validation") through agent-action phrasings ("blindly follows", "accepts a bad plan", "selective retrieval", "follows the supervisor blindly") to broader discovery terms with lower specificity ("defers to", "flip flops", "stale preference"). Highly generic conversational terms with unfavourable signal-to-noise ratios ("great question", "overly positive", "lenient", "conformity", bare "one-sided", bare "second-guess") were removed or replaced with more specific phrasings following lexicon review. Because hits are candidates for downstream analysis rather than classifications, terms are not weighted or typed at collection time; the `all_search_terms` and `all_search_dimensions` columns record exactly which terms retrieved each issue, so term-level sensitivity analyses remain possible downstream.

For each repository and search term, issues are retrieved using queries of the form `repo:{repository} is:issue "{search_term}" created:{start}..{end}`. With 30 repositories this yields up to 3,180 repository-term searches before window splitting. Most multi-word phrases return zero results for most repositories, and a zero-result query costs a single request, so the marginal cost of specific low-frequency phrases is small.

An optional dry-run cell queries only the `total_count` for every repo-term pair (one request each) and saves them to `github_sycophancy_dry_run_counts.csv`, giving an exact pre-deduplication corpus size estimate before committing to full collection.

### Exhaustive retrieval and the 1,000-result cap

GitHub search returns at most 1,000 results per query. The notebook retrieves **all** pages for each query. When a query's `total_count` exceeds the cap, the date window is split in half recursively until every sub-window fits under the cap; only a single-day window that still exceeds the cap would be truncated, and this is flagged. Every executed query is written to a search log (`github_sycophancy_search_log.csv`) recording the window, `total_count`, results retrieved, whether the window was split, and truncation status, so search completeness is fully auditable.

### Deduplication

Duplicates arise across search terms and across adjacent date windows. Before removal, every search term, facet, and term type that retrieved a given issue is aggregated into `all_search_terms`, `all_search_dimensions`, and `all_search_term_types`, so multi-term hits are preserved.

## Reproducibility settings

| Setting | Value |
| --- | --- |
| `START_DATE` | 2023-01-01 |
| `END_DATE` | 2026-06-30 |
| `SEARCH_SLEEP_SECONDS` | 2 |
| Result cap handling | Recursive date-window splitting at 1,000 results |

All unique candidate issues retrieved by the search proceed to comment retrieval and thread construction. No comment-count filter or per-dimension sampling is applied.

Note on runtime: 3,180 base queries plus pagination and window splits, at a 2-second sleep per search request (the authenticated search limit is 30 requests per minute), means the search stage takes roughly two to four hours, before comment retrieval. A GitHub personal access token is required in practice.

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

## Citation

Code published alongside a manuscript on sycophancy in agentic AI systems. Please cite the manuscript when using this notebook in derived work.

## License

MIT License. See `LICENSE` for details.
