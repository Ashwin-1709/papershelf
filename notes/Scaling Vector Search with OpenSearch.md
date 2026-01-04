## Brushup on Lucene & OpenSearch
### Key terms
- Node - A single instance of the opensearch server like a librarian. Nodes can communicate with each other to access data owned by other nodes.
  - Specialized nodes: master, coordinator, data & ingest.
- Shard - A single instance  of the lucene where data is stored like a library shelf. Node is what allows your data to be accessed
- Cluster - A single instance of a library.

### Component Hierarchy
Index $\to$ Shard $\to$ Segment: If an **Index** is a library and a **Shard** is a bookshelf then **Segment** is the actual book.
- Shard is collection of many smaller **immutable** segments which are merged by a compaction process. Any update leads to creation of a new segment which are over time merged.
- Lucene uses some data structure inspired by LSM trees and supports sequential or concurrent search over segments.
- When a query reaches a shard, it will search in all segments and append to a temporary list which is further ranked and returned to the node.

# Billion-Scale Vector Search Notes

## 1. High-Level Architecture & Stack
* **Engine:** OpenSearch with **Meta® FAISS** (chosen for algorithm flexibility and GPU potential over standard Lucene).
* **Workflow:** Raw Data → **Apache Hive** → **Apache Spark** (Bulk Indexing) → **OpenSearch**.
* **Scale:** 1.5 Billion documents, ~400 dimensions per vector.
* **Hardware:** 80 Data Nodes (24 cores, 200GB RAM, 1TB Disk).

---

## 2. Ingestion & Indexing Optimization (79% Speedup)
*Goal: Reduce ingestion time from 12.5h to 2.5h.*

### A. Maximizing CPU Utilization
* **Parallelism:** Increased Spark cores, executors, and partitions to saturate cluster capacity.
* **Native Threading:** Increased `knn.algo_param.index_thread_qty` to utilize more CPU cores during HNSW graph construction.

### B. Reducing I/O Amplification (The SSTable/Segment Strategy)
* **Disable Refresh:** Set `index.refresh_interval: -1` to prevent the creation of tiny, unmerged segments during bulk loads:
  * **Refresh** parameter controls when transition from RAM to Disk actually happens.
  * Write Request $\to$ OpenSearch $\to$ WAL & RAM $\to$ Disk
  * OpenSearch will never automatically turn that RAM buffer into a segment.
* **Flush Tuning:** Increased `flush_threshold_size` (from 518MB to 1024MB) and set flush sync to 120s to reduce disk write frequency.
  * After memory buffer exceeds flush threshold, the data is written to disk.
* **Merge Policy:** Increased `segments_per_tier` and `max_merge_at_once` to 15 to reduce background compaction overhead.
* **Storage Reduction:** Disabled `_source` and `doc_values`, shrinking index size from **11TB to 4TB**.

---

## 3. Query Performance Tuning (52% Latency Reduction)
*Goal: Meet strict 100ms P99 latency at 2K QPS.*

### A. Shard & Node Mapping
* **The Sweet Spot:** Found optimal performance when **Shard Count = Node Count** (80 shards for 80 nodes).
  * Oversharding (shards > nodes): Increases parallelism but merging of results due to divide and conquer can toss out the gains from parallelism.
  * Undersharding (nodes > shards): Some nodes are idle, underutilization of usage.
* **Impact:** Maximizes hardware parallelism while minimizing the overhead of result aggregation (Gather phase).

### B. High Availability & Replicas
* **Replica Count:** Increasing replicas improved overall performance by "smoothing out" latency spikes from slower nodes (tail latency).

### C. Memory Management
* **RAM Residency:** HNSW graphs must stay entirely in memory. If the circuit breaker (`knn.memory.circuit_breaker.limit`) triggers, latency jumps to **tens of seconds** due to disk swapping.
  * More context: OpenSearch node divides RAM into two distinct buckets - JVM & Off-Heap/Native where the graph is actually stored. The query is faster when the graph traversal is mostly in memory but we need to have a bound as the graph might be very huge.

---

## 4. Concurrent Segment Search (CSS) Insights
* **Findings:** In this specific 1.5B scale, CSS actually **decreased** saturated QPS (from 10K to 7K) because the search space per shard was already small.
* **Best Use Cases:** CSS is recommended when shards are very large, compute cores are underutilized, or there are many segments per shard.

---

## 5. Stability & Deployment
* **Blue/Green Strategy:** Uber builds the massive index on a **separate cluster** and swaps traffic via a config switch (`ObjectConfig`) to prevent ingestion load from impacting search latency.

---