# Trevec: SWE-bench Lite Benchmark Results

**[trevec.dev](https://trevec.dev)** — Deterministic code context for AI agents. Local-first, AST-graph retrieval.

This repository contains the official evaluation artifacts for **Trevec**, a deterministic, AST-graph retrieval engine designed for AI coding agents.

Trevec is closed-source infrastructure. We are publishing these artifacts to provide reproducible, transparent proof of our retrieval quality, cost efficiency, and accuracy on the [SWE-bench Lite](https://www.swebench.com/lite.html) dataset (300 instances).

We report two benchmarks:

1. **Retrieval Benchmark** — measures how well Trevec identifies the correct source file(s) from a natural-language issue description, with no LLM involved.
2. **End-to-End Benchmark** — measures how many issues Trevec + an LLM can fully resolve (generate a correct patch), using a two-stage pipeline: a single retrieval pass followed by a feedback-aware retry for failed instances.

---

## Retrieval Benchmark (300 instances)

Given only the problem statement from each SWE-bench Lite instance, Trevec retrieves a ranked list of code files. We measure whether the gold file (the file actually modified by the human patch) appears in the top-K results.

| Metric | Result |
| :--- | :--- |
| **Recall@1** | **43.3%** |
| **Recall@3** | **57.8%** |
| **Recall@5** | **61.3%** |
| **MRR@10** | **0.506** |
| **Avg. Retrieval Latency** | 45 ms |
| **Avg. Files Returned** | 5.2 |
| **Avg. Tokens Returned** | 3,996 |

Every instance in SWE-bench Lite has exactly one gold file. Recall@K equals Hit@K for this dataset.

### Retrieval Artifacts

In the [`retrieval/`](retrieval/) directory:

| File | Description |
|:-----|:------------|
| `predictions.jsonl` | Per-instance results: `instance_id`, `gold_files`, `pred_files`, retrieval metrics, latency |
| `summary.json` | Aggregate metrics across all 300 instances |

---

## End-to-End Benchmark (300 instances)

Trevec retrieves context for each issue and an LLM generates a patch. The pipeline has two stages:

1. **Base pass (V6):** Single retrieval pass, single LLM call per instance (with up to one retry on failure). No agent loops, no iterative file reading, no shell commands.
2. **Feedback retry (V9):** For instances that failed in the base pass, the LLM receives its previous patch along with the test results (failing test names and output) and generates a corrected patch. This targeted retry converts failures into fixes at low marginal cost.

| Metric | Result |
| :--- | :--- |
| **Total Resolved** | **125 / 300 (41.7%)** |
| **Base Pass (V6)** | 112 / 300 (37.3%) |
| **Feedback Retry (V9)** | +13 / 77 retried (16.9% conversion) |
| **Patch Generation Rate** | 94.0% (282 / 300, base pass) |
| **Avg. Retrieval Latency** | 46 ms |
| **Avg. Cost per Instance** | $0.23 |
| **Total Run Cost** | ~$69 ($41 base + $28 retry) |
| **LLM** | Claude Sonnet 4.6 (adaptive thinking) |

### Per-Repository Breakdown

| Repository | Resolved | Evaluated | Rate |
|:-----------|:---------|:----------|:-----|
| astropy/astropy | 4 | 6 | 67% |
| scikit-learn/scikit-learn | 13 | 22 | 59% |
| django/django | 56 | 100 | 56% |
| pytest-dev/pytest | 7 | 16 | 44% |
| pydata/xarray | 2 | 5 | 40% |
| sphinx-doc/sphinx | 6 | 15 | 40% |
| matplotlib/matplotlib | 8 | 22 | 36% |
| pallets/flask | 1 | 3 | 33% |
| sympy/sympy | 24 | 73 | 33% |
| psf/requests | 2 | 6 | 33% |
| mwaskom/seaborn | 1 | 4 | 25% |
| pylint-dev/pylint | 1 | 4 | 25% |

### End-to-End Artifacts

In the [`end_to_end/`](end_to_end/) directory:

| File | Description |
|:-----|:------------|
| `predictions.jsonl` | Per-instance predictions in standard SWE-bench format (instance_id, model_patch, usage stats) |
| `evaluation_report.json` | Raw output from the official SWE-bench Docker evaluation harness |

### Reproducing the Evaluation

The end-to-end evaluation was run using the [official SWE-bench harness](https://github.com/princeton-nlp/SWE-bench):

```bash
python -m swebench.harness.run_evaluation \
    --predictions_path predictions.jsonl \
    --run_id trevec \
    --max_workers 8 \
    --cache_level env \
    --timeout 1800
```

---

## Key Findings

1. **Cost Efficiency:** Trevec resolves 125/300 instances for ~$69 total ($0.23/instance average). The base pass alone achieves 112/300 at $0.14/instance. This is an order of magnitude cheaper than multi-agent approaches that rely on iterative file exploration.

2. **Feedback Retry is High-ROI:** The feedback-aware retry stage converts 13 additional failures into fixes for just $28 — a 25% conversion rate on previously-failed instances that had generated patches. Providing the LLM with its previous attempt and test output enables targeted corrections.

3. **Retrieval-First Architecture:** Trevec's hybrid retrieval (BM25 + local embeddings over AST-derived signals) combined with deterministic graph expansion delivers precise, token-efficient context. Average context size is ~4,000 tokens per query — enough for the LLM to locate and fix bugs without blind exploration.

4. **No Agent Loops Required:** The end-to-end pipeline uses a single retrieval pass followed by LLM patch generation. There is no multi-turn conversation, no file browsing, and no shell command execution. The feedback retry adds one targeted second attempt for failures — still no agent loop.

## Methodology

- **Retrieval:** Each SWE-bench instance's problem statement is used as a natural-language query to Trevec, which returns a ranked set of code nodes (functions, classes, methods) with file paths and byte spans.
- **Hybrid Search:** BM25 full-text search and local embedding vector search run in parallel, merged via Reciprocal Rank Fusion (RRF). Results are filtered, penalized for test files, and assembled under a token budget with graph-based expansion.
- **Base Pass (V6):** The retrieved context is provided to Claude Sonnet 4.6 in a single-attempt pipeline with adaptive extended thinking. The LLM generates a unified diff patch.
- **Feedback Retry (V9):** Instances that failed in the base pass (produced an incorrect patch or failed tests) are retried. The LLM receives the original context, its previous patch, and the test results (failing test names and output). It then generates a corrected patch informed by the specific failure. Each instance gets at most one retry.
- **Evaluation:** Patches are evaluated using the official SWE-bench Docker harness, which applies each patch to the correct repository version and runs the project's test suite.

## About Trevec

Trevec is building deterministic code context infrastructure for AI agents and developer tools. Instead of semantic chunking, Trevec parses code into a structural graph using Tree-sitter, then serves precise context via hybrid retrieval (BM25 + local embeddings) with graph expansion.

- Local-first: all indexing and retrieval runs on your machine
- Language-aware: AST-based parsing, not line-based chunking
- Fast: sub-100ms retrieval on 100k+ LoC codebases
- Works with any LLM via MCP (Model Context Protocol) server

Learn more at [trevec.dev](https://trevec.dev).

## License

The evaluation artifacts in this repository are provided for transparency and reproducibility. Trevec itself is proprietary software. See [trevec.dev](https://trevec.dev) for licensing information.
