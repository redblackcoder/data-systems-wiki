# LLM-as-Judge for Search Relevance

An automated evaluation architecture that uses a frontier LLM to assign multi-graded relevance scores to query-document pairs, replacing expensive human annotation at scale. The judge performs deep semantic reasoning over structured metadata — assessing whether an asset's columns, descriptions, and tags can resolve the user's search intent.

## How it works

The judge receives a triad of inputs:
1. **User search query** — the natural language information need
2. **Asset metadata** — structured representation (schema, descriptions, tags, usage stats)
3. **Grading rubric** — explicit evaluation criteria with score definitions

The model outputs:
- A numerical relevance score (0-3)
- A mandatory Chain-of-Thought (CoT) rationale explaining the score

CoT rationales serve dual purposes: they improve reasoning accuracy (forcing deliberation before scoring) and provide human-readable audit trails for diagnosing ranking failures.

## The 4-point graded relevance rubric

| Score | Label | Criteria |
|---|---|---|
| **3** | Perfect Match | Authoritative, definitive source. Schema completely satisfies intent. Key terms validated by tags and columns. No further searching needed. |
| **2** | Relevant | Contains necessary information but may require joins to other tables, or represents a secondary valid source (e.g., downstream dashboard vs. raw table). Strong semantic overlap. |
| **1** | Marginally Relevant | Topically related but insufficient. Shares keyword overlap or business domain but lacks specific columns, temporal scope, or granularity requested. |
| **0** | Irrelevant | No semantic or operational relevance. Any textual overlap is coincidental. Cannot fulfill the user's objective. |

This 4-point scale aligns with NDCG's exponential discounting — the difference between a 3 and a 2 matters more than between a 1 and a 0.

## Bias mitigation

### Verbosity bias
LLMs assign higher scores to longer, more verbose documents regardless of utility. A heavily documented but irrelevant legacy table may outscore a sparsely documented but correct production table.

**Mitigation**: explicitly instruct the judge to prioritize schema matching and critical column presence over description length.

### Domain knowledge gaps
Enterprise environments use proprietary acronyms and internal nomenclature that the judge model may not understand.

**Mitigation**: inject "grading notes" or an abbreviated enterprise data dictionary into the system prompt, providing localized contextual grounding.

### Position bias
When multiple candidates are presented together, LLMs favor items earlier in the prompt.

**Mitigation**: evaluate each query-document pair in isolation (one at a time), not as a batch comparison.

## Calibration and trust

Automated judgments require continuous validation:
- Sample ~5% of LLM judgments for human expert audit
- Calculate inter-annotator agreement (Cohen's Kappa or Pearson correlation) between LLM and human scores
- If alignment drifts below threshold, iteratively refine the rubric and prompt

The goal is not perfect agreement on every judgment, but statistical consistency at the aggregate metric level.

## Key points
- JSON mode / function calling ensures deterministic, machine-readable output for pipeline automation
- CoT rationales are non-optional — they both improve accuracy and enable root-cause analysis of ranking failures
- The rubric must be specific to the domain (enterprise metadata) — generic "relevance" instructions produce high variance
- LLM judges are scalable but not a replacement for human calibration — they amplify human judgment, not eliminate it
- Pair with [[evaluation-dataset-generation|composite scoring]] to blend semantic judgments with operational signals

## Connections
- [[evaluation-dataset-generation]] — the judge provides semantic scores that feed into composite relevance
- [[search-evaluation-metrics]] — graded scores are required for NDCG computation
- [[rag-evaluation-metrics]] — LLM-as-a-judge is also the pattern behind RAG evaluation frameworks (Ragas, DeepEval)
- [[synthetic-query-generation]] — SQG produces the queries that the judge evaluates
- [[search-ranking-signals]] — popularity/usage signals complement semantic judgments in composite scoring

## Sources
- [[sources/papers/Generating Search Evaluation Datasets]] — judge architecture, rubric design, bias mitigation, calibration methodology
