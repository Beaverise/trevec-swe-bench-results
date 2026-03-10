# Trevec: SWE-bench Lite Benchmark Results

**[trevec.dev](https://trevec.dev)** — Deterministic code context for AI agents. Local-first, AST-graph retrieval.

This repository contains the official evaluation artifacts for **Trevec**, a deterministic, AST-graph retrieval engine designed for AI coding agents.

Trevec is closed-source infrastructure. We are publishing these artifacts to provide reproducible, transparent proof of our retrieval quality, cost efficiency, and accuracy on the [SWE-bench Lite](https://www.swebench.com/lite.html) dataset (300 instances).

We report two benchmarks:

1. **Retrieval Benchmark** — measures how well Trevec identifies the correct source file(s) from a natural-language issue description, with no LLM involved.
2. **End-to-End Benchmark** — measures how many issues Trevec + an LLM can fully resolve (generate a correct patch) in a single retrieval pass with no agent loops.

---

## Retrieval Benchmark (300 instances)

Given only the problem statement from each SWE-bench Lite instance, Trevec retrieves a ranked list of code files. We measure whether the gold file (the file actually modified by the human patch) appears in the top-K results.

| Metric | Result |
| :--- | :--- |
| **Recall@1** | **42.9%** |
| **Recall@3** | **57.8%** |
| **Recall@5** | **61.2%** |
| **MRR@10** | **0.503** |
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

Trevec retrieves context for each issue, then a single LLM call generates a patch. No agent loops, no iterative file reading, no shell commands. One retrieval pass, one LLM call (with up to one retry on failure).

| Metric | Result |
| :--- | :--- |
| **Total Resolved** | **112 / 300 (37.3%)** |
| **Patch Success Rate** | 39.7% (112 / 282 patches generated) |
| **Patch Generation Rate** | 94.0% (282 / 300) |
| **Avg. Retrieval Latency** | 46 ms |
| **Avg. Cost per Instance** | $0.14 |
| **Total Run Cost** | ~$41 |
| **LLM** | Claude Sonnet 4.6 (adaptive thinking) |

### Per-Repository Breakdown

| Repository | Resolved | Evaluated | Rate |
|:-----------|:---------|:----------|:-----|
| astropy/astropy | 4 | 6 | 67% |
| scikit-learn/scikit-learn | 12 | 22 | 55% |
| django/django | 49 | 100 | 49% |
| pydata/xarray | 2 | 5 | 40% |
| sphinx-doc/sphinx | 6 | 15 | 40% |
| pytest-dev/pytest | 6 | 16 | 38% |
| matplotlib/matplotlib | 8 | 22 | 36% |
| psf/requests | 2 | 6 | 33% |
| sympy/sympy | 21 | 73 | 29% |
| mwaskom/seaborn | 1 | 4 | 25% |
| pylint-dev/pylint | 1 | 4 | 25% |
| pallets/flask | 0 | 3 | 0% |

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

1. **Cost Efficiency:** By serving the exact required code context in a single 40ms retrieval pass, Trevec achieves competitive resolve rates at ~$0.14/instance — an order of magnitude cheaper than multi-agent approaches that rely on iterative file exploration.

2. **Retrieval-First Architecture:** Trevec's hybrid retrieval (BM25 + local embeddings over AST-derived signals) combined with deterministic graph expansion delivers precise, token-efficient context. Average context size is ~4,000 tokens per query — enough for the LLM to locate and fix bugs without blind exploration.

3. **No Agent Loops Required:** The end-to-end pipeline uses a single retrieval pass followed by a single LLM call. There is no multi-turn conversation, no file browsing, and no shell command execution. This makes the pipeline fast, reproducible, and cheap.

## Methodology

- **Retrieval:** Each SWE-bench instance's problem statement is used as a natural-language query to Trevec, which returns a ranked set of code nodes (functions, classes, methods) with file paths and byte spans.
- **Hybrid Search:** BM25 full-text search and local embedding vector search run in parallel, merged via Reciprocal Rank Fusion (RRF). Results are filtered, penalized for test files, and assembled under a token budget with graph-based expansion.
- **Patch Generation (end-to-end only):** The retrieved context is provided to Claude Sonnet 4.6 in a single-attempt pipeline with adaptive extended thinking. The LLM generates a unified diff patch.
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
