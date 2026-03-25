# gRPC & Protocol Buffers

## What is gRPC?

gRPC (Google Remote Procedure Call) is a high-performance RPC framework that uses Protocol Buffers for serialization and HTTP/2 for transport. Unlike REST's resource-oriented style, gRPC is explicitly procedure-oriented: you define services with methods, and calling them feels like local function calls.

gRPC is designed for efficient, typed, service-to-service communication — not for browsers or public APIs.

## Protocol Buffers

Protocol Buffers (protobuf) are Google's language-neutral, platform-neutral mechanism for serializing structured data. They serve as both the interface definition language (IDL) and the wire format for gRPC.

```protobuf
syntax = "proto3";

package order;

import "google/protobuf/timestamp.proto";

service OrderService {
  rpc CreateOrder(CreateOrderRequest) returns (Order);
  rpc GetOrder(GetOrderRequest) returns (Order);
  rpc ListOrders(ListOrdersRequest) returns (stream Order);        // Server streaming
  rpc UploadItems(stream OrderItem) returns (UploadSummary);       // Client streaming
  rpc LiveUpdates(stream OrderQuery) returns (stream OrderUpdate); // Bidirectional
}

message CreateOrderRequest {
  string user_id = 1;
  repeated OrderItem items = 2;
}

message GetOrderRequest {
  string id = 1;
}

message ListOrdersRequest {
  string user_id = 1;
  int32 page_size = 2;
  string page_token = 3;
}

message Order {
  string id = 1;
  string user_id = 2;
  repeated OrderItem items = 3;
  OrderStatus status = 4;
  google.protobuf.Timestamp created_at = 5;
}

message OrderItem {
  string product_id = 1;
  int32 quantity = 2;
  int64 price_cents = 3;
}

enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;
  ORDER_STATUS_PENDING = 1;
  ORDER_STATUS_CONFIRMED = 2;
  ORDER_STATUS_SHIPPED = 3;
  ORDER_STATUS_DELIVERED = 4;
}
```

**Key protobuf concepts:**
- Field numbers (1, 2, 3) are the wire identity — never reuse or change them
- `repeated` = list/array
- `optional` fields were reintroduced in proto3 for explicit presence tracking
- `oneof` for mutually exclusive fields
- Nested messages for grouping related fields
- Use `google.protobuf.Timestamp`, `Duration`, `Struct` for common types
- Enums must have a zero value (convention: `_UNSPECIFIED = 0`)

**Why protobuf over JSON?**
- Binary format: 3-10x smaller than JSON on the wire
- Schema-enforced: types are known at compile time, not runtime
- Code generation: clients and servers generated from the same `.proto` file
- Backward/forward compatible: add fields without breaking existing clients

## Streaming Patterns

gRPC's use of HTTP/2 enables four communication patterns.

### Unary RPC (Request-Response)
The simplest pattern, equivalent to a REST call.
```
Client --[CreateOrderRequest]--> Server
Client <--[Order]--------------- Server
```

### Server Streaming
Server sends a stream of messages in response to a single client request. Use for: real-time feeds, large result sets, progress updates.
```
Client --[ListOrdersRequest]--> Server
Client <--[Order]-------------- Server
Client <--[Order]-------------- Server
Client <--[Order]-------------- Server
```

### Client Streaming
Client sends a stream of messages; server responds once after the stream ends. Use for: file uploads, batch ingestion, aggregation.
```
Client --[OrderItem]--> Server
Client --[OrderItem]--> Server
Client --[OrderItem]--> Server
Client ---- END ------> Server
Client <--[Summary]---- Server
```

### Bidirectional Streaming
Both sides send streams independently. Use for: chat, collaborative editing, real-time sync.
```
Client --[Query]-----> Server
Client <--[Update]---- Server
Client --[Query]-----> Server
Client <--[Update]---- Server
Client <--[Update]---- Server
```

## tonic in Rust

`tonic` is the de facto gRPC framework for Rust. It generates Rust code from `.proto` files at build time.

### Build configuration

```toml
# Cargo.toml
[dependencies]
tonic = "0.12"
prost = "0.13"
tokio = { version = "1", features = ["full"] }

[build-dependencies]
tonic-build = "0.12"
```

```text
// build.rs
PROCEDURE MAIN():
    COMPILE_PROTOS("proto/order.proto")
```

### Server implementation

```text
// Include generated protobuf code for "order" package

STRUCTURE MyOrderService:
    db ← shared reference to Database

IMPLEMENT OrderService FOR MyOrderService:

    PROCEDURE CREATE_ORDER(request):
        req ← EXTRACT inner data FROM request

        IF req.items IS empty THEN
            RETURN Error("Order must have at least one item")

        order ← AWAIT self.db.CREATE_ORDER(req.user_id, req.items)
        IF order FAILED THEN
            RETURN Error("Failed to create order: " + error message)

        RETURN Response(order)

    PROCEDURE GET_ORDER(request):
        id ← EXTRACT inner data FROM request, GET id

        result ← AWAIT self.db.GET_ORDER(id)
        IF result IS present THEN
            RETURN Response(result)
        ELSE
            RETURN Error("Order {id} not found")

    // Server streaming: returns a stream of orders
    PROCEDURE LIST_ORDERS(request):
        req ← EXTRACT inner data FROM request
        CREATE channel (sender, receiver) with buffer size 32
        db ← CLONE self.db

        SPAWN async task:
            orders ← AWAIT db.LIST_ORDERS_FOR_USER(req.user_id)
            FOR EACH order IN orders:
                IF SEND(order) TO sender FAILS THEN
                    BREAK  // Client disconnected

        RETURN Response(stream FROM receiver)

PROCEDURE MAIN():
    addr ← "[::1]:50051"
    service ← NEW MyOrderService (default)

    BUILD gRPC Server
        ADD OrderServiceServer(service)
        SERVE at addr
```

### Client usage

```text
PROCEDURE MAIN():
    client ← AWAIT CONNECT OrderServiceClient TO "http://[::1]:50051"

    request ← NEW CreateOrderRequest {
        user_id ← "user_123",
        items ← [...]
    }

    response ← AWAIT client.CREATE_ORDER(request)
    PRINT "Created order:", response
```

## When to Use gRPC

**Use gRPC for:**
- Internal service-to-service communication (not browser-facing)
- High-throughput, low-latency requirements (binary protocol, HTTP/2 multiplexing)
- Streaming data (server-side, client-side, or bidirectional)
- When type safety and code generation save development time
- Polyglot environments (generate clients in any language from `.proto`)
- Mobile clients on constrained networks (smaller payloads)

**Don't use gRPC for:**
- Public APIs consumed by web browsers (gRPC-Web exists but adds complexity)
- APIs where human readability of requests/responses matters (debugging, curl)
- Simple CRUD where REST is sufficient and the team knows it well
- When you need HTTP caching (CDNs, browser caches don't understand gRPC)

## Real-World Usage

**Google** uses gRPC internally for virtually all inter-service communication. Google Cloud APIs expose gRPC alongside REST.

**Netflix** uses gRPC for communication between microservices, particularly where low-latency streaming is critical.

**Dropbox** migrated from a custom RPC framework to gRPC, citing code generation and cross-language support as key drivers.

**Buf** (buf.build) provides modern tooling for Protocol Buffers: linting, breaking change detection, and a schema registry. If you're using gRPC seriously, Buf replaces the aging `protoc` compiler workflow.

## gRPC vs REST vs GraphQL

| Dimension | REST | GraphQL | gRPC |
|-----------|------|---------|------|
| Wire format | JSON (text) | JSON (text) | Protobuf (binary) |
| Transport | HTTP/1.1 or 2 | HTTP/1.1 or 2 | HTTP/2 |
| Schema | OpenAPI (optional) | GraphQL schema (required) | Protobuf (required) |
| Streaming | No (WebSockets separate) | Subscriptions | Native (4 patterns) |
| Browser support | Native | Native | Requires gRPC-Web |
| Code generation | Optional | Optional | Built-in |
| Human readable | Yes | Yes | No (binary) |
| Best for | Public APIs | Flexible client queries | Service-to-service |

## Proto Best Practices

- **Never reuse field numbers** — even after deleting a field, reserve the number
- **Use `reserved`** to prevent accidental reuse: `reserved 3, 5; reserved "old_field";`
- **Prefix enum values** with the enum name: `ORDER_STATUS_PENDING`, not `PENDING`
- **Use wrapper types** for optional primitives: `google.protobuf.Int32Value` when you need to distinguish 0 from "not set"
- **Design for evolution** — add fields freely, never change field types or numbers
- **Keep `.proto` files in a shared repository** or use Buf's schema registry
