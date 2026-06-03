# Synthetic Query Generation

A technique for scaling search evaluation by using LLMs to generate realistic user queries from enterprise metadata documents. Operates the retrieval paradigm in reverse: instead of finding a document given a query, the LLM produces queries that a specific document would answer — guaranteeing a deterministic ground-truth mapping.

## Why SQG is necessary

Human-curated QnA datasets provide high fidelity but limited coverage. An enterprise catalog may contain tens of thousands of assets, yet curated datasets cover only a few hundred popular ones. SQG addresses:
- **Cold-start problem** — newly ingested assets have no historical queries
- **Long-tail coverage** — obscure but critical assets get evaluation coverage
- **Statistical power** — hundreds of synthetic queries per evaluation run

## How it works

### Metadata preprocessing

Raw metadata (DDL statements, database dumps) must be transformed into LLM-digestible formats:
- Flatten into structured JSON/YAML: table name, column descriptions, primary/foreign keys, glossary terms, governance classifications
- **Subschema enumeration** — identify logical clusters of connected tables via foreign key relationships (e.g., a fact table + its dimension tables)
- Inject interconnected subschemas to enable multi-hop query generation that reflects real BI workflows

### Prompt engineering for diversity

The prompt strategy is the critical factor in avoiding homogeneous, artificially clean queries:

1. **Persona assignment** — simulate specific organizational roles (data scientist exploring features vs. marketing analyst seeking KPIs)
2. **Lexical constraints** — explicitly prohibit using exact table names or column headers, forcing the model to use synonyms, business terminology, and semantic descriptions
3. **Intent variation** — require diverse query types for the same asset:
   - Exact-match lookups: "where is the customer churn dataset?"
   - Exploratory searches: "tables with European demographic data and encrypted emails"
   - Analytical questions: "schema for calculating MRR by sales territory"

### Validation and filtering

LLM-generated queries undergo automated quality checks:
- **Semantic validation** — verify each query is logically answerable from the provided schema context
- **Hallucination detection** — reject queries referencing non-existent columns, fictional metrics, or out-of-scope data
- **Deduplication** — prune near-duplicate queries that don't add evaluation diversity

Surviving queries are paired immutably with their source asset IDs.

## Key points
- Each synthetic query has a perfect, deterministic ground-truth mapping — no entity linking required
- Lexical constraints simulate the vocabulary mismatch that plagues real enterprise search (users don't know exact table names)
- Persona variation captures different search intents that a single prompt style would miss
- SQG is complementary to QnA adaptation, not a replacement — human queries capture real organizational vocabulary
- Regenerate synthetic datasets periodically as the catalog evolves to maintain representativeness

## Connections
- [[evaluation-dataset-generation]] — SQG is one of two dataset generation strategies in the evaluation framework
- [[search-evaluation-metrics]] — synthetic queries feed directly into NDCG@10 and Hit Rate@10 computation
- [[learning-to-rank]] — SQG can also generate training data for ranking models, not just evaluation
- [[data-catalog-search-architecture]] — SQG tests the full retrieval stack against realistic intent diversity

## Sources
- [[sources/papers/Generating Search Evaluation Datasets]] — SQG architecture: subschema enumeration, prompt engineering, validation protocols
