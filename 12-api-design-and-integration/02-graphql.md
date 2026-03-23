# GraphQL

## What is GraphQL?

GraphQL is a query language for APIs and a runtime for fulfilling those queries. Created by Facebook in 2012 (open-sourced in 2015), it lets clients request exactly the data they need — no more, no less — through a single endpoint.

Unlike REST, where the server decides what data each endpoint returns, GraphQL puts the client in control of the response shape.

## The Problem GraphQL Solves

REST APIs often suffer from two inefficiencies:

**Over-fetching** — An endpoint returns more data than the client needs. A mobile app displaying a user's name and avatar gets the entire user object with 30 fields.

**Under-fetching** — Building one view requires multiple requests to different endpoints.

```
# REST: 3 requests to build a user profile page
GET /users/123          -> { id, name, email, avatar_url, ... }
GET /users/123/posts    -> [{ id, title, created_at, ... }, ...]
GET /users/123/followers -> [{ id, name, ... }, ...]

# GraphQL: 1 request
query {
  user(id: 123) {
    name
    avatarUrl
    posts(first: 5) {
      title
      createdAt
    }
    followersCount
  }
}
```

The client describes the exact shape of the data it needs, and the server returns precisely that — nothing more.

## Schema Design

The schema is the contract between client and server. It defines types, queries, mutations, and subscriptions.

```graphql
type User {
  id: ID!
  name: String!
  email: String!
  posts(first: Int, after: String): PostConnection!
  followersCount: Int!
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
  tags: [String!]!
  createdAt: DateTime!
}

# Relay-style connection for cursor-based pagination
type PostConnection {
  edges: [PostEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type PostEdge {
  node: Post!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

type Query {
  user(id: ID!): User
  posts(filter: PostFilter, first: Int, after: String): PostConnection!
  post(id: ID!): Post
}

input PostFilter {
  authorId: ID
  tag: String
  createdAfter: DateTime
}

type Mutation {
  createPost(input: CreatePostInput!): CreatePostPayload!
  updateUser(id: ID!, input: UpdateUserInput!): UpdateUserPayload!
  deletePost(id: ID!): DeletePostPayload!
}

input CreatePostInput {
  title: String!
  content: String!
  tags: [String!]
}

type CreatePostPayload {
  post: Post
  errors: [UserError!]!
}

type UserError {
  field: String!
  message: String!
}
```

**Schema design principles:**
- Use `!` for non-nullable fields — be intentional about what can be null
- Prefer Relay-style connections for paginated lists
- Return payload types from mutations (not raw objects) so you can include errors
- Use `input` types for mutation arguments
- Design from the client's perspective, not the database schema

## Resolvers

Resolvers are functions that populate each field in the schema. The GraphQL runtime calls resolvers to build the response.

```python
# Conceptual resolver structure (Python for clarity)
class Query:
    def user(self, info, id):
        return db.users.find_by_id(id)

class User:
    def posts(self, info, first=10, after=None):
        return db.posts.find_by_author(self.id, first=first, after=after)

    def followers_count(self, info):
        return db.followers.count(user_id=self.id)
```

Each field can have its own resolver. If no resolver is defined, the runtime uses a default that returns the field of the same name from the parent object.

## The N+1 Problem

The most critical performance issue in GraphQL.

```graphql
query {
  posts(first: 20) {
    edges {
      node {
        title
        author {    # This triggers a separate DB query per post
          name
        }
      }
    }
  }
}
```

A naive implementation: 1 query for 20 posts + 20 queries for each post's author = 21 queries. If multiple posts share an author, those are redundant queries.

### DataLoader

DataLoader solves N+1 by batching and deduplicating requests within a single execution cycle.

```python
# Without DataLoader: 20 individual queries
SELECT * FROM users WHERE id = 1;
SELECT * FROM users WHERE id = 2;
SELECT * FROM users WHERE id = 1;  # duplicate!
...

# With DataLoader: 1 batched query
SELECT * FROM users WHERE id IN (1, 2, 3, 5, 8);
```

**How DataLoader works:**
1. During a single GraphQL execution tick, all `author` resolvers request a user ID
2. DataLoader collects all requested IDs
3. At the end of the tick, it makes one batched query
4. Results are distributed back to the individual resolvers
5. Duplicate IDs are deduplicated automatically

DataLoader is available in every major language: `dataloader` (JS), `aiodataloader` (Python), `async-graphql` has built-in DataLoader support (Rust).

## Mutations

Mutations are how clients modify data. Design them carefully.

```graphql
mutation {
  createPost(input: { title: "GraphQL Guide", content: "...", tags: ["api"] }) {
    post {
      id
      title
      createdAt
    }
    errors {
      field
      message
    }
  }
}
```

**Mutation design guidelines:**
- Name mutations as verb + noun: `createPost`, `updateUser`, `cancelOrder`
- Accept a single `input` argument — easier to evolve
- Return a payload type with both the result and potential errors
- Make mutations as specific as possible: `archivePost` instead of generic `updatePost(input: { archived: true })`

## Subscriptions

Subscriptions enable real-time updates over WebSocket connections.

```graphql
subscription {
  postCreated(authorId: "123") {
    id
    title
    author {
      name
    }
  }
}
```

The server pushes updates to the client whenever a matching event occurs. Under the hood, this typically uses WebSockets (via `graphql-ws` protocol) with a pub/sub system (Redis, Kafka) on the server.

**When subscriptions work well:** Chat, notifications, live dashboards, collaborative editing.

**When they don't:** High-frequency data (stock tickers) — WebSockets add overhead vs. raw TCP/gRPC streaming.

## When GraphQL Beats REST

| Scenario | REST | GraphQL |
|----------|------|---------|
| Simple CRUD API | Simple, well-understood | Overkill |
| Mobile apps with varying data needs | Multiple endpoints or custom ones | Clients request exactly what they need |
| Public API for external developers | Familiar, easy to cache | Steeper learning curve |
| Complex, deeply nested data | Multiple round trips | Single query |
| File uploads | Native multipart support | Awkward (separate endpoint) |
| Real-time updates | Requires WebSockets separately | Built-in subscription support |
| Microservices aggregation | API gateway stitching | Federation composes schemas |

**GraphQL shines when:**
- Multiple client types (web, iOS, Android) need different data shapes
- The data model is deeply relational
- Teams want to iterate on the frontend without backend changes
- You need to aggregate data from multiple services (via federation)

**GraphQL is wrong when:**
- The API is simple CRUD with few consumers
- You need aggressive HTTP caching (GraphQL uses POST, harder to cache)
- File upload/download is a primary use case
- The team lacks GraphQL experience and the project is time-constrained

## GitHub's Dual API Approach

GitHub is the best example of running REST and GraphQL side by side:

- **REST API (v3)** — Stable, well-documented, used for simple operations. Every resource has a URL. Great for scripts, CLI tools, and simple integrations.
- **GraphQL API (v4)** — Used for complex queries. Fetch a repository with its issues, PRs, contributors, and labels in one request. Powers GitHub's own web frontend.

GitHub didn't replace REST with GraphQL — they added GraphQL for use cases where REST was inefficient. REST endpoints still exist, are maintained, and are the recommended starting point for most integrations.

**Key lesson:** REST and GraphQL are complementary, not competing. Choose based on the use case.

## GraphQL Security Considerations

GraphQL's flexibility creates unique security challenges:

- **Query depth limiting** — Prevent deeply nested queries that could cause exponential work: `{ user { friends { friends { friends { ... } } } } }`
- **Query complexity analysis** — Assign costs to fields and reject queries exceeding a budget
- **Rate limiting by query cost** — Not all queries are equal. A simple `{ user { name } }` is cheaper than `{ users(first: 100) { posts { comments { author } } } }`
- **Introspection in production** — Disable schema introspection in production to avoid leaking your full API surface
- **Persisted queries** — In production, only allow pre-approved query hashes. Eliminates arbitrary query execution.

## Rust: async-graphql Example

```rust
use async_graphql::*;

#[derive(SimpleObject)]
struct User {
    id: ID,
    name: String,
    email: String,
}

struct QueryRoot;

#[Object]
impl QueryRoot {
    async fn user(&self, ctx: &Context<'_>, id: ID) -> Result<Option<User>> {
        let db = ctx.data::<Database>()?;
        Ok(db.find_user(&id).await?)
    }

    async fn posts(
        &self,
        ctx: &Context<'_>,
        first: Option<i32>,
        after: Option<String>,
    ) -> Result<Connection<String, Post>> {
        // Relay-style cursor pagination built-in
        todo!()
    }
}

let schema = Schema::build(QueryRoot, MutationRoot, SubscriptionRoot)
    .limit_depth(10)
    .limit_complexity(200)
    .data(database)
    .finish();
```
