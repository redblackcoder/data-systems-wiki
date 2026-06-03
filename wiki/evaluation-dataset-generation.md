# Evaluation Dataset Generation

A methodology for constructing ground-truth evaluation datasets for enterprise metadata retrieval systems under conditions of extreme clickstream sparsity. When user interaction logs are minimal or absent, the evaluation pipeline must synthesize relevance judgments through alternative engineering approaches.

## The ground truth deficit

Enterprise data catalogs face a unique evaluation challenge: unlike consumer search engines with billions of daily interactions, internal search services generate insufficient click data to train models or measure performance. The fundamental bottleneck is acquiring a robust mapping of natural language queries to relevance-graded asset IDs.

Key constraints:
- Low query volume — statistically insignificant for implicit relevance signals
- No click tracking — interaction telemetry may not exist yet
- Structured metadata — assets are schemas, tags, and glossary terms, not narrative text
- Operational utility matters — a deprecated table with matching keywords is still irrelevant

## QnA-to-IR benchmark conversion

Existing golden QnA datasets (human-verified question-answer pairs) can be adapted into retrieval benchmarks through a multi-step pipeline:

1. **BM25 candidate pooling** — use the golden *answer* text (not the question) as a query against the BM25 index, since answers contain precise terminology matching target assets
2. **LLM entity linking** — present the LLM with the original question, golden answer, and top-N candidate metadata payloads; the model resolves which specific asset logically answers the question
3. **Hard negative retention** — retain unlinked candidates from the BM25 pool as negative examples (TREC pooling methodology), providing diverse hard negatives that share keyword overlap but are semantically incorrect

This converts text-to-text QnA pairs into text-to-asset-ID retrieval benchmarks.

## Composite relevance scoring

Enterprise search relevance is not purely semantic. Among semantically identical tables, operational signals must act as tie-breakers. The composite relevance function blends the LLM judge's semantic score with quantitative usage telemetry:

```
C_final = α · S_semantic + β · (S_semantic × U_normalized)
```

Where:
- `S_semantic` — the 0-3 rubric score from the [[llm-as-judge-search|LLM judge]]
- `U_normalized` — popularity/usage signal normalized to [0, 1] via logarithmic smoothing + min-max scaling
- `α`, `β` — hyperparameter weights (e.g., α=0.7, β=0.3 emphasizes semantics with usage as boost)

The multiplicative term `S_semantic × U_normalized` is a critical safeguard: popularity cannot rescue a semantically irrelevant document (if `S_semantic = 0`, the composite stays zero regardless of usage).

This yields a continuous relevance spectrum compatible with NDCG computation without modification.

## Continuous evaluation pipeline

The evaluation lifecycle operates as an iterative loop:

1. **Dataset construction** — ingest latest metadata schemas, run QnA adaptation + [[synthetic-query-generation|SQG]], version-control the dataset
2. **Retrieval execution** — issue queries against the search system as a black box, capture top-10 ranked results
3. **Automated grading** — dispatch query-document pairs to the [[llm-as-judge-search|LLM judge]] for semantic scoring
4. **Signal blending** — join LLM scores with popularity/usage telemetry, apply composite function
5. **Metric computation** — calculate [[search-evaluation-metrics|NDCG@10 and Hit Rate@10]]
6. **Diagnostics** — isolate degraded queries using judge CoT rationales for root-cause analysis

Dataset versioning ensures that metric fluctuations reflect actual algorithmic changes, not silent drift in test data.

## Key points
- The adapted QnA approach gives high-fidelity, domain-specific queries; SQG gives volume and long-tail coverage — use both
- Hard negatives from BM25 pooling are essential for testing discriminatory power
- Composite scoring prevents the "popular but irrelevant" failure mode while rewarding authoritative sources among semantic ties
- Version-control evaluation datasets as rigorously as production code
- The pipeline must run continuously — search quality degrades silently when the index or corpus changes

## Connections
- [[search-evaluation-metrics]] — NDCG@10 and Hit Rate@10 are computed from these generated datasets
- [[synthetic-query-generation]] — the primary method for scaling evaluation beyond human-curated queries
- [[llm-as-judge-search]] — provides the semantic relevance scores that feed composite scoring
- [[learning-to-rank]] — BM25 candidate pooling leverages the same first-stage retrieval
- [[data-catalog-search-architecture]] — evaluation validates the entire search stack end-to-end

## Sources
- [[sources/papers/Generating Search Evaluation Datasets]] — complete framework: QnA adaptation, composite relevance, pipeline orchestration
