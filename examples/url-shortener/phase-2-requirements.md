# Phase 2 -- Requirements

## Functional Requirements

- **FR-1:** Accept a URL via POST, return a short code (e.g., `https://sho.rt/ab12Xz`)
- **FR-2:** GET on a short code returns 301 redirect to the original URL
- **FR-3:** GET on `/stats/{code}` returns click count, referrers, and last-accessed timestamp
- **FR-4:** Short codes are 6-character base62 (a-z, A-Z, 0-9) -- gives ~56 billion combinations
- **FR-5:** Submitting the same URL twice returns the same short code (idempotent)
- **FR-6:** Short codes don't expire unless explicitly deleted via DELETE endpoint

## Non-Functional Requirements

- **NFR-1:** Redirect latency under 100ms at p99 (this is the hot path -- everything else can be slower)
- **NFR-2:** 99.9% uptime for the redirect path (analytics can tolerate brief outages)
- **NFR-3:** Rate limiting: 100 creates/min per IP, no limit on redirects
- **NFR-4:** Input validation: reject non-HTTP(S) URLs, reject URLs longer than 2048 chars
- **NFR-5:** All API responses include appropriate CORS headers

## Explicit Exclusions (v1)

- No user accounts or authentication (all links are public)
- No link editing after creation
- No custom short codes (auto-generated only)
- No QR code generation
- No link expiration/TTL
- No bulk import

## Definition of Done

A deployed API where: you can POST a URL, GET the short code and be redirected, and GET stats that show at least a click count. Redirect works under 100ms with 1000 concurrent connections. Rate limiting actually blocks the 101st request.

## Decisions & Rejected Alternatives

| Decision | Rejected alternative | Why |
|---|---|---|
| Hash-based short codes (SHA-256, take first 6 chars of base62 encoding) | Sequential integer IDs | Sequential IDs are enumerable -- an attacker can scrape every URL in the system by incrementing. Hash-based codes are practically random |
| Redis for redirect lookup | DynamoDB | Redirect is the hot path. Redis gives single-digit ms reads from memory. DynamoDB would add 5-10ms per read and we'd pay per request. PostgreSQL stays as the source of truth; Redis is the read cache |
| PostgreSQL as primary store | Redis-only | Redis is fast but not durable by default. Losing all URLs on a restart isn't acceptable. PostgreSQL is the persistent store; Redis is populated from it |
| 301 (permanent) redirects | 302 (temporary) redirects | 301s let browsers cache the redirect, reducing load. Since we don't support link editing, the redirect is truly permanent. Tradeoff: analytics undercounts repeat visits from the same browser |
| Rate limit by IP | Rate limit by API key | No user accounts in v1, so there are no API keys. IP-based is imperfect (NAT, proxies) but good enough for v1 abuse prevention |
