# **Engineering Architectures for High-Precision Search Relevance in Structured Data Discovery**

The optimization of search relevance within data catalogs representing structured assets has transitioned from a metadata management problem to a complex information retrieval engineering challenge. In modern decentralized data environments, practitioners require discovery systems that operate with the precision and contextual awareness of consumer-grade search engines. Achieving this requires a multi-layered approach that integrates robust ingestion architectures, sophisticated ranking models based on diverse metadata and lineage signals, and the strategic application of large language models for both re-ranking and the synthesis of semantic context.

## **Ingestion Architectures and Index Stability**

The foundational layer of a search relevance system is the ingestion pipeline, which must ensure that the search index remains a consistent and reliable reflection of the underlying data estate. Engineering strategies for metadata ingestion have evolved from simple push-based models to sophisticated pull-based streaming architectures designed to handle the scale and burstiness of modern enterprise data updates.

### **Evolution Toward Pull-Based Ingestion**

Traditional search platforms often rely on push-based ingestion, where upstream metadata producers write directly to the search cluster. While straightforward to implement, this model introduces significant risks regarding reliability and consistency. Under conditions of bursty traffic—such as massive batch schema updates or widespread lineage re-computations—the target search shards can be overwhelmed, leading to indexing failures, increased visibility lag, and operational instability.1

The shift toward pull-based ingestion utilizes durable message streams, such as Kafka or Kinesis, as intermediate buffers. In this architecture, search shards pull data from stream partitions, which decouple consumption from processing and allow for granular control over ingestion rates. Uber’s search infrastructure illustrates this transition, where the adoption of pull-based streaming enabled deterministic recovery through per-shard bounded queues.1 By mapping each search shard to a specific stream partition, the system can perform deterministic replays of metadata events following a failure.

| Ingestion Component | Mechanism | Operational Benefit |
| :---- | :---- | :---- |
| **Durable Stream (Kafka/Kinesis)** | Acting as a buffer between producers and indexers. | Absorbs traffic spikes; prevents shard queue overflows.1 |
| **Blocking Queue** | Decoupling message polling from indexing threads. | Enables parallel processing and backpressure handling.1 |
| **External Versioning** | Applying version timestamps to metadata records. | Ensures out-of-order messages do not overwrite newer updates.1 |
| **Offset Tracking** | Maintaining pointers to the last processed stream message. | Allows for deterministic recovery and consistency.1 |

This architecture also allows for distinct ingestion modes tailored to different resource constraints. For example, segment replication allows ingestion only on primary shards, with replicas fetching completed segments, thereby reducing total CPU usage at the cost of a slight lag in search visibility.1 Conversely, an all-active mode ensures near-instant visibility by ingesting on all shard copies, which is critical for highly dynamic data environments but requires higher compute investments.1

### **Layered Index Management for Real-Time Freshness**

For search systems that must maintain high freshness while managing large metadata volumes, a layered indexing approach is often employed. Systems like Sia utilize a tri-level architecture consisting of a memory-resident Live Index, a Snapshot Index, and a Base Index.2 The Live Index buffers incoming metadata updates, providing immediate queryability. To mitigate memory pressure, this Live Index is periodically flushed—typically every 30 minutes—to create a Snapshot Index. Over time, these snapshots are consolidated into a highly compressed and optimized Base Index through a periodic compaction process.2

While this layered approach optimizes memory usage and write throughput, it introduces significant technical debt. Every query operator added to the underlying search library (such as Lucene) requires a parallel implementation across the custom index structures, which slows the adoption of new search features.2 Engineering teams often find that the complexity of maintaining base-snapshot-live architectures is only justified for a small subset of use cases. For standard data discovery, near real-time (NRT) search in engines like OpenSearch or Elasticsearch, which manage compaction internally, offers a more sustainable operational path.2

## **Signal Extraction from Metadata and Relational Lineage**

The relevance of search results in a data catalog is determined by the breadth and quality of the signals extracted from the metadata. These signals are categorized into technical metadata, behavioral patterns, and relational lineage, each providing a different dimension of context.

### **Technical and Logical Metadata Attributes**

Technical metadata provides the base for lexical matching, while logical metadata adds human-defined context. In a platform like LinkedIn DataHub, the search engine indexes a wide array of metadata types to ensure comprehensive coverage.3

| Metadata Category | Specific Attributes | Role in Relevance |
| :---- | :---- | :---- |
| **Technical Metadata** | Table names, column names, data types, database schemas. | Foundational keyword matching and technical discovery.3 |
| **Logical Metadata** | User-defined tags, asset descriptions, domain classifications. | Bridges the gap between technical names and business terms.3 |
| **Ownership Data** | Primary owners, contributing teams, subject matter experts. | Enables filtering by authority and improves trust signals.3 |
| **Governance Metadata** | PII classifications, retention policies, sensitivity levels. | Ensures results align with privacy and security constraints.3 |

Effective extraction involves more than just capturing these fields; it requires normalizing them to facilitate better retrieval. For example, descriptions can be analyzed for length and semantic density, as longer, more detailed descriptions often correlate with higher asset quality and discoverability.5

### **Lineage and Structural Authority**

Lineage is increasingly recognized as a first-class signal for search ranking. In decentralized data ecosystems, the authority of a dataset is often reflected in its position within the lineage graph. Assets that serve as the source for numerous downstream tables, dashboards, and machine learning models possess a high "structural authority".4

By modeling lineage relationships explicitly—often using a graph database like Neo4j or a federated metadata service like Metacat—search systems can calculate centrality scores.3 These scores function similarly to the PageRank algorithm used in web search, where the relevance of a data asset is boosted based on the number and quality of its dependencies.7 A table that is the "gold source" for corporate financial reporting will naturally have higher structural authority than an experimental table created by an individual analyst.

### **Behavioral Signals and Popularity Metrics**

Usage patterns provide a dynamic signal that reflects the community's collective intelligence. In systems like Amundsen, popularity is a primary ranking feature, ensuring that highly queried tables appear earlier in search results than those that are rarely accessed.7

Commonly extracted behavioral signals include:

* **Query Frequency:** The total number of times a table or dashboard is queried over a rolling window (e.g., 30 or 90 days).7  
* **User Diversity:** The number of unique users or teams interacting with the asset, which helps distinguish between broadly useful datasets and those used only within a single silo.  
* **Downstream Consumption:** The count of active dashboards or production pipelines that consume the data asset.4  
* **Freshness and Updates:** Monitoring when a table was last updated or when its schema last changed helps the ranker prioritize active, maintained assets over deprecated or stale ones.7

The integration of these signals into the ranking model allows the search engine to adapt to changing organizational priorities without manual tuning of weights for every individual asset.

## **Multi-Stage Ranking Models and Learning to Rank**

Scalable search relevance engineering typically adopts a multi-stage architecture. The initial stage focuses on broad retrieval (high recall), while subsequent stages apply complex machine learning models to refine the ordering (high precision).

### **First-Stage Retrieval: Vector and Lexical Hybridization**

The first stage of the search pipeline aims to narrow down the entire corpus of data assets to several hundred candidates. Traditional lexical search using BM25 is excellent for matching exact technical terms, such as specific column names or table IDs.8 However, lexical search fails to capture semantic intent, particularly when users employ synonyms or natural language descriptions.

To address this, semantic search utilizing embeddings and vector indices is employed. Two-tower architectures are a common engineering pattern, where one tower encodes the query and the other encodes the data asset.9 This allows for efficient candidate retrieval from an inverted index or an HNSW (Hierarchical Navigable Small World) graph.9

To optimize the cost and performance of vector retrieval, engineers employ several techniques:

* **Matryoshka Representation Learning (MRL):** This approach generates nested embeddings where the initial dimensions contain the core semantic signal. This allows for serving smaller embedding "cuts" (e.g., 256 dimensions instead of 1024\) with minimal loss in retrieval quality, significantly reducing storage and compute costs.10  
* **Quantization:** Converting high-precision float32 embeddings to compact int7 or int8 representations reduces the memory footprint of the index and accelerates dot-product calculations during Approximate Nearest Neighbor (ANN) search.10

| Retrieval Factor | Optimization Strategy | Latency/Cost Impact |
| :---- | :---- | :---- |
| **Embedding Size** | MRL (e.g., 256 dimensions) | \~50% reduction in storage costs.10 |
| **Precision** | Quantization (int8/int7) | Significant reduction in memory and CPU usage.10 |
| **Graph Density** | Adjusting per-shard k (e.g., 1200 to 200\) | \~34% latency reduction; \~17% CPU savings.10 |

### **Second-Stage Ranking: Gradient Boosted Trees and LTR**

Once a set of candidates is retrieved, a more computationally intensive ranking model is applied. Learning to Rank (LTR) utilizes trained machine learning models to build a ranking function that optimizes for a specific objective, such as NDCG (Normalized Discounted Cumulative Gain) or MAP (Mean Average Precision).11

Gradient Boosted Decision Trees (GBDTs), specifically XGBoost and LightGBM, are the preferred models for this stage because they excel at processing the structured, heterogeneous features typical of metadata catalogs.12 These models can natively handle missing values—which are common in documentation—by learning the optimal split direction during training.12

#### **Pairwise and Listwise Optimization**

In the context of LTR, pairwise and listwise approaches are generally superior to pointwise methods. Pointwise ranking treats each query-document pair as an independent regression task, whereas pairwise methods, such as LambdaMART (implemented in XGBoost), focus on the relative order of documents.14

The LambdaMART algorithm scales the logistic loss with the change in NDCG resulting from a swap of two items in the ranking. This ensures the model focuses its learning on correcting errors at the top of the search list, where relevance is most critical for users.14

For highly relational data, where assets are connected through a complex web of lineage and ownership, standard GBDTs may encounter a bottleneck in feature engineering. In these cases, Graph Neural Networks (GNNs) can achieve significantly higher AUROC scores by operating directly on the relational structure, capturing temporal and multi-hop patterns that are difficult to flatten into a single table.13

| Model Type | Best For | Training Speed | Interpretability |
| :---- | :---- | :---- | :---- |
| **XGBoost / LightGBM** | Single flat tables, pre-engineered features. | Fast (minutes to hours) | High (SHAP values).13 |
| **Deep Neural Networks** | Large, high-dimensional vector inputs. | Moderate | Low |
| **Graph Neural Networks** | Multi-table relational data and lineage. | Slower | Moderate.13 |

## **Application of LLMs for Re-ranking and Metadata Synthesis**

The integration of Large Language Models (LLMs) represents a significant advancement in search relevance engineering, providing capabilities for semantic interpretation and automated metadata generation that were previously unattainable.

### **Query Normalization and Intent Classification**

Data catalog users often utilize ambiguous or technically inconsistent phrasing. LLMs are highly effective at query normalization—mapping diverse user inputs to a standardized canonical form.17 This process involves identifying the core concept (e.g., "table rename"), removing filler words ("how to in my product"), and mapping variations ("renaming", "change name") to a single intent.17

Furthermore, LLMs can perform "schema linking" to bridge the gap between conceptual natural language queries and technical asset names.18 By providing the LLM with a scoped subset of the database schema and metadata, the model can reason through the relationships required to satisfy a query, such as identifying the specific joins and filters needed to extract "active customers by region" from a set of normalized tables.18

### **LLM-Based Re-ranking and Contextual Compression**

A powerful pattern in modern discovery systems is the use of LLMs for final-stage re-ranking. While vector search is effective for initial retrieval, an LLM can apply a deeper level of semantic understanding to the top results, considering the nuances and context of both the query and the metadata.20

Because LLMs are computationally expensive and have finite context windows, engineering strategies focus on maximizing information density:

* **Contextual Compression:** Post-retrieval, only the most relevant sentences or attributes are extracted from a metadata object. This filtering removes noise and ensures the LLM's attention is focused on the data most likely to determine relevance.20  
* **Semantic Chunking:** Metadata documents are split into chunks based on topic boundaries (e.g., one H2 section or one logical entity per chunk). This prevents the loss of context that occurs when splitting documents arbitrarily at character limits.21  
* **Hybrid RRF (Reciprocal Rank Fusion):** Combining the scores from lexical (BM25) and semantic retrieval lists ensures that results appearing in both lists—indicating both keyword and conceptual alignment—are prioritized.8

### **Automated Metadata Synthesis and Denoising**

The lack of consistent documentation is a primary barrier to data discoverability. LLMs can be utilized to synthesize metadata by analyzing schemas, sample data, and query logs.22 The SoLM (Self-Supervised Denoising) framework is one such approach, where a model is trained to regenerate entire metadata records, thereby cleaning, normalizing, and completing sparse or poorly structured documentation.23

This synthesis can take several forms:

* **Semantic Tagging:** Automatically assigning business glossary terms to technical columns based on the data's distribution and usage patterns.24  
* **Summary Generation:** Creating human-readable table and column descriptions that reflect the actual data content.22  
* **Review Clustering:** Grouping user comments or data quality alerts to provide a holistic view of an asset's reliability.8

By transforming a webpage or a technical DDL statement from a simple document into a "queryable knowledge base" through structured data markup (such as JSON-LD), engineers can significantly improve the confidence of LLM-based search agents in the accuracy of the retrieved information.25

## **Evaluation Metrics and Ground Truth Collection**

Improving search relevance is an iterative process that requires rigorous measurement using both offline and online metrics.

### **Quantitative Ranking Metrics**

Engineers utilize several standard metrics to evaluate how well a system orders its results. All of these metrics are calculated "at K," focusing only on the top results most likely to be seen by a user.26

* **Mean Reciprocal Rank (MRR):** Measures the average of the reciprocal ranks of the first relevant result. This is ideal for navigational queries where the user is looking for a specific, single asset.27  
* **Mean Average Precision (MAP):** Averages the precision at each relevant item's position. This is used when multiple assets may be relevant to a single query.26  
* **Normalized Discounted Cumulative Gain (NDCG):** The industry standard for complex retrieval. It accounts for graded relevance scores (e.g., 0 for irrelevant to 4 for highly relevant) and applies a logarithmic discount to results further down the list.27

The mathematical formulation for Discounted Cumulative Gain (DCG) is:

![][image1]  
This is then normalized by the Ideal DCG (IDCG), which represents the score of a perfect ranking of the same results.26

### **Ground Truth and Judgment Lists**

To compute these metrics, a "judgment list" is required—a set of queries and documents with pre-assigned relevance labels.11 Generating these lists is a significant engineering and operational undertaking.

| Method | Source | Strengths | Weaknesses |
| :---- | :---- | :---- | :---- |
| **Human Rating** | Subject matter experts (SMEs). | High accuracy; handles nuance and ambiguity.30 | Expensive; slow; difficult to scale.14 |
| **Behavioral Logs** | User clicks, "saves," and downloads. | High volume; reflects actual user preferences.15 | Subject to position bias and noise.16 |
| **LLM-as-a-Judge** | High-capacity LLMs (e.g., GPT-4, Claude 3). | Fast; repeatable; scalable evaluation.33 | Potential for model bias; requires careful prompting. |

High-quality ground truth collection often involves a "human-in-the-loop" model, where human evaluators label several thousand representative queries, which are then used to train the LTR model.15 To mitigate position bias in behavioral data—where users click the first result regardless of quality—engineers use techniques like propensity scoring or position debiasing in the ranking objective.14

## **Operational Reliability and "Data Canaries"**

A search relevance system is only as good as the trust users have in its results. In dynamic metadata environments, "data corruption"—where the index no longer matches the source of truth—can lead to poor search experiences and potentially erroneous business decisions.33

To protect against this, organizations like Netflix implement "Data Canary" systems. This pattern involves a dedicated orchestrator that validates catalog metadata transformations using production traffic.33 By routing a tiny fraction (e.g., 0.2%) of global traffic through a canary cluster and comparing the results to a baseline cluster serving the production version, teams can detect regressions in minutes.33

The canary system looks for a high differential in error rates (e.g., 10x differential) between the clusters, which serves as a clear signal to block the publication of a new metadata version.33 This level of proactive validation ensures that improvements to the search ranking model are not undermined by underlying data state issues.

## **Synthesizing Multi-Modal Signals for Advanced Discovery**

Modern search engineering also involves synthesizing signals across different data modalities. While text-based metadata remains the primary driver for structured asset discovery, other modalities can provide supplementary context.35

For example, a "weighted fusion" strategy can be used to balance contributions from different modalities based on user intent. In a system searching for visual dashboards, image features (extracted using models like CLIP or ViT) can be merged with text descriptions and behavioral popularity signals.35 This hybrid approach typically utilizes Elasticsearch for text-based retrieval and a vector database like FAISS, Milvus, or Pinecone for similarity search, merging the results through a unified scoring function.8

## **Implementation Best Practices for Data Teams**

Engineering for search relevance requires a pragmatic approach that prioritizes high-impact features and avoids common pitfalls in the modeling process.

### **Feature Selection and Regularization**

When training LTR models, it is essential to identify which features most influence the predictions. Techniques like Random Forests provide a robust mechanism for assessing feature importance, allowing engineers to prune low-impact features that add noise to the model.36 Regularization techniques, such as L1 (Alpha) and L2 (Lambda) in XGBoost and LightGBM, are critical to prevent overfitting, especially when working with sparse metadata datasets.37

### **Avoiding the "Pre-processing Purgatory"**

One advantage of GBDTs like XGBoost is their ability to work with raw categorical features and missing values without requiring extensive imputation or one-hot encoding.12 This allows engineering teams to move quickly from signal extraction to model training, focusing their efforts on feature discovery rather than data cleaning.12

### **Progressive Tuning and Cross-Validation**

The development of a ranking model should follow a progressive tuning strategy. Engineers should start with a small, interpretable model, analyze its errors using tools like SHAP (SHapley Additive exPlanations), and expand the feature set gradually.12 K-fold cross-validation is essential to ensure that the model generalizes well across different subsets of the data catalog, particularly when dealing with diverse domains like finance, marketing, and engineering metadata.12

In conclusion, the engineering of search relevance for structured assets is a multi-dimensional task that requires a deep integration of infrastructure stability, signal extraction, and advanced machine learning. By evolving from static catalogs to active discovery systems that leverage lineage, usage patterns, and the interpretive power of LLMs, organizations can unlock the full potential of their data ecosystems. The key to success lies in the ability to balance the recall provided by vector-based retrieval with the high-precision ordering of LTR models, all while maintaining a rigorous, metric-driven approach to evaluation and trust.

#### **Works cited**

1. Uber Moves In-House Search Indexing to Pull-Based Ingestion in OpenSearch \- InfoQ, accessed May 1, 2026, [https://www.infoq.com/news/2026/02/uber-pull-based-opensearch/](https://www.infoq.com/news/2026/02/uber-pull-based-opensearch/)  
2. The Evolution of Uber's Search Platform, accessed May 1, 2026, [https://www.uber.com/us/en/blog/evolution-of-ubers-search-platform/](https://www.uber.com/us/en/blog/evolution-of-ubers-search-platform/)  
3. LinkedIn DataHub Guide (2025): Setup, and Alternatives \- Atlan, accessed May 1, 2026, [https://atlan.com/linkedin-datahub-metadata-management-open-source/](https://atlan.com/linkedin-datahub-metadata-management-open-source/)  
4. Netflix Reimagines Discovery & Governance with DataHub, accessed May 1, 2026, [https://datahub.com/customer-stories/netflix/](https://datahub.com/customer-stories/netflix/)  
5. Understanding Airbnb's Search Rank Algorithm: Insights for ..., accessed May 1, 2026, [https://myangelhost.com/understanding-airbnbs-search-rank-algorithm/](https://myangelhost.com/understanding-airbnbs-search-rank-algorithm/)  
6. Netflix Metacat: Origin, Architecture, Features & How to Set Up in 2025? \- Atlan, accessed May 1, 2026, [https://atlan.com/metacat-netflix-open-source-metadata-platform/](https://atlan.com/metacat-netflix-open-source-metadata-platform/)  
7. Amundsen, accessed May 1, 2026, [https://www.amundsen.io/amundsen/](https://www.amundsen.io/amundsen/)  
8. Structuring ecommerce data for LLM retrieval and RAG | ContentGecko, accessed May 1, 2026, [https://contentgecko.io/kb/llmo/structuring-data-for-llm-retrieval/](https://contentgecko.io/kb/llmo/structuring-data-for-llm-retrieval/)  
9. Innovative Recommendation Applications Using Two Tower Embeddings at Uber, accessed May 1, 2026, [https://www.uber.com/us/en/blog/innovative-recommendation-applications-using-two-tower-embeddings/](https://www.uber.com/us/en/blog/innovative-recommendation-applications-using-two-tower-embeddings/)  
10. Evolution and Scale of Uber's Delivery Search Platform, accessed May 1, 2026, [https://www.uber.com/us/en/blog/evolution-and-scale-of-ubers-delivery-search-platform/](https://www.uber.com/us/en/blog/evolution-and-scale-of-ubers-delivery-search-platform/)  
11. Learning To Rank (LTR) | Elastic Docs, accessed May 1, 2026, [https://www.elastic.co/docs/solutions/search/ranking/learning-to-rank-ltr](https://www.elastic.co/docs/solutions/search/ranking/learning-to-rank-ltr)  
12. The Art of XGBoost and LightGBM for Data Science in Microsoft Fabric, accessed May 1, 2026, [https://community.fabric.microsoft.com/t5/Data-Science-Community-Blog/The-Art-of-XGBoost-and-LightGBM-for-Data-Science-in-Microsoft/ba-p/4844647](https://community.fabric.microsoft.com/t5/Data-Science-Community-Blog/The-Art-of-XGBoost-and-LightGBM-for-Data-Science-in-Microsoft/ba-p/4844647)  
13. Machine Learning on Structured Data FAQ: 15 Questions Answered \- Kumo.ai, accessed May 1, 2026, [https://kumo.ai/resources/learn/faq/ml-structured-data/](https://kumo.ai/resources/learn/faq/ml-structured-data/)  
14. Learning to Rank — xgboost 3.3.0-dev documentation, accessed May 1, 2026, [https://xgboost.readthedocs.io/en/latest/tutorials/learning\_to\_rank.html](https://xgboost.readthedocs.io/en/latest/tutorials/learning_to_rank.html)  
15. Replicating The Bing Search Algorithm (Learn-to-Rank Model) \- Medium, accessed May 1, 2026, [https://medium.com/@humzahmalik/learning-to-rank-algorithm-a-worked-through-example-ebc8629089b9](https://medium.com/@humzahmalik/learning-to-rank-algorithm-a-worked-through-example-ebc8629089b9)  
16. Learning to Rank — xgboost 2.0.3 documentation, accessed May 1, 2026, [https://xgboost.readthedocs.io/en/release\_2.0.0/tutorials/learning\_to\_rank.html](https://xgboost.readthedocs.io/en/release_2.0.0/tutorials/learning_to_rank.html)  
17. When “Rename Table” ≠ “Table Rename”: Solving the Query ..., accessed May 1, 2026, [https://medium.com/@hsmanojkumar2003/when-rename-table-table-rename-solving-the-query-variation-problem-in-llm-systems-ea374bdd093c](https://medium.com/@hsmanojkumar2003/when-rename-table-table-rename-solving-the-query-variation-problem-in-llm-systems-ea374bdd093c)  
18. Text-to-SQL: Comparison of LLM Accuracy \- AIMultiple, accessed May 1, 2026, [https://aimultiple.com/text-to-sql](https://aimultiple.com/text-to-sql)  
19. Text-to-SQL LLM : A Practical Guide \- PuppyGraph, accessed May 1, 2026, [https://www.puppygraph.com/blog/text-to-sql-llm](https://www.puppygraph.com/blog/text-to-sql-llm)  
20. LLM Re-ranking: Enhancing Search and Retrieval with AI \- DEV Community, accessed May 1, 2026, [https://dev.to/simplr\_sh/llm-re-ranking-enhancing-search-and-retrieval-with-ai-28b7](https://dev.to/simplr_sh/llm-re-ranking-enhancing-search-and-retrieval-with-ai-28b7)  
21. Why Retrieval Is the New Ranking: 12 LLM Methods Reshaping AI SEO \- SearchMinistry, accessed May 1, 2026, [https://searchministry.au/blog/llm-retrieval-methods-ai-seo](https://searchministry.au/blog/llm-retrieval-methods-ai-seo)  
22. Exploring LLM Capabilities in Extracting DCAT-Compatible Metadata for Data Cataloging, accessed May 1, 2026, [https://arxiv.org/html/2507.05282v1](https://arxiv.org/html/2507.05282v1)  
23. Lightweight LLM for converting text to structured data \- Amazon ..., accessed May 1, 2026, [https://www.amazon.science/blog/lightweight-llm-for-converting-text-to-structured-data](https://www.amazon.science/blog/lightweight-llm-for-converting-text-to-structured-data)  
24. Accelerating Data Literacy Using Machine Learning Data Catalogs \- Artefact, accessed May 1, 2026, [https://www.artefact.com/blog/accelerating-data-literacy-using-machine-learning-data-catalogs/](https://www.artefact.com/blog/accelerating-data-literacy-using-machine-learning-data-catalogs/)  
25. How to Structure Data for AI Indexing \- eSEOspace, accessed May 1, 2026, [https://eseospace.com/blog/how-to-structure-data-for-ai-indexing/](https://eseospace.com/blog/how-to-structure-data-for-ai-indexing/)  
26. Evaluation Metrics for Search and Recommendation Systems \- Weaviate, accessed May 1, 2026, [https://weaviate.io/blog/retrieval-evaluation-metrics](https://weaviate.io/blog/retrieval-evaluation-metrics)  
27. Evaluating recommendation systems (mAP, MMR, NDCG) \- Shaped.ai, accessed May 1, 2026, [https://www.shaped.ai/blog/evaluating-recommendation-systems-map-mmr-ndcg](https://www.shaped.ai/blog/evaluating-recommendation-systems-map-mmr-ndcg)  
28. Evaluation Measures in Information Retrieval | Pinecone, accessed May 1, 2026, [https://www.pinecone.io/learn/offline-evaluation/](https://www.pinecone.io/learn/offline-evaluation/)  
29. Metrics to evaluate Search Results | Niklas Heidloff, accessed May 1, 2026, [https://heidloff.net/article/search-evaluations/](https://heidloff.net/article/search-evaluations/)  
30. Improving Search Relevance: High-Quality Data with a Personal Touch | DataForce, accessed May 1, 2026, [https://www.dataforce.ai/blog/improving-search-relevance](https://www.dataforce.ai/blog/improving-search-relevance)  
31. What is Ground Truth in Machine Learning? | Domino Data Lab, accessed May 1, 2026, [https://domino.ai/data-science-dictionary/ground-truth](https://domino.ai/data-science-dictionary/ground-truth)  
32. What Is Ground Truth in Machine Learning? \- IBM, accessed May 1, 2026, [https://www.ibm.com/think/topics/ground-truth](https://www.ibm.com/think/topics/ground-truth)  
33. The Data Canary: How Netflix Validates Catalog Metadata, accessed May 1, 2026, [https://netflixtechblog.medium.com/the-data-canary-how-netflix-validates-catalog-metadata-18b699d58e36](https://netflixtechblog.medium.com/the-data-canary-how-netflix-validates-catalog-metadata-18b699d58e36)  
34. Ground truth generation and review best practices for evaluating generative AI question-answering with FMEval | Artificial Intelligence \- AWS, accessed May 1, 2026, [https://aws.amazon.com/blogs/machine-learning/ground-truth-generation-and-review-best-practices-for-evaluating-generative-ai-question-answering-with-fmeval/](https://aws.amazon.com/blogs/machine-learning/ground-truth-generation-and-review-best-practices-for-evaluating-generative-ai-question-answering-with-fmeval/)  
35. How do you implement query expansion for multimodal search? \- Milvus, accessed May 1, 2026, [https://milvus.io/ai-quick-reference/how-do-you-implement-query-expansion-for-multimodal-search](https://milvus.io/ai-quick-reference/how-do-you-implement-query-expansion-for-multimodal-search)  
36. Feature Ranking with Random Forests | CodeSignal Learn, accessed May 1, 2026, [https://codesignal.com/learn/courses/feature-selection-and-streamlining/lessons/feature-ranking-with-random-forests](https://codesignal.com/learn/courses/feature-selection-and-streamlining/lessons/feature-ranking-with-random-forests)  
37. Choice of relevance label for a learning to rank model \- Data Science Stack Exchange, accessed May 1, 2026, [https://datascience.stackexchange.com/questions/131241/choice-of-relevance-label-for-a-learning-to-rank-model](https://datascience.stackexchange.com/questions/131241/choice-of-relevance-label-for-a-learning-to-rank-model)

[image1]: <data:image/png;base64,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>
