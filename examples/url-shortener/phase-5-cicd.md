# Phase 5 -- CI/CD Pipeline

## Pipeline Stages

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: shortener_test
          POSTGRES_PASSWORD: test
        ports: ["5432:5432"]
      redis:
        image: redis:7
        ports: ["6379:6379"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements.txt -r requirements-dev.txt

      # Quality gates
      - run: ruff check .                     # Linting
      - run: ruff format --check .            # Formatting
      - run: pytest --cov=app --cov-fail-under=85 tests/

      # Security gates
      - run: bandit -r app/                   # SAST
      - run: pip-audit                        # SCA / dependency vulnerabilities
      - run: gitleaks detect --source .       # Secret scanning
```

## Gate-to-Threat Mapping

| Gate | Tool | Catches threat | Doesn't catch |
|---|---|---|---|
| SAST | `bandit` | T7 (XSS patterns), hardcoded secrets, insecure function calls | Logic bugs, business logic flaws |
| SCA | `pip-audit` | Known CVEs in dependencies (supply chain) | Zero-days, malicious packages not yet flagged |
| Secret scanning | `gitleaks` | Committed API keys, database passwords, tokens | Secrets in env vars or config management |
| Unit tests | `pytest` | T4 (SSRF -- test that no outbound requests are made), T2 (rate limiting on 404s), input validation | Real network issues, race conditions |
| Integration tests | `pytest` + real Redis/Postgres | T5 (Redis auth -- test that unauthenticated Redis connection fails), actual redirect latency | Production-scale load |
| Linting | `ruff` | Code quality, import hygiene, basic type issues | Security vulnerabilities |

## Custom Security Gates (from threat model)

These go beyond the standard tooling:

```python
# tests/security/test_no_outbound_requests.py
# Maps to T4: SSRF prevention
def test_shorten_does_not_fetch_destination(monkeypatch):
    """Ensure URL creation never makes outbound HTTP requests."""
    import socket
    def deny_all(*args, **kwargs):
        raise RuntimeError("Outbound connection attempted during URL creation")
    monkeypatch.setattr(socket, "create_connection", deny_all)
    response = client.post("/shorten", json={"url": "https://example.com/long"})
    assert response.status_code == 201

# tests/security/test_error_no_leakage.py
# Maps to T2, error handling review
def test_404_reveals_nothing():
    response = client.get("/nonexistent-code")
    body = response.json()
    assert response.status_code == 404
    assert body == {"error": "not found"}
    assert "traceback" not in response.text.lower()
```

## Dummy Product

Minimal FastAPI app that exercises every gate:

- `app/main.py` -- FastAPI app with `/shorten`, `/{code}`, `/stats/{code}`, `/health`
- `app/models.py` -- SQLAlchemy `Url` and `Click` models
- `app/cache.py` -- `RedisClient` wrapper with `get`/`set`
- `app/analytics.py` -- `AnalyticsEmitter` interface + in-memory implementation
- `tests/` -- Unit tests for each component with injected fakes, plus the security tests above

The dummy product is intentionally simple. It returns hardcoded responses where real logic would go, but it proves the pipeline catches real problems: missing tests, vulnerable dependencies, leaked secrets, and SSRF patterns.

## What the Pipeline Proves vs. Doesn't Catch

**Proves:**
- No known vulnerable dependencies ship
- No secrets in the repo
- No obvious SAST findings (insecure hashing, SQL injection patterns)
- Redirect path doesn't make outbound requests (SSRF gate)
- Error responses don't leak internals
- 85%+ code coverage (as a side effect of testing behavior, not a target)

**Doesn't catch:**
- Open redirect abuse (T1) -- requires runtime blocklist checking, not a static gate
- Redis misconfiguration in production (T5) -- integration tests use a test Redis, not the production instance
- Performance under real load (NFR-1) -- need a separate load test stage (add k6 or locust later)
- Data retention compliance (T6) -- policy enforcement, not a CI gate
