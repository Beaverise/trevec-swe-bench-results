# Trevec: SWE-bench Lite Benchmark Results

This repository contains the official evaluation artifacts for **Trevec**, a deterministic, AST-graph retrieval engine designed for AI coding agents.

Trevec is closed-source infrastructure. We are publishing these prediction artifacts to provide reproducible, transparent proof of our retrieval quality, cost efficiency, and accuracy on the [SWE-bench Lite](https://www.swebench.com/lite.html) dataset.

## V5 Run Summary (March 2026)

In our V5 run across all 300 instances of SWE-bench Lite, Trevec achieved a **35.0% end-to-end resolve rate** using a single retrieval pass per instance — no multi-step `grep`, `find`, or `read_file` agent loops.

| Metric | Result |
| :--- | :--- |
| **Total Resolved** | **105 / 300 (35.0%)** |
| **Patch Success Rate** | 56.5% (105 / 186 patches generated) |
| **Avg. Retrieval Latency** | 62 ms |
| **Avg. Cost per Instance** | $0.22 |
| **Total Run Cost** | ~$66 |
| **LLM** | Claude Sonnet 4.6 (single-attempt) |

### Per-Repository Breakdown

| Repository | Resolved | Evaluated | Rate |
|:-----------|:---------|:----------|:-----|
| astropy/astropy | 4 | 5 | 80% |
| django/django | 48 | 75 | 64% |
| psf/requests | 4 | 6 | 67% |
| pydata/xarray | 2 | 3 | 67% |
| sphinx-doc/sphinx | 5 | 8 | 62% |
| pytest-dev/pytest | 7 | 12 | 58% |
| matplotlib/matplotlib | 5 | 9 | 56% |
| sympy/sympy | 21 | 43 | 49% |
| scikit-learn/scikit-learn | 8 | 17 | 47% |
| mwaskom/seaborn | 1 | 4 | 25% |
| pallets/flask | 0 | 3 | 0% |

### Key Findings

1. **Cost Efficiency:** By serving the exact required code context in a single 62ms retrieval pass, Trevec achieves competitive resolve rates at ~$0.22/instance — an order of magnitude cheaper than multi-agent approaches that rely on iterative file exploration.

2. **Retrieval-First Architecture:** Trevec's hybrid retrieval (BM25 + local embeddings over AST-derived signals) combined with deterministic graph expansion delivers precise, token-efficient context. Average context size was ~15,000 tokens per query — enough for the LLM to locate and fix bugs without blind exploration.

3. **Test-File Penalty:** SWE-bench issue descriptions frequently match test files that describe symptoms rather than source files containing the bug. Trevec's configurable path-based penalty system suppresses test file pollution in retrieval results.

## Artifacts

In the [`v5_lite_300/`](v5_lite_300/) directory:

| File | Description |
|:-----|:------------|
| `predictions.jsonl` | Per-instance predictions in standard SWE-bench format (instance_id, model_patch, usage stats) |
| `evaluation_report.json` | Raw output from the official SWE-bench Docker evaluation harness |

### Reproducing the Evaluation

The evaluation was run using the [official SWE-bench harness](https://github.com/princeton-nlp/SWE-bench):

```bash
python -m swebench.harness.run_evaluation \
    --predictions_path predictions.jsonl \
    --run_id trevec_v5 \
    --max_workers 8 \
    --cache_level env \
    --timeout 1800
```

## Methodology

- **Retrieval:** Each SWE-bench instance's problem statement is used as a natural-language query to Trevec, which returns a ranked set of code nodes (functions, classes, methods) with file paths and byte spans.
- **Patch Generation:** The retrieved context is provided to Claude Sonnet 4.6 in a single-attempt pipeline with adaptive extended thinking. The LLM generates a unified diff patch.
- **No agent loops:** The pipeline does not use iterative file reading, shell commands, or multi-turn agent conversations. One retrieval pass, one LLM call (with up to one retry on failure).
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
