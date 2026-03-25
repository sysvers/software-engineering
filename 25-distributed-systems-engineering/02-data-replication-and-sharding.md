# Data Replication and Sharding

## Overview

Replication copies data across multiple nodes to improve availability and read throughput. Sharding splits data across nodes so no single node must store or process everything. These two techniques are complementary and often used together: data is sharded across groups, and each shard is replicated for durability.

---

## Replication Patterns

### Single-Leader (Primary-Replica)

One node accepts writes and replicates to followers. The simplest model and the default in PostgreSQL, MySQL, and MongoDB.

**How it works:**
1. Client sends write to the leader
2. Leader writes to its local log
3. Leader sends the change to followers (synchronously or asynchronously)
4. Followers apply the change

**Trade-offs:**
- Simple write ordering -- the leader defines a total order of operations
- Leader is a write bottleneck and single point of failure during failover
- Synchronous replication guarantees durability but adds latency
- Asynchronous replication is faster but risks data loss if the leader crashes before replication completes

**Failover concerns:** When the leader fails, a follower must be promoted. This involves detecting the failure, choosing a new leader (often the most up-to-date follower), and reconfiguring clients. Split-brain situations (two nodes both think they are leader) are the most dangerous failure mode.

### Multi-Leader

Multiple nodes accept writes and synchronize with each other. Useful for multi-datacenter deployments where you want local writes in each region.

**Conflict resolution strategies:**
- **Last-writer-wins (LWW):** Use timestamps to pick the "latest" write. Simple but lossy -- concurrent writes are silently discarded. Clock skew makes this unreliable.
- **CRDTs (Conflict-free Replicated Data Types):** Data structures that mathematically guarantee convergence without coordination. Examples: G-Counters, OR-Sets, LWW-Registers. Used in Riak, Redis CRDTs, and Automerge.
- **Application-level merging:** The application defines merge logic for conflicts. Most flexible but most complex.

**When to use:** Multi-datacenter deployments, offline-capable applications (each device is a "leader"), collaborative editing.

### Leaderless (Dynamo-Style)

Clients write to and read from multiple replicas. No single node is special. Consistency is tuned via quorum parameters.

**Quorum rule:** For strong consistency, W + R > N, where:
- N = total replicas
- W = number of nodes that must acknowledge a write
- R = number of nodes that must respond to a read

```text
/// Quorum parameters for a leaderless replication system.
STRUCTURE QuorumConfig
    num_replicas : Integer      // N
    write_quorum : Integer      // W
    read_quorum : Integer       // R

PROCEDURE QuorumConfig.NEW(n, w, r) → QuorumConfig
    ASSERT w + r > n, "W + R must exceed N for strong consistency"
    ASSERT w ≤ n AND r ≤ n, "Quorum sizes cannot exceed replica count"
    RETURN QuorumConfig { num_replicas ← n, write_quorum ← w, read_quorum ← r }

PROCEDURE QuorumConfig.IS_STRONGLY_CONSISTENT() → Boolean
    RETURN self.write_quorum + self.read_quorum > self.num_replicas

PROCEDURE QuorumConfig.WRITE_SUCCEEDED(acks) → Boolean
    RETURN acks ≥ self.write_quorum

PROCEDURE QuorumConfig.READ_SUCCEEDED(responses) → Boolean
    RETURN responses ≥ self.read_quorum
```

**Repair mechanisms:**
- **Read repair:** When a read returns stale data from some replicas, the client writes the newest version back to the stale replicas.
- **Anti-entropy:** Background processes compare replicas and reconcile differences using Merkle trees.

---

## Sharding Strategies

### Range-Based Sharding

Data is split by key ranges (e.g., users A-M on shard 1, N-Z on shard 2).

- Efficient range scans and natural data locality
- Prone to hotspots if key distribution is skewed (e.g., time-series data with recent timestamps as keys)
- Used by HBase, Google Bigtable, and early MongoDB

### Hash-Based Sharding

A hash function maps keys to shards uniformly.

- Distributes load evenly regardless of key distribution
- Range queries require scatter-gather across all shards
- Used by Redis Cluster, Cassandra (with token ranges), and DynamoDB

### Consistent Hashing

Nodes are placed on a hash ring. Keys are assigned to the next node clockwise on the ring. Adding or removing a node only redistributes a fraction of keys.

```text
/// A consistent hash ring supporting virtual nodes for better distribution.
STRUCTURE ConsistentHashRing
    ring : SortedMap<Integer, String>
    virtual_nodes_per_physical : Integer

PROCEDURE HASH_KEY(key) → Integer
    RETURN HASH(key)

PROCEDURE ConsistentHashRing.ADD_NODE(node_id)
    FOR i ← 0 TO self.virtual_nodes_per_physical - 1 DO
        virtual_key ← node_id + "#" + i
        hash ← HASH_KEY(virtual_key)
        self.ring[hash] ← node_id
    END FOR

PROCEDURE ConsistentHashRing.REMOVE_NODE(node_id)
    FOR i ← 0 TO self.virtual_nodes_per_physical - 1 DO
        virtual_key ← node_id + "#" + i
        hash ← HASH_KEY(virtual_key)
        DELETE self.ring[hash]
    END FOR

/// Find the node responsible for a given key.
PROCEDURE ConsistentHashRing.GET_NODE(key) → Optional<String>
    IF self.ring IS EMPTY THEN RETURN NULL
    hash ← HASH_KEY(key)
    // Find the first node with a hash ≥ the key's hash (clockwise).
    node ← FIRST ENTRY IN self.ring WHERE entry.key ≥ hash
    IF node IS NULL THEN
        node ← FIRST ENTRY IN self.ring    // Wrap around the ring.
    END IF
    RETURN node.value
```

**Virtual nodes:** Each physical node maps to multiple positions on the ring (e.g., 150-256 virtual nodes). This smooths out the distribution and prevents a single node from receiving a disproportionate share of keys when another node leaves.

---

## Real-World Examples

### Discord -- Message Storage Sharding

Discord stores billions of messages across a Cassandra cluster (later migrated to ScyllaDB). Their sharding approach:

- **Shard key:** `(channel_id, bucket)` where bucket is a time window. This co-locates messages from the same channel for efficient retrieval.
- **Problem encountered:** Large servers with millions of messages in a single channel created hotspots. A single partition could become too large for a single node.
- **Solution:** They introduced time-bucketed partitions, splitting a channel's messages across multiple partitions by time range. Recent messages (hot data) and historical messages (cold data) are accessed differently, and the bucketing aligns with these access patterns.
- **Migration from Cassandra to ScyllaDB:** Cassandra's garbage collection pauses caused latency spikes. ScyllaDB (a C++ rewrite of Cassandra's architecture) eliminated GC pauses while maintaining the same data model and sharding strategy.

### Instagram -- Sharded PostgreSQL

Instagram chose to shard PostgreSQL rather than adopt a NoSQL database, achieving scale while keeping the benefits of relational data modeling:

- **Shard key selection:** Each logical shard ID is embedded in the primary key using a custom ID generation scheme (similar to Snowflake IDs). The ID encodes a timestamp, shard ID, and auto-incrementing sequence.
- **Thousands of logical shards on fewer physical servers:** Logical shards are mapped to physical databases using a mapping table. Rebalancing moves logical shards between physical servers without changing the application's sharding logic.
- **Benefit:** Schema changes, foreign keys, and joins work within a shard. Cross-shard queries are avoided by aligning the shard key with the dominant access pattern (user-centric queries shard by user ID).

---

## Common Pitfalls

1. **Over-sharding too early:** Sharding introduces significant operational complexity -- cross-shard queries, rebalancing, and distributed joins. Many teams shard prematurely when a single well-tuned database with read replicas would suffice.

2. **Poor shard key choice:** A shard key that does not align with access patterns causes frequent cross-shard queries. A shard key with low cardinality creates hotspots.

3. **Ignoring rebalancing:** When you add or remove nodes, data must move. Without consistent hashing or a logical-to-physical mapping layer, rebalancing requires reshuffling the entire dataset.

4. **Replication lag assumptions:** Clients reading from followers may see stale data. If your application requires read-your-writes consistency, route reads to the leader or use session-sticky routing.

---

## Key Takeaways

- Single-leader replication is the right default. Use multi-leader or leaderless only when you have a clear requirement (multi-region writes, extreme availability).
- Quorum tuning (W, R, N) lets you trade consistency for availability and latency on a per-operation basis.
- Consistent hashing is essential for any system that needs elastic scaling. Virtual nodes are required for even distribution.
- Shard only when you have evidence a single node cannot handle the load. When you do, choose the shard key based on your dominant query pattern.
- Replication and sharding are orthogonal. A well-designed system shards for write scalability and replicates each shard for read scalability and durability.
