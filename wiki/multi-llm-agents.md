# Multi-LLM Agents

An architecture pattern where different sub-agent roles within a compound AI system use different LLMs, selected based on each model's complementary strengths. Rather than using one model for everything, the system routes planning, search, reasoning, and judging to whichever LLM performs best at that specific task.

## How it works

1. **Decompose agent into roles** — identify distinct sub-agent functions (plan, search, reason, judge, code generation)
2. **Assign LLMs per role** — choose the best model for each role based on accuracy, cost, and latency characteristics
3. **Optimize with GEPA** — use prompt optimization methods to further improve each sub-agent's performance with its assigned model
4. **Route dynamically** — the orchestrator dispatches to the appropriate model for each step

## Key results

- Combined system (GPT-5.4 + Opus 4.6): 91.6% accuracy at 0.95x relative cost
- This is both more accurate AND cheaper than the single-model baseline
- GEPA optimization improves accuracy while reducing token cost for each model:
  - GPT 5.4-mini: best cost efficiency (lowest cost, ~77% accuracy after optimization)
  - GPT 5.4 / Sonnet 4.5: highest accuracy (~78%) at moderate cost
  - Opus 4.6: highest baseline cost, but optimization brings cost down significantly

## Why different LLMs for different roles

Different models have different strengths:
- Some excel at planning and decomposition
- Some are better at search/retrieval tasks
- Some are stronger at mathematical reasoning and code generation
- Some are cost-effective enough for high-volume search steps

Using one expensive model for everything wastes money on tasks a cheaper model handles equally well.

## Key points
- This is not just about cost — accuracy also improves because each sub-agent uses its best-fit model
- GEPA (prompt optimization) is key to extracting maximum performance from each model assignment
- The Databricks platform makes this practical by providing seamless access to frontier models (Opus, GPT, Gemini), open-source, and custom-trained models
- Combined with [[parallel-thinking]], the cost increase from parallelism is fully offset

## Connections
- [[data-agents]] — enables the full Genie system to achieve 91.6% accuracy
- [[parallel-thinking]] — Multi-LLM offsets parallel thinking's cost overhead
- [[specialized-knowledge-search]] — the search sub-agent is one of the roles that benefits from model selection

## Sources
- [[sources/articles/Pushing the Frontier for Data Agents with Genie.pdf]] — GEPA optimization results and multi-LLM architecture
