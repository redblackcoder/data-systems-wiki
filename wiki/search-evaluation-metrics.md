# Search Evaluation Metrics

The quantitative framework for measuring and improving search relevance. Rigorous evaluation requires both offline metrics (computed against judgment lists) and online validation (canary systems detecting regressions on live traffic).

## Metric taxonomy

The fundamental distinction: **rank-aware** metrics penalize relevant items that appear lower in the list, while **order-agnostic** (set-based) metrics only care whether items are present within the top-K, not their relative ordering.

| Category | Metric | Relevance type | Best for |
|---|---|---|---|
| **Rank-aware** | NDCG@K | Graded (0-4) | Human-facing search; embedding evaluation (MTEB benchmark) |
| **Rank-aware** | MAP@K | Binary (0/1) | Multi-answer scenarios; recommender systems |
| **Rank-aware** | MRR | Binary (0/1) | Q&A, chatbots, known-item navigational search |
| **Order-agnostic** | Hit Rate@K | Binary (0/1) | ANN index evaluation; candidate generation |
| **Order-agnostic** | Recall@K | Binary (0/1) | Measuring coverage within LLM token limits |

### Rank-aware metrics

**NDCG** (Normalized Discounted Cumulative Gain) — the industry standard. Accounts for the fact that a highly relevant result at position 5 is less valuable than at position 1 (logarithmic discount), and that a "somewhat relevant" result has partial value (graded scores). Default metric on the MTEB retrieval leaderboard for evaluating embedding models.

**Common misconception**: MRR and MAP are often incorrectly called "order-agnostic." They are rank-aware — the position of results directly impacts their scores. The distinction from NDCG is that MRR/MAP use binary relevance (relevant or not), while NDCG uses graded relevance.

**NDCG limitations**: susceptible to presentation bias; assumes unjudged documents have zero relevance (penalizes newly indexed content); does not inherently measure result diversity.

### Order-agnostic metrics

Critical for **agentic systems**: an LLM processes its entire context window simultaneously through attention. Whether the key fact is in chunk 1 or chunk 10 doesn't matter — what matters is whether it's present at all.

Two failure modes for agents:
- **Under-retrieval** — missing necessary facts → incomplete reasoning
- **Over-retrieval** — too much noise → hallucination, topic drift, wasted tokens

## Ground truth collection

| Method | Strengths | Weaknesses |
|---|---|---|
| **Human rating (SMEs)** | High accuracy, handles nuance | Expensive, slow, hard to scale |
| **Behavioral logs (clicks, saves)** | High volume, reflects real preferences | Position bias, noise |
| **LLM-as-a-Judge** | Fast, repeatable, scalable | Model bias, requires careful prompting |

**Position bias mitigation**: users click the first result regardless of quality. Techniques include propensity scoring and position debiasing in the ranking objective.

A practical approach: human evaluators label several thousand representative queries as training data for LTR models, supplemented by behavioral signals from production.

## Operational reliability: Data Canary pattern

Search relevance degrades silently when the index drifts from the source of truth ("data corruption"). Netflix's Data Canary pattern validates against this:

1. Route a fraction (0.2%) of global traffic to a canary cluster
2. Canary serves the new metadata version; baseline serves production version
3. Compare error rates between clusters
4. **Block publication** if differential exceeds threshold (e.g., 10x error rate)

This detects regressions in minutes, ensuring ranking model improvements aren't undermined by underlying data state issues.

## Key points
- NDCG is the standard objective for LTR training (via LambdaMART) because it handles graded relevance and position discount
- Ground truth is the bottleneck — most teams underinvest here, which caps how good the ranking model can become
- LLM-as-a-Judge is increasingly viable for scaling evaluation but needs calibration against human judgments
- Canary systems are essential for trust — users lose confidence after even one visible regression
- For agentic/RAG systems, order-agnostic metrics (Hit Rate@K, Recall@K) are more relevant than NDCG — see [[rag-evaluation-metrics]]
- No single metric describes a modern retrieval system — deploy a matrix of metrics tailored to each architectural stage

## Connections
- [[data-catalog-search-architecture]] — evaluation is the validation layer for the entire stack
- [[learning-to-rank]] — NDCG/MAP are the objectives LTR models optimize
- [[search-ranking-signals]] — ground truth labels reveal which signals actually matter
- [[rag-evaluation-metrics]] — extends these metrics to evaluate RAG pipelines (retriever + generator)
- [[precision-recall-tradeoff]] — the fundamental tension that multi-stage architectures resolve
- [[evaluation-dataset-generation]] — how to construct the ground-truth datasets these metrics consume
- [[synthetic-query-generation]] — scaling evaluation coverage via LLM-generated queries
- [[llm-as-judge-search]] — automated graded relevance judgments for NDCG computation

## Sources
- [[sources/papers/Improving Data Catalog Search Relevance]] — metric definitions, ground truth methods, Netflix Data Canary pattern
- [[sources/papers/Search Quality Metrics]] — metric taxonomy (rank-aware vs order-agnostic), NDCG limitations, agent evaluation requirements
- [[sources/papers/Generating Search Evaluation Datasets]] — NDCG@10/Hit Rate@10 mathematical formulations, composite relevance scoring
