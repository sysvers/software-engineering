# Webhooks & Event-Driven APIs

## What Are Webhooks?

Webhooks invert the request direction: instead of clients polling for updates, your service calls the client when something happens. They are HTTP callbacks — when an event occurs, you make an HTTP POST to a URL the client registered.

Webhooks are the simplest form of event-driven API. They require no persistent connections, no message brokers, and no special protocols — just HTTP.

## How Webhooks Work

```
1. Client registers a webhook URL:
   POST /webhooks
   {
     "url": "https://client.com/events",
     "events": ["order.shipped", "order.delivered"]
   }

2. Event occurs in your system (order shipped)

3. Your service POSTs to the registered URL:
   POST https://client.com/events
   Content-Type: application/json
   X-Webhook-Signature: sha256=abc123...
   X-Event-ID: evt_abc123
   X-Event-Type: order.shipped

   {
     "id": "evt_abc123",
     "type": "order.shipped",
     "created": "2024-03-15T10:30:00Z",
     "data": {
       "order_id": "order_456",
       "tracking_number": "ABC789",
       "carrier": "fedex"
     }
   }

4. Client responds with 2xx to acknowledge receipt

5. If client returns non-2xx or times out, retry with backoff
```

## Webhook Design Principles

### Event Envelope

Use a consistent envelope for all events:

```json
{
  "id": "evt_abc123",
  "type": "order.shipped",
  "api_version": "2024-03-15",
  "created": "2024-03-15T10:30:00Z",
  "data": {
    "object": {
      "id": "order_456",
      "status": "shipped",
      "tracking_number": "ABC789"
    }
  }
}
```

**Key fields:**
- `id` — Unique event identifier for deduplication
- `type` — Dot-separated event name (noun.verb pattern)
- `created` — When the event occurred (not when it was sent)
- `api_version` — Which API version the payload conforms to
- `data` — The event payload, containing the relevant object

### Event Naming

Use a `resource.action` naming convention:

```
order.created
order.updated
order.shipped
order.delivered
order.cancelled
payment.succeeded
payment.failed
invoice.finalized
customer.subscription.created
customer.subscription.deleted
```

Group related events by resource. Use past tense for completed actions. Avoid ambiguous names like `order.changed` — be specific about what changed.

## Retry Strategy

Webhook delivery will fail. Networks are unreliable, servers go down, and deployments happen. A robust retry strategy is essential.

### Exponential Backoff

```
Attempt 1: Immediate
Attempt 2: 1 minute later
Attempt 3: 5 minutes later
Attempt 4: 30 minutes later
Attempt 5: 2 hours later
Attempt 6: 8 hours later
Attempt 7: 24 hours later
... stop after N attempts (typically 5-10)
```

**Implementation considerations:**
- Set a reasonable timeout for each attempt (5-30 seconds)
- Consider a 2xx response as success, anything else as failure
- After all retries are exhausted, mark the webhook as failed
- Notify the webhook owner via email/dashboard after repeated failures
- Disable the endpoint after sustained failures (e.g., 3 consecutive days)

### Retry Queue Architecture

```
Event occurs
  → Write event to events table (durable)
  → Enqueue delivery job

Delivery worker:
  → Read job from queue
  → POST to webhook URL
  → If 2xx: mark delivered
  → If failure: requeue with backoff delay
  → If max retries exceeded: mark failed, alert owner
```

Use a persistent job queue (Postgres-backed, SQS, or similar), not an in-memory queue. Events must survive process restarts.

## Idempotency

Because webhooks are retried, clients will sometimes receive the same event multiple times. Events must be idempotent.

**Server-side:** Include a unique event ID in every webhook delivery. The same event always has the same ID, even across retries.

**Client-side:** Track processed event IDs and skip duplicates:

```python
def handle_webhook(event):
    if already_processed(event["id"]):
        return 200  # Acknowledge but skip processing

    process_event(event)
    mark_processed(event["id"])
    return 200
```

**Design events for idempotency:** Prefer "state" events (`order.status_changed` with the new status) over "delta" events (`order.quantity_increased_by_5`). State events are naturally idempotent — processing the same state twice produces the same result.

## Webhook Signatures

Webhook payloads must be verified to prevent spoofing. Without signatures, anyone who knows the webhook URL can send fake events.

### HMAC Signing

The standard approach:

```
Server-side (sending):
  1. Serialize the payload to JSON
  2. Compute HMAC-SHA256(payload, webhook_secret)
  3. Send the signature in a header

Client-side (verifying):
  1. Read the raw request body (before parsing JSON)
  2. Compute HMAC-SHA256(body, webhook_secret)
  3. Compare with the signature header using constant-time comparison
  4. Reject if signatures don't match
```

**Rust implementation for signing:**

```rust
use hmac::{Hmac, Mac};
use sha2::Sha256;
use hex;

type HmacSha256 = Hmac<Sha256>;

fn sign_payload(payload: &[u8], secret: &[u8]) -> String {
    let mut mac = HmacSha256::new_from_slice(secret)
        .expect("HMAC can take key of any size");
    mac.update(payload);
    let result = mac.finalize();
    format!("sha256={}", hex::encode(result.into_bytes()))
}

fn verify_signature(payload: &[u8], secret: &[u8], signature: &str) -> bool {
    let expected = sign_payload(payload, secret);
    // Constant-time comparison to prevent timing attacks
    constant_time_eq(expected.as_bytes(), signature.as_bytes())
}
```

### Timestamp Verification

Include a timestamp in the signature to prevent replay attacks:

```
signature = HMAC-SHA256(timestamp + "." + payload, secret)
header: X-Webhook-Signature: t=1710500000,v1=abc123...
```

The client verifies that the timestamp is recent (within 5 minutes) and that the signature matches. This prevents an attacker from replaying captured webhook payloads.

Stripe uses exactly this approach in their webhook signature verification.

## Event Logs

Provide an event log API so clients can recover from missed webhooks:

```
GET /events?type=order.shipped&created_after=2024-03-14T00:00:00Z&limit=100

{
  "data": [
    { "id": "evt_abc", "type": "order.shipped", "created": "...", "data": {...} },
    { "id": "evt_def", "type": "order.shipped", "created": "...", "data": {...} }
  ],
  "has_more": true,
  "next_cursor": "cursor_xyz"
}
```

**Why event logs matter:**
- Webhooks are "at least once" delivery — they can be missed
- Clients may have downtime and miss events
- Event logs enable clients to replay events after recovery
- Auditing and debugging require a history of all events
- Event logs are the source of truth; webhooks are notifications

## Real-World Examples

### Stripe Webhooks

Stripe's webhook system is the gold standard:

- **Event types:** Over 200 event types covering every state change (`payment_intent.succeeded`, `invoice.paid`, `customer.subscription.updated`)
- **Signature verification:** `Stripe-Signature` header with timestamp and HMAC-SHA256
- **Retry schedule:** Up to 3 days with exponential backoff
- **Event retrieval:** Full Events API for replaying missed events
- **Webhook endpoints:** Multiple endpoints per account, each subscribed to specific event types
- **Testing:** CLI tool (`stripe listen`) forwards events to localhost during development
- **Versioning:** Events conform to the account's API version

### Slack Event API

Slack's approach includes several notable patterns:

- **URL verification challenge:** When registering a webhook URL, Slack sends a challenge that must be echoed back — proving URL ownership
  ```json
  POST https://your-url.com
  { "type": "url_verification", "challenge": "abc123" }

  Response: { "challenge": "abc123" }
  ```
- **Retry behavior:** `X-Slack-Retry-Num` and `X-Slack-Retry-Reason` headers on retries
- **Request signing:** `X-Slack-Signature` using HMAC-SHA256 with a signing secret
- **Event subscriptions:** Configured per Slack app, scoped to specific event types and permissions

### GitHub Webhooks

- **Per-repository and per-organization** webhook configuration
- **Event filtering:** Subscribe to specific events (push, pull_request, issues, etc.)
- **Secret-based HMAC** signatures in `X-Hub-Signature-256`
- **Redelivery:** Manual redeliver from the GitHub webhook settings UI
- **Ping event:** Sent on webhook creation to verify the URL works

## Webhook vs Polling vs Streaming

| Approach | Latency | Complexity | Reliability |
|----------|---------|------------|-------------|
| **Polling** | High (interval-dependent) | Low | High (client controls) |
| **Webhooks** | Low (near real-time) | Medium | Medium (delivery not guaranteed) |
| **SSE** | Very low | Medium | Medium (connection drops) |
| **WebSocket** | Very low | High | Medium (connection management) |
| **gRPC streaming** | Very low | High | High (with reconnection) |

**Use webhooks when:**
- Notifying external systems of events
- Consumers are other web services (not browsers)
- Events are infrequent relative to polling cost
- You need a simple, widely-understood integration pattern

**Use polling when:**
- Webhook infrastructure is too complex for the use case
- Consumers are behind firewalls that block incoming connections
- You need strong consistency guarantees (poll and reconcile)

**Use streaming when:**
- Events are high-frequency
- Latency requirements are sub-second
- Consumers need continuous, ordered event streams

## Implementation Checklist

Building a webhook system? Ensure you have:

- [ ] Persistent event storage (database, not just queue)
- [ ] Unique event IDs for deduplication
- [ ] HMAC signature verification with timestamp
- [ ] Exponential backoff retry with jitter
- [ ] Maximum retry limit with alerting
- [ ] Event log API for replay
- [ ] Webhook endpoint management (create, update, delete, list)
- [ ] Event type filtering per endpoint
- [ ] Delivery logs visible to the consumer (dashboard)
- [ ] Automatic endpoint disabling after sustained failures
- [ ] Development/testing tools (CLI forwarding, test events)
- [ ] Monitoring: delivery success rate, latency, failure reasons
