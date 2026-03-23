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

```rust
/// Quorum parameters for a leaderless replication system.
struct QuorumConfig {
    num_replicas: usize,      // N
    write_quorum: usize,      // W
    read_quorum: usize,       // R
}

impl QuorumConfig {
    fn new(n: usize, w: usize, r: usize) -> Self {
        assert!(w + r > n, "W + R must exceed N for strong consistency");
        assert!(w <= n && r <= n, "Quorum sizes cannot exceed replica count");
        Self {
            num_replicas: n,
            write_quorum: w,
            read_quorum: r,
        }
    }

    fn is_strongly_consistent(&self) -> bool {
        self.write_quorum + self.read_quorum > self.num_replicas
    }

    fn write_succeeded(&self, acks: usize) -> bool {
        acks >= self.write_quorum
    }

    fn read_succeeded(&self, responses: usize) -> bool {
        responses >= self.read_quorum
    }
}
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

```rust
use std::collections::BTreeMap;
use std::hash::{Hash, Hasher};
use std::collections::hash_map::DefaultHasher;

/// A consistent hash ring supporting virtual nodes for better distribution.
struct ConsistentHashRing {
    ring: BTreeMap<u64, String>,
    virtual_nodes_per_physical: usize,
}

impl ConsistentHashRing {
    fn new(virtual_nodes: usize) -> Self {
        Self {
            ring: BTreeMap::new(),
            virtual_nodes_per_physical: virtual_nodes,
        }
    }

    fn hash_key(key: &str) -> u64 {
        let mut hasher = DefaultHasher::new();
        key.hash(&mut hasher);
        hasher.finish()
    }

    fn add_node(&mut self, node_id: &str) {
        for i in 0..self.virtual_nodes_per_physical {
            let virtual_key = format!("{}#{}", node_id, i);
            let hash = Self::hash_key(&virtual_key);
            self.ring.insert(hash, node_id.to_string());
        }
    }

    fn remove_node(&mut self, node_id: &str) {
        for i in 0..self.virtual_nodes_per_physical {
            let virtual_key = format!("{}#{}", node_id, i);
            let hash = Self::hash_key(&virtual_key);
            self.ring.remove(&hash);
        }
    }

    /// Find the node responsible for a given key.
    fn get_node(&self, key: &str) -> Option<&String> {
        if self.ring.is_empty() {
            return None;
        }
        let hash = Self::hash_key(key);
        self.ring
            .range(hash..)
            .next()
            .or_else(|| self.ring.iter().next())
            .map(|(_, node)| node)
    }
}
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
