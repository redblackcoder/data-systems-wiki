# Data Agents

Data agents are LLM-powered systems that answer complex questions about enterprise data spanning structured sources (tables, dashboards, notebooks) and unstructured sources (documents, workspace files). Unlike coding agents that operate in static, deterministic environments like a filesystem, data agents work within dynamic, constantly evolving data lakehouses encompassing hundreds of thousands of assets.

## How they differ from coding agents

Coding agents have a key advantage: they can write tests to verify their output and iterate until tests pass. Data agents face a fundamentally harder problem — the "specification" is just a high-level user query with no expected correct answer to validate against.

A data agent must navigate multi-step reasoning: cross-system discovery, resolving contradictions between sources, performing calculations, and self-correcting when intermediate results reveal wrong assumptions.

## Key challenges

- **Scale of data discovery** — Enterprise customers have millions of structured and unstructured sources. Finding the right ones to answer a query breaks conventional search methods.
- **Determining "source of truth"** — Business knowledge is drawn from many sources (table metadata, company docs, internal messages) that are often outdated, contradictory, or superseded. The agent must determine which is authoritative.
- **Lack of verifiable tests** — No unit tests exist for open-ended data queries. Queries may not even be answerable due to data incompleteness, and the agent must identify and surface such cases.

## Genie's approach (Databricks)

Genie solves complex queries through distinct phases:
1. **Parallel multi-agent discovery** — sub-agents map the landscape across dashboards and tables in parallel
2. **Data investigation** — SQL extraction, comparative analysis
3. **Root-cause investigation** — escalates to docs when SQL stops explaining
4. **Self-correction loop** — corrects incorrect initial assumptions
5. **Reconciliation** — builds a unified picture
6. **Verification** — verifies exact numbers the user asked about

## Key points
- Genie achieved 91.6% accuracy on internal benchmarks vs 32.1% for a leading code agent + Databricks MCP
- The three technical advances that enable this: [[specialized-knowledge-search]], [[parallel-thinking]], [[multi-llm-agents]]
- Data agents operate in a paradigm where the environment is dynamic and semantically rich, not static and deterministic

## Connections
- [[specialized-knowledge-search]] — solves the data discovery challenge
- [[parallel-thinking]] — addresses the lack of verifiable tests
- [[multi-llm-agents]] — optimizes cost/latency while maintaining accuracy

## Sources
- [[sources/articles/Pushing the Frontier for Data Agents with Genie.pdf]] — primary source for all content on this page
