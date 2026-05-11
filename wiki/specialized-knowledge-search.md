# Specialized Knowledge Search

A technique for enterprise data discovery that builds semantic search indices from existing data assets (workspace tables, notebooks, dashboards, documents, files) to efficiently find relevant assets for a user query. Used in Databricks Genie to dramatically improve table discovery over conventional search.

## How it works

1. **Derive semantic context** — extract rich metadata from existing enterprise assets (table schemas, dashboard definitions, notebook content, document text)
2. **Construct search index** — build a search index enriched with this semantic context
3. **Multi-index parallel search** — query multiple search indices in parallel, using rich metadata signals to rank results
4. **Iterative refinement** — the search sub-agent can take multiple steps (up to 16) to refine its search, with each step improving recall

## Key results

- With semantic context: Recall@10 of 0.774 at 16 steps
- Without semantic context: Recall@10 of 0.466 at 16 steps
- At just 2 steps, semantic context already achieves 0.450 vs 0.075 without — a 6x improvement early on
- Up to 40% improvement on table discovery benchmarks overall

## Key points
- The gap between with/without semantic context is largest at low step counts — semantic context provides a much better starting point
- This addresses the "scale of data discovery" challenge where enterprises have millions of data assets
- Multiple metadata signals (not just table names) are critical for grounding the search
- Performance scales with max steps allowed, but diminishing returns after ~8 steps

## Connections
- [[data-agents]] — solves the data discovery challenge unique to data agents
- [[parallel-thinking]] — can be combined with specialized search for further gains
- [[multi-llm-agents]] — the search sub-agent can use a different, optimized LLM

## Sources
- [[sources/articles/Pushing the Frontier for Data Agents with Genie.pdf]] — benchmark results and architecture description
