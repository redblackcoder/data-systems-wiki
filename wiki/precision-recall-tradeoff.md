# Precision-Recall Tradeoff

The fundamental, inescapable tension in any information retrieval system: improving recall (finding everything relevant) almost inevitably introduces noise (lowering precision), while enforcing strict precision filters causes the system to drop marginally relevant items (lowering recall). Modern search architectures resolve this by optimizing each metric at different stages of the pipeline.

## How it works

**Precision** = of everything the system retrieved, what fraction was truly relevant? (purity)
- Perfect precision (1.0) = zero false positives, every result is relevant

**Recall** = of everything relevant that exists, what fraction did the system find? (completeness)
- Perfect recall (1.0) = zero false negatives, nothing relevant was missed

Widening the retrieval net catches more relevant items (↑ recall) but also catches noise (↓ precision). Tightening filters improves purity (↑ precision) but drops borderline-relevant items (↓ recall).

## When to optimize precision

Precision is critical when **the cost of a false positive is high**:

- **E-commerce search** — showing a Samsung case when the user searched "iPhone 15 case" frustrates the user and destroys conversion rates
- **Generative AI context injection** — LLMs typically cite 2-7 domains per response. A single irrelevant chunk fed to the model can cause hallucinated or factually incorrect output
- **Re-ranking stage** — the final top-10 presented to users must be curated

## When to optimize recall

Recall is critical when **the cost of a false negative is catastrophic**:

- **Legal e-discovery** — missing one relevant email could lose a multimillion-dollar lawsuit
- **Medical literature review** — missing a clinical trial about an adverse drug interaction could harm patients
- **Vector database ANN evaluation** — engineers measure how often approximate algorithms capture true nearest neighbors vs brute-force exact search (target: 90-95% recall under load)

In these domains, users willingly sift through hundreds of irrelevant results to ensure nothing critical is missed.

## Multi-stage resolution

Modern systems don't choose between precision and recall — they build pipelines that optimize each at different stages:

| Stage | Optimizes | Method | Goal |
|---|---|---|---|
| **Candidate generation** | Recall | Lightweight vector search or BM25 across millions of records | Top-1000 candidates; don't miss the answer |
| **Re-ranking** | Precision + NDCG | Computationally heavy cross-encoder scoring each candidate | Top-10 perfectly curated results |

The first stage accepts low precision (lots of noise) to guarantee high recall. The second stage filters that noise into a precise, well-ordered final list.

## Key points
- The tradeoff is inherent and inescapable — you cannot maximize both simultaneously at the same pipeline stage
- The decision of which to optimize depends entirely on the domain-specific cost of false positives vs false negatives
- Multi-stage architectures are the engineering solution: high-recall candidate generation → high-precision re-ranking
- For RAG systems, this maps to Context Recall (did we find everything?) vs Context Precision (is the best stuff on top?) — see [[rag-evaluation-metrics]]

## Connections
- [[search-evaluation-metrics]] — precision and recall are the foundational concepts all other metrics build upon
- [[rag-evaluation-metrics]] — Context Precision and Context Recall are the RAG-specific manifestation
- [[learning-to-rank]] — multi-stage ranking is how the tradeoff is resolved architecturally
- [[data-catalog-search-architecture]] — precision/recall tension exists at every layer of the search stack

## Sources
- [[sources/papers/Search Quality Metrics]] — precision/recall definitions, domain-specific optimization strategies, multi-stage architecture resolution
