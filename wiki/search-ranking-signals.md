# Search Ranking Signals

The features extracted from data asset metadata that drive relevance ranking in data catalog search. Effective search requires combining signals across three dimensions: what the asset technically is, how it connects to other assets, and how humans interact with it.

## Signal categories

### Technical and logical metadata

| Category | Examples | Role |
|---|---|---|
| **Technical metadata** | Table/column names, data types, schemas | Foundational keyword matching |
| **Logical metadata** | Tags, descriptions, domain classifications | Bridges technical names to business terms |
| **Ownership** | Primary owners, contributing teams, SMEs | Trust signals, authority filtering |
| **Governance** | PII classifications, retention policies, sensitivity | Privacy/security constraint alignment |

Description length and semantic density correlate with asset quality — longer, more detailed descriptions often indicate better-maintained, more discoverable assets.

### Lineage and structural authority

Lineage position in the data graph acts as a relevance signal analogous to PageRank in web search. Assets that serve as sources for many downstream tables, dashboards, and ML models have high "structural authority."

Implementation approaches:
- Graph databases (Neo4j) or federated metadata services (Netflix Metacat) to model relationships
- Centrality scores calculated over the lineage graph
- A "gold source" table for financial reporting naturally outranks an analyst's experimental table

### Behavioral signals

Usage patterns reflect the community's collective intelligence:

- **Query frequency** — total queries over a rolling window (30-90 days)
- **User diversity** — unique users/teams, distinguishes broadly useful from single-silo assets
- **Downstream consumption** — count of active dashboards/pipelines consuming the asset
- **Freshness** — last update time, last schema change — prioritizes actively maintained assets

These signals allow the ranking model to adapt to changing organizational priorities without manual weight tuning.

## Key points
- No single signal category is sufficient — the power comes from combining all three dimensions
- Lineage-as-signal is the most underutilized in practice despite being one of the strongest authority indicators
- Behavioral signals decay naturally (rolling windows), making the system self-correcting for deprecated assets
- Governance metadata increasingly drives ranking in regulated industries — compliance constraints filter before relevance

## Connections
- [[data-catalog-search-architecture]] — signals are the second layer of the search stack
- [[learning-to-rank]] — signals become features in the GBDT/LTR models
- [[pull-based-ingestion]] — ingestion quality determines signal freshness
- [[specialized-knowledge-search]] — derives semantic context from these same metadata categories

## Sources
- [[sources/papers/Improving Data Catalog Search Relevance]] — signal taxonomy, lineage authority, behavioral metrics from LinkedIn DataHub and Amundsen
