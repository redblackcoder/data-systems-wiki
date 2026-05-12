# Learning to Rank

A multi-stage approach to ordering search results using ML models trained on relevance judgments. In data catalog search, this means moving from broad candidate retrieval (hundreds of results) to precise, personalized ordering that accounts for metadata quality, lineage, usage patterns, and semantic intent.

## How it works

### First stage: Hybrid retrieval (high recall)

Narrow the full corpus to ~hundreds of candidates using two complementary methods:

| Method | Strengths | Weaknesses |
|---|---|---|
| **BM25 (lexical)** | Exact technical terms, column names, table IDs | Misses synonyms and natural language intent |
| **Vector search (semantic)** | Captures intent, synonyms, conceptual similarity | May miss exact identifiers |

**Two-tower architecture**: one encoder for queries, one for data assets. Enables efficient ANN retrieval via HNSW graphs.

**Cost optimizations**:
- Matryoshka Representation Learning (MRL) — nested embeddings where initial dimensions carry core signal; serve 256 dims instead of 1024 for ~50% storage savings
- Quantization — float32 → int8 reduces memory footprint and accelerates dot-product ANN search
- Graph density tuning — reducing per-shard k (1200 → 200) yields ~34% latency reduction, ~17% CPU savings

**Reciprocal Rank Fusion (RRF)** merges lexical and semantic result lists, prioritizing assets that appear in both.

### Second stage: GBDT ranking (high precision)

Gradient Boosted Decision Trees (XGBoost/LightGBM) excel here because:
- Handle structured, heterogeneous metadata features natively
- Tolerate missing values (common in documentation) by learning optimal split direction
- Train fast (minutes to hours) with high interpretability (SHAP values)

**LambdaMART** (pairwise LTR): scales logistic loss by the NDCG change from swapping two items. Focuses learning on correcting errors at the top of the results where user attention concentrates.

| Approach | Focus |
|---|---|
| Pointwise | Each query-doc pair as independent regression |
| Pairwise (LambdaMART) | Relative order of document pairs |
| Listwise | Entire result list optimization |

For highly relational data, **Graph Neural Networks** can outperform GBDTs by operating directly on the lineage graph (multi-hop patterns, temporal features).

### Third stage: LLM re-ranking

Apply deep semantic understanding to top results:
- **Contextual compression** — extract only the most relevant metadata attributes before passing to LLM
- **Semantic chunking** — split by topic boundaries (one entity per chunk), not arbitrary character limits
- LLMs also perform **query normalization** — mapping diverse phrasings ("rename table", "table rename", "change name") to canonical intent

## LLM metadata synthesis

LLMs can generate missing metadata to improve discoverability:
- Semantic tagging — assign glossary terms to columns based on distribution and usage
- Summary generation — human-readable descriptions from schemas and sample data
- SoLM (Self-Supervised Denoising) — train a model to regenerate/clean entire metadata records

## Key points
- GBDTs dominate second-stage ranking for structured metadata because they handle missing values and heterogeneous features without extensive preprocessing
- LambdaMART's focus on top-of-list accuracy matches user behavior — most people only look at the first few results
- MRL and quantization together can cut vector retrieval costs by 50%+ with minimal quality loss
- LLM re-ranking is powerful but expensive — apply only to top-K candidates, not the full corpus

## Connections
- [[data-catalog-search-architecture]] — ranking is the third layer of the search stack
- [[search-ranking-signals]] — these are the features fed into the ranking models
- [[search-evaluation-metrics]] — NDCG/MAP are the objectives the LTR models optimize for
- [[specialized-knowledge-search]] — the agent-level search system that benefits from these ranking techniques

## Sources
- [[sources/papers/Improving Data Catalog Search Relevance]] — multi-stage ranking architecture, LambdaMART, MRL, contextual compression
