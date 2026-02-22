# Phase 4 -- Threat Model

## Trust Boundaries

1. **Internet -> API Service:** Untrusted input from any source. This is the main attack surface.
2. **API Service -> Redis:** Internal network. Redis has no auth in default config.
3. **API Service -> PostgreSQL:** Internal network, authenticated connection.
4. **API Service -> Analytics Collector:** In-process, same trust domain.

## Threat Analysis

| # | Threat | Impact | Likelihood | Mitigation |
|---|--------|--------|------------|------------|
| T1 | **Open redirect abuse.** Attacker creates short links to phishing sites, uses our domain's reputation to bypass email filters | High | High | Validate destination URLs against a blocklist of known phishing domains. Check Google Safe Browsing API on creation. Log all created URLs for retroactive scanning |
| T2 | **URL enumeration.** Attacker iterates short codes to discover all stored URLs (some may be sensitive/unlisted) | Medium | Medium | Hash-based codes aren't sequential, so blind enumeration is ~56B keyspace. Add rate limiting on 404s (10/min per IP). Monitor for scanning patterns |
| T3 | **Denial of service via mass URL creation.** Attacker floods POST /shorten to exhaust database storage | Medium | High | Rate limit: 100 creates/min per IP. Idempotent creation means duplicate URLs don't consume storage. Set a hard cap on total URLs if needed |
| T4 | **SSRF via URL validation.** If the server fetches the destination URL to validate it (e.g., check if it's live), attacker submits `http://169.254.169.254/...` to hit cloud metadata | High | Medium | Don't fetch destination URLs server-side. Validation is syntactic only (valid URL format, HTTP(S) scheme). Never make outbound requests to user-supplied URLs |
| T5 | **Redis poisoning.** If Redis is accessible on the internal network without auth, an attacker with network access can overwrite cache entries to redirect users anywhere | Critical | Low | Enable Redis AUTH. Bind to localhost or private subnet. Use TLS for Redis connections. Even if Redis is poisoned, redirect through PostgreSQL on cache miss as a fallback verification |
| T6 | **Analytics data leakage.** Click data (IP addresses, referrers) is PII in many jurisdictions | Medium | Medium | Hash IP addresses before storage (SHA-256 + salt). Don't store raw referrer URLs -- extract domain only. Add a data retention policy (delete clicks older than 90 days) |
| T7 | **Stored XSS via URL.** Malicious URL containing JavaScript is rendered in analytics dashboard or API responses | Medium | Low | URLs are never rendered as HTML. API returns JSON only. If a dashboard is added later, output-encode all URL values. Content-Type headers must be `application/json` |

## IAM & Infrastructure

- **Database credentials:** Stored in environment variables (v1). Move to a secrets manager before production. DB user has access only to the `shortener` schema, not the whole server.
- **Redis:** No sensitive data beyond the URL mappings. If compromised, attacker can redirect users (see T5) but can't access other systems.
- **Deployment role (if cloud):** Needs access to the database and Redis only. No S3, no SQS, no other services. Scope the IAM role to exactly these two resources.

## Error Handling

- 404 on unknown short codes returns `{"error": "not found"}` -- no information about whether the code was ever valid
- Database connection errors return 503, not stack traces
- Rate limit responses return 429 with `Retry-After` header -- don't reveal the exact limit threshold in the response body
- Validation errors return 422 with field-level detail (this is fine -- it's the user's own input)

## Supply Chain

- Pin all Python dependencies with hashes in `requirements.txt` (use `pip-compile --generate-hashes`)
- Run `pip-audit` in CI to check for known vulnerabilities
- FastAPI, SQLAlchemy, and redis-py are mature, widely-used packages -- low supply chain risk
- `gitleaks` in CI to catch accidentally committed secrets
- Container base image: use `python:3.12-slim`, pin the digest, rebuild weekly
