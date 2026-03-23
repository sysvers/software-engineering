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

```rust
pub struct ShortUrl {
    pub code: ShortCode,
    pub original_url: Url,
    pub created_at: DateTime<Utc>,
    pub expires_at: Option<DateTime<Utc>>,
    pub click_count: u64,
}

pub struct ShortCode(String);  // Newtype for type safety

impl ShortCode {
    pub fn generate() -> Self {
        // Base62 encoding of random bytes → 7-char code
        // 62^7 = 3.5 trillion possibilities
        Self(nanoid::nanoid!(7, &ALPHABET))
    }
}

impl ShortUrl {
    pub fn is_expired(&self) -> bool {
        self.expires_at
            .map(|exp| Utc::now() > exp)
            .unwrap_or(false)
    }
}
```

### Repository Trait

```rust
trait UrlRepository {
    async fn save(&self, url: &ShortUrl) -> Result<(), RepositoryError>;
    async fn find_by_code(&self, code: &ShortCode) -> Result<Option<ShortUrl>, RepositoryError>;
    async fn increment_clicks(&self, code: &ShortCode) -> Result<(), RepositoryError>;
}
```

### Application Layer

```rust
pub struct UrlService<R: UrlRepository> {
    repo: R,
}

impl<R: UrlRepository> UrlService<R> {
    pub async fn shorten(&self, original: &str) -> Result<ShortUrl, UrlError> {
        let url = Url::parse(original).map_err(|_| UrlError::InvalidUrl)?;
        let code = ShortCode::generate();
        let short_url = ShortUrl::new(code, url);
        self.repo.save(&short_url).await?;
        Ok(short_url)
    }

    pub async fn resolve(&self, code: &str) -> Result<Url, UrlError> {
        let code = ShortCode::from(code);
        let short_url = self.repo.find_by_code(&code).await?
            .ok_or(UrlError::NotFound)?;

        if short_url.is_expired() {
            return Err(UrlError::Expired);
        }

        self.repo.increment_clicks(&code).await?;
        Ok(short_url.original_url)
    }
}
```

### API Layer

```rust
async fn create_short_url(
    State(svc): State<Arc<UrlService<impl UrlRepository>>>,
    Json(req): Json<ShortenRequest>,
) -> Result<Json<ShortenResponse>, AppError> {
    let short_url = svc.shorten(&req.url).await?;
    Ok(Json(ShortenResponse {
        short_url: format!("https://sho.rt/{}", short_url.code.as_str()),
        expires_at: short_url.expires_at,
    }))
}

async fn redirect(
    State(svc): State<Arc<UrlService<impl UrlRepository>>>,
    Path(code): Path<String>,
) -> Result<Redirect, AppError> {
    let original = svc.resolve(&code).await?;
    Ok(Redirect::temporary(original.as_str()))
}
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

```rust
use std::collections::HashMap;
use std::net::IpAddr;
use std::time::{Duration, Instant};
use tokio::sync::RwLock;
use std::sync::Arc;

pub struct RateLimiterConfig {
    pub max_requests: usize,
    pub window: Duration,
}

pub struct RateLimiter {
    config: RateLimiterConfig,
    windows: Arc<RwLock<HashMap<IpAddr, SlidingWindow>>>,
}

struct SlidingWindow {
    timestamps: Vec<Instant>,
}

impl SlidingWindow {
    fn count_in_window(&mut self, window: Duration) -> usize {
        let cutoff = Instant::now() - window;
        self.timestamps.retain(|&t| t > cutoff);
        self.timestamps.len()
    }

    fn record(&mut self) {
        self.timestamps.push(Instant::now());
    }
}
```

### Rate Limiter Implementation

```rust
pub struct RateLimitResult {
    pub allowed: bool,
    pub remaining: usize,
    pub reset_at: Instant,
}

impl RateLimiter {
    pub async fn check(&self, ip: IpAddr) -> RateLimitResult {
        let mut windows = self.windows.write().await;
        let window = windows.entry(ip).or_insert(SlidingWindow {
            timestamps: Vec::new(),
        });

        let current = window.count_in_window(self.config.window);

        if current < self.config.max_requests {
            window.record();
            RateLimitResult {
                allowed: true,
                remaining: self.config.max_requests - current - 1,
                reset_at: Instant::now() + self.config.window,
            }
        } else {
            RateLimitResult {
                allowed: false,
                remaining: 0,
                reset_at: Instant::now() + self.config.window,
            }
        }
    }
}
```

### Middleware Integration (axum)

```rust
async fn rate_limit_middleware(
    State(limiter): State<Arc<RateLimiter>>,
    ConnectInfo(addr): ConnectInfo<SocketAddr>,
    request: Request,
    next: Next,
) -> Response {
    let result = limiter.check(addr.ip()).await;

    if !result.allowed {
        return (
            StatusCode::TOO_MANY_REQUESTS,
            [("X-RateLimit-Remaining", "0")],
        ).into_response();
    }

    let mut response = next.run(request).await;
    response.headers_mut().insert(
        "X-RateLimit-Remaining",
        result.remaining.to_string().parse().unwrap(),
    );
    response
}
```

### Actor-Based Alternative

For higher concurrency, use an actor to avoid lock contention:

```rust
enum RateLimitCommand {
    Check {
        ip: IpAddr,
        reply: oneshot::Sender<RateLimitResult>,
    },
    Cleanup,  // Periodic cleanup of expired entries
}

async fn rate_limit_actor(mut rx: mpsc::Receiver<RateLimitCommand>, config: RateLimiterConfig) {
    let mut windows: HashMap<IpAddr, SlidingWindow> = HashMap::new();

    while let Some(cmd) = rx.recv().await {
        match cmd {
            RateLimitCommand::Check { ip, reply } => {
                let window = windows.entry(ip).or_insert(SlidingWindow {
                    timestamps: Vec::new(),
                });
                let current = window.count_in_window(config.window);
                let allowed = current < config.max_requests;
                if allowed { window.record(); }
                let _ = reply.send(RateLimitResult {
                    allowed,
                    remaining: if allowed { config.max_requests - current - 1 } else { 0 },
                    reset_at: Instant::now() + config.window,
                });
            }
            RateLimitCommand::Cleanup => {
                windows.retain(|_, w| !w.timestamps.is_empty());
            }
        }
    }
}
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

```rust
pub struct Activity {
    pub id: ActivityId,
    pub author_id: UserId,
    pub content: String,
    pub created_at: DateTime<Utc>,
}

pub struct FeedItem {
    pub activity: Activity,
    pub author_name: String,
}

pub struct FeedPage {
    pub items: Vec<FeedItem>,
    pub next_cursor: Option<DateTime<Utc>>,
}
```

### Feed Service

```rust
const CELEBRITY_THRESHOLD: usize = 10_000;

pub struct FeedService<R: FeedRepository> {
    repo: R,
}

impl<R: FeedRepository> FeedService<R> {
    /// Called when a user creates a new post
    pub async fn publish(&self, activity: &Activity) -> Result<(), FeedError> {
        self.repo.save_activity(activity).await?;

        let follower_count = self.repo.follower_count(activity.author_id).await?;

        if follower_count < CELEBRITY_THRESHOLD {
            // Fan-out on write: push to all followers' feeds
            let follower_ids = self.repo.get_followers(activity.author_id).await?;
            self.repo.fan_out_to_feeds(&follower_ids, activity.id, activity.created_at).await?;
        }
        // Celebrity posts are merged at read time — no fan-out

        Ok(())
    }

    /// Called when a user views their feed
    pub async fn get_feed(
        &self,
        user_id: UserId,
        cursor: Option<DateTime<Utc>>,
        limit: usize,
    ) -> Result<FeedPage, FeedError> {
        // Get pre-computed feed items
        let mut items = self.repo.get_feed_items(user_id, cursor, limit).await?;

        // Merge in celebrity posts (fan-out on read)
        let celebrity_followees = self.repo
            .get_celebrity_followees(user_id, CELEBRITY_THRESHOLD).await?;

        for celeb_id in celebrity_followees {
            let celeb_posts = self.repo
                .get_recent_activities(celeb_id, cursor, limit).await?;
            items.extend(celeb_posts);
        }

        // Sort merged results by time, take top `limit`
        items.sort_by(|a, b| b.created_at.cmp(&a.created_at));
        items.truncate(limit);

        let next_cursor = items.last().map(|item| item.created_at);
        Ok(FeedPage { items, next_cursor })
    }
}
```

### Repository Trait

```rust
trait FeedRepository {
    async fn save_activity(&self, activity: &Activity) -> Result<(), RepositoryError>;
    async fn fan_out_to_feeds(
        &self, follower_ids: &[UserId], activity_id: ActivityId, created_at: DateTime<Utc>,
    ) -> Result<(), RepositoryError>;
    async fn get_feed_items(
        &self, user_id: UserId, cursor: Option<DateTime<Utc>>, limit: usize,
    ) -> Result<Vec<FeedItem>, RepositoryError>;
    async fn get_followers(&self, user_id: UserId) -> Result<Vec<UserId>, RepositoryError>;
    async fn follower_count(&self, user_id: UserId) -> Result<usize, RepositoryError>;
    async fn get_celebrity_followees(
        &self, user_id: UserId, threshold: usize,
    ) -> Result<Vec<UserId>, RepositoryError>;
    async fn get_recent_activities(
        &self, author_id: UserId, cursor: Option<DateTime<Utc>>, limit: usize,
    ) -> Result<Vec<FeedItem>, RepositoryError>;
}
```

### Design Decisions

- **Hybrid fan-out** avoids the write amplification problem of pure fan-out-on-write (a celebrity posting would create millions of feed entries) while keeping reads fast for normal users.
- **Cursor-based pagination** (using `created_at`) instead of offset-based, so inserting new items does not shift pages.
- **Repository trait** makes it possible to test the feed merging logic with an in-memory implementation, without touching a database.
