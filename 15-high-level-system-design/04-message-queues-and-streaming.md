# Message Queues and Event Streaming

## The Problem: Direct Service Communication

When Service A calls Service B synchronously, they are tightly coupled. If B is slow, A is slow. If B is down, A fails. If A produces work faster than B can consume it, requests pile up and both degrade.

Message queues and event streaming decouple producers from consumers. The producer sends a message and moves on. The consumer processes it at its own pace. This decoupling is foundational to building resilient, scalable distributed systems.

## Message Queues: Point-to-Point

A message queue delivers each message to exactly one consumer. Once a consumer acknowledges a message, it is removed from the queue. This is the task distribution pattern.

```
Producer ──→ [Queue] ──→ Consumer 1
                    ──→ Consumer 2  (each message goes to ONE consumer)
                    ──→ Consumer 3
```

### How They Work

1. A producer publishes a message to a named queue
2. The queue stores the message durably (on disk or in memory, depending on configuration)
3. A consumer pulls the message (or the broker pushes it)
4. The consumer processes the message and sends an acknowledgment (ACK)
5. The broker removes the acknowledged message from the queue

If the consumer crashes before ACKing, the message becomes visible again and another consumer picks it up. This provides at-least-once delivery.

### Key Implementations

**RabbitMQ:**
- Implements the AMQP protocol with rich routing via exchanges (direct, topic, fanout, headers)
- Supports message priorities, dead letter queues, and TTL per message
- Messages are typically deleted after consumption
- Excellent for complex routing patterns where different messages go to different queues based on routing keys
- Erlang-based, strong on reliability, moderate throughput (tens of thousands of messages per second per node)

**Amazon SQS:**
- Fully managed, zero operational overhead
- Two flavors: Standard (at-least-once, best-effort ordering, nearly unlimited throughput) and FIFO (exactly-once, strict ordering, 3,000 messages/second with batching)
- Visibility timeout model: a message becomes invisible to other consumers while one is processing it. If the consumer does not delete it within the timeout, it reappears
- Dead letter queues built in for messages that repeatedly fail processing
- No server to manage, no capacity to plan — it just scales

**Redis Streams (lightweight alternative):**
- Built into Redis, useful when you already have Redis deployed
- Supports consumer groups (multiple consumers sharing work)
- Lower operational overhead than deploying a dedicated message broker
- Not as durable or feature-rich as RabbitMQ or SQS for critical workloads

### Use Cases for Message Queues

- **Background job processing:** Email sending, PDF generation, image resizing. The web server enqueues the job and responds to the user immediately
- **Order fulfillment pipelines:** An order is placed and a message triggers payment processing, then inventory reservation, then shipping label creation — each as a separate step
- **Rate smoothing:** If your API receives 10,000 requests per second but your downstream service handles 1,000, the queue absorbs the burst
- **Retry with backoff:** Failed messages go to a dead letter queue for inspection or automatic retry with exponential backoff

## Event Streaming: Publish/Subscribe with Persistence

Event streaming is fundamentally different from message queues. Events are published to a topic, stored durably (for hours, days, or indefinitely), and consumed by multiple independent consumer groups. Each consumer group gets every event.

```
Producer ──→ [Topic: "orders"] ──→ Consumer Group A (Inventory Service)
                               ──→ Consumer Group B (Analytics Pipeline)
                               ──→ Consumer Group C (Email Notifications)
                               ──→ Consumer Group D (Fraud Detection)
```

### How They Work

1. A producer publishes an event to a topic
2. The topic is divided into partitions (for parallelism and ordering)
3. Each event is appended to a partition and assigned a sequential offset
4. Consumer groups independently track their position (offset) in each partition
5. Events are NOT deleted after consumption — they remain for the configured retention period
6. A new consumer group can start from the beginning and replay the entire history

### Key Implementations

**Apache Kafka:**
- The de facto standard for event streaming, created at LinkedIn in 2011
- Topics are divided into partitions, distributed across brokers for horizontal scaling
- Events within a partition are strictly ordered; cross-partition ordering is not guaranteed
- Throughput: millions of events per second per cluster
- Retention: configurable from hours to infinite (with compaction). Compacted topics keep the latest value per key indefinitely
- Ecosystem: Kafka Connect (integrations), Kafka Streams (stream processing), ksqlDB (SQL over streams)
- Operational complexity is significant — managing ZooKeeper (or KRaft in newer versions), broker configurations, partition rebalancing

**Redpanda:**
- Kafka API-compatible but written in C++ (no JVM, no ZooKeeper)
- Simpler operations: single binary, built-in Raft consensus
- Lower tail latency (no JVM garbage collection pauses)
- Suitable as a drop-in replacement for Kafka with lower operational burden

**Amazon Kinesis:**
- AWS-managed event streaming
- Simpler than Kafka but less flexible (fixed shard model, 7-day max retention without enhanced fan-out)
- Good for AWS-native architectures where operational simplicity matters more than flexibility

### Partitions and Ordering

A topic is split into partitions. Events with the same partition key always go to the same partition, guaranteeing order for that key.

```
Topic: "orders" (3 partitions)

Partition 0: [order-101, order-104, order-107, ...]  ← user-A's orders
Partition 1: [order-102, order-105, order-108, ...]  ← user-B's orders
Partition 2: [order-103, order-106, order-109, ...]  ← user-C's orders
```

Each consumer in a consumer group is assigned one or more partitions. Within a partition, events are processed in order. This gives you per-key ordering (all events for user-A are in order) without requiring global ordering (which would prevent parallelism).

### Consumer Offsets and Replay

Each consumer group tracks its offset per partition — the position of the last consumed event. This enables:

- **Replay:** A new analytics service can join and process all historical events from offset 0
- **Reprocessing:** If a bug corrupted data, fix the code, reset the offset, and reprocess
- **Independent progress:** The inventory service can be 5 seconds behind while the analytics pipeline processes events from 3 hours ago — they do not block each other

## Head-to-Head Comparison

| Feature | Message Queue (RabbitMQ/SQS) | Event Streaming (Kafka/Redpanda) |
|---------|------------------------------|----------------------------------|
| **Delivery** | Each message to one consumer | Each event to all consumer groups |
| **After consumption** | Message deleted | Event retained for replay |
| **Ordering** | FIFO within a queue | Ordered within a partition |
| **Replay** | Not possible (message is gone) | Replay from any offset |
| **Throughput** | Thousands-tens of thousands/sec | Millions/sec |
| **Routing** | Rich (exchanges, routing keys) | Simple (topic + partition key) |
| **Consumer coupling** | Producer knows the queue | Producer does not know consumers |
| **Operational complexity** | Moderate (RabbitMQ) / Low (SQS) | High (Kafka) / Moderate (Redpanda) |
| **Best mental model** | Task inbox | Append-only event log |

## When to Use Each

**Choose a message queue when:**
- You need task distribution — each job processed by exactly one worker
- You need complex routing (message A goes to queue X, message B goes to queue Y based on content)
- Message order across all messages matters (use SQS FIFO)
- You want simple operations and your throughput is moderate
- Messages have no value after processing (a job is done, discard it)

**Choose event streaming when:**
- Multiple services need the same events independently
- You need event replay (reprocessing, new consumers catching up)
- You are building an event-driven or event-sourced architecture
- Throughput requirements are very high (hundreds of thousands to millions of events per second)
- You want an audit log or event history as the source of truth
- Consumers should be decoupled from producers (producer does not know or care who consumes)

## Real-World Examples

### LinkedIn and Kafka

Kafka was born at LinkedIn to solve a specific problem: their data pipeline was a tangled mess of point-to-point connections between systems. Each new data consumer required a new integration. The solution was a central event bus — all data changes are published to Kafka topics, and any system that needs the data subscribes independently.

Today, LinkedIn's Kafka clusters handle over 7 trillion messages per day. Use cases include:
- Activity tracking (profile views, searches, clicks)
- Metrics and monitoring pipelines
- Change data capture from databases to search indexes
- Real-time recommendations

The key insight: Kafka turned their data architecture from an O(n^2) integration problem (every producer connected to every consumer) into an O(n) problem (every system connects to Kafka).

### Uber's Event-Driven Architecture

Uber uses Kafka extensively for their real-time systems:
- **Trip events:** Every state change (requested, matched, in-progress, completed) is an event in Kafka
- **Driver location updates:** Millions of GPS pings per second flow through Kafka to the matching and ETA services
- **Surge pricing:** Real-time supply/demand events feed the dynamic pricing engine

Uber built their own Kafka management platform (uReplicator) to handle cross-datacenter replication, ensuring events are available in all regions for disaster recovery.

### Amazon SQS at Scale

Amazon itself uses SQS internally for decoupling services. A notable public example: when you place an order on Amazon, the order confirmation is returned immediately. Behind the scenes, an SQS message triggers a chain of asynchronous processing — payment capture, inventory allocation, warehouse picking, shipping label generation. Each step is a separate consumer reading from a queue, allowing independent scaling and failure isolation.

## Common Pitfalls

- **Using Kafka when SQS would suffice.** If you have one producer and one consumer processing background jobs, Kafka's operational overhead is not justified. Use SQS or RabbitMQ.
- **Ignoring poison messages.** A malformed message that crashes the consumer will be retried forever. Always configure dead letter queues with a maximum retry count.
- **Assuming ordering across partitions.** Kafka only guarantees order within a partition. If you need global ordering, you need a single partition — which eliminates parallelism. Design your partition key so that events that must be ordered share a key.
- **Not monitoring consumer lag.** Consumer lag (the difference between the latest offset and the consumer's offset) tells you if consumers are keeping up. Growing lag means you need more consumer instances or faster processing.
- **Unbounded retention without compaction.** Keeping all events forever without log compaction will exhaust disk space. Use compaction for topics where only the latest value per key matters.
- **Fire-and-forget publishing.** If the producer does not wait for an ACK from the broker, messages can be silently lost. Configure appropriate ACK levels (Kafka: `acks=all` for critical data).

## Key Takeaways

1. Message queues are for task distribution (one consumer per message). Event streaming is for event broadcasting (all consumer groups get every event).
2. Event streaming's killer feature is replay — the ability for new consumers to process historical events.
3. Kafka is the industry standard for event streaming but carries significant operational cost. Consider Redpanda for simpler operations or managed services (Confluent Cloud, Amazon MSK) to offload operations.
4. Partition keys determine ordering guarantees. Design them carefully based on your ordering requirements.
5. Start with a message queue (SQS is nearly zero-ops). Graduate to event streaming when you have multiple consumers needing the same data or require replay.
6. Monitor consumer lag relentlessly — it is the primary health metric for event streaming systems.
