# Pull-Based Ingestion

An ingestion architecture for search systems where search shards pull metadata from durable message streams (Kafka/Kinesis) rather than having producers push directly into the index. This decouples production rate from consumption rate, enabling backpressure handling, deterministic recovery, and resilience to bursty traffic.

## How it works

1. **Producers emit to durable stream** — metadata changes (schema updates, lineage re-computations) write to Kafka/Kinesis topics
2. **Shards pull from partitions** — each search shard maps to a specific stream partition
3. **Blocking queues** decouple polling from indexing threads, enabling parallel processing
4. **External versioning** — timestamps on metadata records prevent out-of-order overwrites
5. **Offset tracking** — each shard maintains pointers for deterministic replay after failure

## Why pull beats push

Push-based ingestion risks shard queue overflows during burst traffic (batch schema updates, lineage re-computations). Pull-based gives the consumer control over ingestion rate. Uber's transition to pull-based streaming in OpenSearch enabled deterministic recovery through per-shard bounded queues.

## Layered index management

For systems needing high freshness with large volumes, a tri-level architecture provides the best tradeoff:

| Layer | Persistence | Purpose |
|---|---|---|
| **Live Index** | Memory-resident | Immediate queryability for incoming updates |
| **Snapshot Index** | Flushed periodically (~30 min) | Relieves memory pressure |
| **Base Index** | Compacted from snapshots | Highly compressed, optimized for read |

**Tradeoff**: layered indices require every query operator to have parallel implementations across all layers, slowing feature adoption. For standard data discovery, NRT search in OpenSearch/Elasticsearch (which manages compaction internally) is often the more sustainable path.

## Ingestion modes

| Mode | Behavior | Use case |
|---|---|---|
| **Segment replication** | Ingest on primary shards only; replicas fetch segments | Lower CPU, slight visibility lag |
| **All-active** | Ingest on all shard copies | Near-instant visibility, higher compute |

## Key points
- Offset tracking enables deterministic recovery — replay from last good offset after failure
- External versioning is critical for correctness when messages arrive out of order
- The Live/Snapshot/Base pattern (used by Sia at Uber) trades engineering complexity for memory efficiency
- Most teams should default to Elasticsearch/OpenSearch NRT unless they have extreme freshness or volume requirements

## Connections
- [[data-catalog-search-architecture]] — ingestion is the foundational layer of the search stack
- [[search-ranking-signals]] — ingestion quality directly affects signal freshness and completeness

## Sources
- [[sources/papers/Improving Data Catalog Search Relevance]] — Uber's pull-based architecture, layered index management patterns
