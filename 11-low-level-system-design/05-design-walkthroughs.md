# Design Walkthroughs

Step-by-step low-level design for three common systems. Each walkthrough follows the same structure: requirements, module design, key data structures, and Rust implementation.

## URL Shortener

### Requirements

- Shorten URLs (long URL in, short code out).
- Redirect short URLs to originals.
- Track click counts.
- Short codes expire optionally.

### Module Design

```
src/
├── api/handlers.rs       # POST /shorten, GET /:code
├── application/url_service.rs
├── domain/short_url.rs   # ShortUrl, ShortCode
└── infrastructure/url_repo.rs
```

### Domain Layer

```text
RECORD ShortUrl
    code: ShortCode
    original_url: Url
    created_at: DateTime
    expires_at: Optional<DateTime>
    click_count: Integer

TYPE ShortCode = WRAPPER(String)  // Newtype for type safety

FUNCTION GENERATE_SHORT_CODE() → ShortCode
    // Base62 encoding of random bytes → 7-char code
    // 62^7 = 3.5 trillion possibilities
    RETURN ShortCode(RANDOM_NANOID(length ← 7, alphabet ← BASE62))

FUNCTION IS_EXPIRED(url: ShortUrl) → Boolean
    IF url.expires_at IS NOT NONE THEN
        RETURN NOW() > url.expires_at
    ELSE
        RETURN FALSE
```

### Repository Trait

```text
INTERFACE UrlRepository
    ASYNC FUNCTION SAVE(url: ShortUrl) → Result<Void, RepositoryError>
    ASYNC FUNCTION FIND_BY_CODE(code: ShortCode) → Result<Optional<ShortUrl>, RepositoryError>
    ASYNC FUNCTION INCREMENT_CLICKS(code: ShortCode) → Result<Void, RepositoryError>
```

### Application Layer

```text
RECORD UrlService<R: UrlRepository>
    repo: R

ASYNC FUNCTION SHORTEN(svc: UrlService, original: String) → Result<ShortUrl, UrlError>
    url ← PARSE_URL(original)
    IF url IS INVALID THEN RETURN Error(InvalidUrl)
    code ← GENERATE_SHORT_CODE()
    short_url ← NEW ShortUrl(code, url)
    AWAIT svc.repo.SAVE(short_url)?
    RETURN short_url

ASYNC FUNCTION RESOLVE(svc: UrlService, code: String) → Result<Url, UrlError>
    code ← ShortCode(code)
    short_url ← AWAIT svc.repo.FIND_BY_CODE(code)?
    IF short_url IS NONE THEN RETURN Error(NotFound)

    IF IS_EXPIRED(short_url) THEN
        RETURN Error(Expired)

    AWAIT svc.repo.INCREMENT_CLICKS(code)?
    RETURN short_url.original_url
```

### API Layer

```text
ASYNC FUNCTION CREATE_SHORT_URL(svc: UrlService, req: ShortenRequest) → Result<ShortenResponse, AppError>
    short_url ← AWAIT SHORTEN(svc, req.url)?
    RETURN ShortenResponse {
        short_url ← "https://sho.rt/" + short_url.code,
        expires_at ← short_url.expires_at
    }

ASYNC FUNCTION REDIRECT(svc: UrlService, code: String) → Result<Redirect, AppError>
    original ← AWAIT RESOLVE(svc, code)?
    RETURN TEMPORARY_REDIRECT(original)
```

### Design Decisions

- **ShortCode as a newtype** prevents accidentally passing a user ID or other string where a short code is expected.
- **Click counting in `resolve`** keeps the side effect in the application layer, not the API layer.
- **Expiry check in application layer** (not database query) so the domain rule is testable without a database.

---

## Rate Limiter

### Requirements

- Limit requests per client (by IP or API key) within a time window.
- Configurable limits per endpoint.
- Return remaining quota in response headers.
- Must handle high concurrency.

### Sliding Window Counter Approach

```text
RECORD RateLimiterConfig
    max_requests: Integer
    window: Duration

RECORD RateLimiter
    config: RateLimiterConfig
    windows: Shared<ReadWriteLock<Map<IpAddr, SlidingWindow>>>

RECORD SlidingWindow
    timestamps: List<Instant>

FUNCTION COUNT_IN_WINDOW(sw: SlidingWindow, window: Duration) → Integer
    cutoff ← CURRENT_INSTANT() - window
    REMOVE all t FROM sw.timestamps WHERE t ≤ cutoff
    RETURN LENGTH(sw.timestamps)

FUNCTION RECORD(sw: SlidingWindow)
    APPEND CURRENT_INSTANT() TO sw.timestamps
```

### Rate Limiter Implementation

```text
RECORD RateLimitResult
    allowed: Boolean
    remaining: Integer
    reset_at: Instant

ASYNC FUNCTION CHECK(limiter: RateLimiter, ip: IpAddr) → RateLimitResult
    AWAIT ACQUIRE WRITE LOCK ON limiter.windows
    window ← windows.GET_OR_INSERT(ip, NEW SlidingWindow { timestamps ← [] })

    current ← COUNT_IN_WINDOW(window, limiter.config.window)

    IF current < limiter.config.max_requests THEN
        RECORD(window)
        RETURN RateLimitResult {
            allowed ← TRUE,
            remaining ← limiter.config.max_requests - current - 1,
            reset_at ← CURRENT_INSTANT() + limiter.config.window
        }
    ELSE
        RETURN RateLimitResult {
            allowed ← FALSE,
            remaining ← 0,
            reset_at ← CURRENT_INSTANT() + limiter.config.window
        }
    RELEASE LOCK
```

### Middleware Integration (axum)

```text
ASYNC FUNCTION RATE_LIMIT_MIDDLEWARE(limiter: RateLimiter, addr: SocketAddr, request: Request, next: Handler) → Response
    result ← AWAIT CHECK(limiter, addr.IP())

    IF NOT result.allowed THEN
        RETURN Response {
            status ← 429,
            headers ← { "X-RateLimit-Remaining": "0" }
        }

    response ← AWAIT next.RUN(request)
    response.headers["X-RateLimit-Remaining"] ← TO_STRING(result.remaining)
    RETURN response
```

### Actor-Based Alternative

For higher concurrency, use an actor to avoid lock contention:

```text
ENUM RateLimitCommand
    Check { ip: IpAddr, reply: Channel<RateLimitResult> }
    Cleanup   // Periodic cleanup of expired entries

ASYNC FUNCTION RATE_LIMIT_ACTOR(rx: Receiver<RateLimitCommand>, config: RateLimiterConfig)
    windows ← EMPTY Map<IpAddr, SlidingWindow>

    WHILE cmd ← AWAIT rx.RECEIVE()
        MATCH cmd
            CASE Check { ip, reply }:
                window ← windows.GET_OR_INSERT(ip, NEW SlidingWindow { timestamps ← [] })
                current ← COUNT_IN_WINDOW(window, config.window)
                allowed ← current < config.max_requests
                IF allowed THEN RECORD(window)
                SEND RateLimitResult {
                    allowed ← allowed,
                    remaining ← IF allowed THEN config.max_requests - current - 1 ELSE 0,
                    reset_at ← CURRENT_INSTANT() + config.window
                } TO reply
            CASE Cleanup:
                REMOVE entries FROM windows WHERE timestamps IS EMPTY
```

---

## Activity Feed System

### Requirements

- Users follow other users.
- When a user posts, followers see it in their feed.
- Feed is sorted by time, paginated.
- Must handle users with millions of followers.

### Strategy: Hybrid Fan-Out

- **Fan-out on write** for users with fewer than N followers: pre-compute each follower's feed at write time.
- **Fan-out on read** for celebrity users (millions of followers): merge their posts into the feed at read time.

### Schema

```sql
-- Fan-out on write: pre-computed feed entries
CREATE TABLE feed_items (
    user_id     BIGINT NOT NULL,
    activity_id BIGINT NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (user_id, activity_id)
);
CREATE INDEX idx_feed_user_time ON feed_items (user_id, created_at DESC);

-- Activities (posts)
CREATE TABLE activities (
    id          BIGSERIAL PRIMARY KEY,
    author_id   BIGINT NOT NULL,
    content     TEXT NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Follow relationships
CREATE TABLE follows (
    follower_id BIGINT NOT NULL,
    followee_id BIGINT NOT NULL,
    PRIMARY KEY (follower_id, followee_id)
);
```

### Domain Types

```text
RECORD Activity
    id: ActivityId
    author_id: UserId
    content: String
    created_at: DateTime

RECORD FeedItem
    activity: Activity
    author_name: String

RECORD FeedPage
    items: List<FeedItem>
    next_cursor: Optional<DateTime>
```

### Feed Service

```text
CONSTANT CELEBRITY_THRESHOLD ← 10000

RECORD FeedService<R: FeedRepository>
    repo: R

// Called when a user creates a new post
ASYNC FUNCTION PUBLISH(svc: FeedService, activity: Activity) → Result<Void, FeedError>
    AWAIT svc.repo.SAVE_ACTIVITY(activity)?

    follower_count ← AWAIT svc.repo.FOLLOWER_COUNT(activity.author_id)?

    IF follower_count < CELEBRITY_THRESHOLD THEN
        // Fan-out on write: push to all followers' feeds
        follower_ids ← AWAIT svc.repo.GET_FOLLOWERS(activity.author_id)?
        AWAIT svc.repo.FAN_OUT_TO_FEEDS(follower_ids, activity.id, activity.created_at)?
    // Celebrity posts are merged at read time -- no fan-out

    RETURN OK

// Called when a user views their feed
ASYNC FUNCTION GET_FEED(svc: FeedService, user_id: UserId, cursor: Optional<DateTime>, limit: Integer) → Result<FeedPage, FeedError>
    // Get pre-computed feed items
    items ← AWAIT svc.repo.GET_FEED_ITEMS(user_id, cursor, limit)?

    // Merge in celebrity posts (fan-out on read)
    celebrity_followees ← AWAIT svc.repo.GET_CELEBRITY_FOLLOWEES(user_id, CELEBRITY_THRESHOLD)?

    FOR EACH celeb_id IN celebrity_followees
        celeb_posts ← AWAIT svc.repo.GET_RECENT_ACTIVITIES(celeb_id, cursor, limit)?
        APPEND celeb_posts TO items

    // Sort merged results by time, take top `limit`
    SORT items BY created_at DESCENDING
    TRUNCATE items TO limit

    next_cursor ← LAST(items).created_at IF items IS NOT EMPTY ELSE NONE
    RETURN FeedPage { items ← items, next_cursor ← next_cursor }
```

### Repository Trait

```text
INTERFACE FeedRepository
    ASYNC FUNCTION SAVE_ACTIVITY(activity: Activity) → Result<Void, RepositoryError>
    ASYNC FUNCTION FAN_OUT_TO_FEEDS(follower_ids: List<UserId>, activity_id: ActivityId, created_at: DateTime) → Result<Void, RepositoryError>
    ASYNC FUNCTION GET_FEED_ITEMS(user_id: UserId, cursor: Optional<DateTime>, limit: Integer) → Result<List<FeedItem>, RepositoryError>
    ASYNC FUNCTION GET_FOLLOWERS(user_id: UserId) → Result<List<UserId>, RepositoryError>
    ASYNC FUNCTION FOLLOWER_COUNT(user_id: UserId) → Result<Integer, RepositoryError>
    ASYNC FUNCTION GET_CELEBRITY_FOLLOWEES(user_id: UserId, threshold: Integer) → Result<List<UserId>, RepositoryError>
    ASYNC FUNCTION GET_RECENT_ACTIVITIES(author_id: UserId, cursor: Optional<DateTime>, limit: Integer) → Result<List<FeedItem>, RepositoryError>
```

### Design Decisions

- **Hybrid fan-out** avoids the write amplification problem of pure fan-out-on-write (a celebrity posting would create millions of feed entries) while keeping reads fast for normal users.
- **Cursor-based pagination** (using `created_at`) instead of offset-based, so inserting new items does not shift pages.
- **Repository trait** makes it possible to test the feed merging logic with an in-memory implementation, without touching a database.
