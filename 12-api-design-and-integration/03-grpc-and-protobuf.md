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

```rust
// build.rs
fn main() -> Result<(), Box<dyn std::error::Error>> {
    tonic_build::compile_protos("proto/order.proto")?;
    Ok(())
}
```

### Server implementation

```rust
use tonic::{Request, Response, Status};
use order::order_service_server::{OrderService, OrderServiceServer};
use order::{CreateOrderRequest, GetOrderRequest, ListOrdersRequest, Order};

pub mod order {
    tonic::include_proto!("order");
}

#[derive(Default)]
pub struct MyOrderService {
    db: Arc<Database>,
}

#[tonic::async_trait]
impl OrderService for MyOrderService {
    async fn create_order(
        &self,
        request: Request<CreateOrderRequest>,
    ) -> Result<Response<Order>, Status> {
        let req = request.into_inner();

        if req.items.is_empty() {
            return Err(Status::invalid_argument("Order must have at least one item"));
        }

        let order = self.db.create_order(&req.user_id, &req.items)
            .await
            .map_err(|e| Status::internal(format!("Failed to create order: {e}")))?;

        Ok(Response::new(order))
    }

    async fn get_order(
        &self,
        request: Request<GetOrderRequest>,
    ) -> Result<Response<Order>, Status> {
        let id = &request.into_inner().id;

        self.db.get_order(id)
            .await
            .map(Response::new)
            .ok_or_else(|| Status::not_found(format!("Order {id} not found")))
    }

    // Server streaming: returns a stream of orders
    type ListOrdersStream = ReceiverStream<Result<Order, Status>>;

    async fn list_orders(
        &self,
        request: Request<ListOrdersRequest>,
    ) -> Result<Response<Self::ListOrdersStream>, Status> {
        let req = request.into_inner();
        let (tx, rx) = tokio::sync::mpsc::channel(32);
        let db = self.db.clone();

        tokio::spawn(async move {
            let orders = db.list_orders_for_user(&req.user_id).await;
            for order in orders {
                if tx.send(Ok(order)).await.is_err() {
                    break; // Client disconnected
                }
            }
        });

        Ok(Response::new(ReceiverStream::new(rx)))
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let addr = "[::1]:50051".parse()?;
    let service = MyOrderService::default();

    tonic::transport::Server::builder()
        .add_service(OrderServiceServer::new(service))
        .serve(addr)
        .await?;

    Ok(())
}
```

### Client usage

```rust
use order::order_service_client::OrderServiceClient;
use order::CreateOrderRequest;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut client = OrderServiceClient::connect("http://[::1]:50051").await?;

    let request = tonic::Request::new(CreateOrderRequest {
        user_id: "user_123".into(),
        items: vec![/* ... */],
    });

    let response = client.create_order(request).await?;
    println!("Created order: {:?}", response.into_inner());
    Ok(())
}
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
