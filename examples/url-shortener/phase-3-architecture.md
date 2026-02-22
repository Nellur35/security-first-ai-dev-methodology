# Phase 3 -- Architecture

## Component Diagram

```
                    +-----------+
   POST /shorten -> |           | -> Redis (read cache)
   GET  /{code}  -> |  API      |    |
   GET  /stats/* -> |  Service  | -> PostgreSQL (persistent store)
   DELETE /{code}-> |  (FastAPI)|    |
                    +-----+-----+
                          |
                    +-----v-------+
                    |  Analytics   | -> PostgreSQL (analytics table)
                    |  Collector   |
                    +--------------+
```

## Components

### API Service (FastAPI)
Single entry point. Handles URL creation, redirect, deletion, and stats retrieval. Stateless -- all state lives in the data stores.

- **URL Shortener:** Validates input, generates hash-based short code, writes to PostgreSQL, populates Redis cache
- **Redirector:** Looks up short code in Redis (fallback: PostgreSQL), returns 301. Fires analytics event
- **Stats Handler:** Queries analytics table, returns JSON

### Analytics Collector
Async worker that processes redirect events. Decoupled from the redirect hot path so analytics writes don't add latency to redirects. Receives events via an in-process queue (v1) -- swap for Redis pub/sub or SQS later if needed.

### Data Stores
- **PostgreSQL:** Source of truth. `urls` table (short_code, original_url, created_at) and `clicks` table (short_code, timestamp, referrer, ip_hash)
- **Redis:** Read-through cache for short_code -> original_url lookups. TTL of 24h. Cache miss falls through to PostgreSQL

## Interfaces

| From | To | Interface | Contract |
|---|---|---|---|
| API Service | PostgreSQL | SQLAlchemy models | `Url` model, `Click` model |
| API Service | Redis | `RedisClient` wrapper | `get(code) -> url`, `set(code, url)` |
| API Service | Analytics Collector | `AnalyticsEmitter` interface | `emit(event: ClickEvent)` |

## Dependency Injection

All external dependencies are injected via FastAPI's `Depends()`:
- `get_db() -> Session` -- swappable with SQLite for tests
- `get_cache() -> RedisClient` -- swappable with `FakeRedis` or dict-based stub
- `get_analytics() -> AnalyticsEmitter` -- swappable with in-memory collector for tests

Each component is testable with zero real infrastructure. Integration tests use Docker containers via `testcontainers-python`.

## Decisions & Rejected Alternatives

| Decision | Rejected alternative | Why |
|---|---|---|
| Monolith (single FastAPI app) | Microservices (separate redirect service) | Two services for a URL shortener is over-engineering. The redirect path is a single cache lookup -- there's nothing to gain from a separate process. Split later if analytics load requires it |
| In-process async queue for analytics | Kafka / SQS | v1 handles maybe 1000 req/s. An in-process asyncio queue is plenty. Adding Kafka means another service to deploy, monitor, and secure. Defined behind an interface so we can swap it later |
| FastAPI | Flask / Express | FastAPI gives async out of the box, auto-generated OpenAPI docs, and Pydantic validation. Redirect latency benefits from async. Flask would need Celery for async; Express would work but the team knows Python |
| SQLAlchemy + Alembic | Raw SQL | Migrations matter from day one. Raw SQL migrations are fragile. SQLAlchemy adds ~2ms overhead per query which is fine for the write path (not used on the redirect hot path -- that's Redis) |
