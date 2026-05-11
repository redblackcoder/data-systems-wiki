# Parallel Thinking

A technique for improving data agent accuracy by sampling multiple independent trajectories (reasoning paths) for the same query, then aggregating the relevant information across trajectories to compute the final answer. Analogous to ensemble methods in ML — diversity of reasoning compensates for the lack of verifiable tests.

## How it works

1. **Sample N trajectories** — run the agent N times independently on the same query (e.g., N=8)
2. **Each trajectory explores differently** — different search paths, different intermediate conclusions
3. **Judge/aggregate** — a judge component evaluates and aggregates findings across all trajectories
4. **Produce final answer** — synthesized from the best evidence across trajectories

## Key results

- GPT-5.4 baseline: 65.5% → with parallel thinking: 75.7% (+10.2pp)
- Opus 4.6 baseline: 73.8% → with parallel thinking: 81.4% (+7.6pp)
- Runtime increases (3.8m → 8.2m for GPT-5.4; 5.5m → 9.5m for Opus 4.6) — roughly 2x latency
- Cost increases ~2.13x at N=8 with Opus 4.6, but Multi-LLM optimization brings it back down to 0.95x

## Why it works for data agents

Coding agents can write tests and iterate until correct. Data agents have no such ground truth. Parallel thinking compensates by:
- Reducing variance in exploration paths
- Catching errors that a single trajectory might miss
- Providing multiple "opinions" for the judge to evaluate

## Key points
- Tradeoff: accuracy vs latency/cost — parallel thinking adds both
- Combining with [[multi-llm-agents]] offsets the cost increase
- N=8 was used in the reported experiments
- The judge/aggregate step is critical — naive majority voting would be insufficient for complex analytical queries

## Connections
- [[data-agents]] — addresses the "no verifiable tests" challenge
- [[multi-llm-agents]] — combined to reduce the cost overhead of parallel trajectories
- [[specialized-knowledge-search]] — each trajectory benefits from better search

## Sources
- [[sources/articles/Pushing the Frontier for Data Agents with Genie.pdf]] — experimental results comparing GPT-5.4 and Opus 4.6
