# SPIRL Beta Requirements v0.1

**Status:** Baseline for implementation  
**Date:** 2026-10-20  
**Target:** 2026-12-14 beta

## 1. Product statement

SPIRL is a downloadable, self-hosted Rust reverse proxy for Linux. It protects a single HTTP application using HTTPS termination, configurable HTTP request rules, and bounded in-memory rate limiting. It must run without AI or a required external service.

The beta is a **functional security-tool beta**, not a claim of production readiness, comprehensive WAF coverage, or protection from volumetric DDoS attacks.

## 2. Intended deployment

- One SPIRL process on one Linux machine.
- One configured upstream HTTP application.
- Public client traffic terminates at SPIRL; the origin must not be independently exposed.
- Public HTTPS uses an administrator-provided certificate and key.
- Initial released binary target: `x86_64-unknown-linux-gnu`; other targets are later work.

## 3. Must-have functional requirements (P0)

| ID | Requirement | Acceptance evidence |
| --- | --- | --- |
| FR-01 | Proxy HTTP/1.1 requests to one configured upstream | GET, POST, paths, headers, and request bodies reach test origin; responses are returned correctly |
| FR-02 | Stream traffic without buffering entire bodies in application memory | Large upload and download tests succeed within configured limits |
| FR-03 | Accept HTTPS using supplied certificate and key | `curl` validates a test HTTPS endpoint with an appropriate trust root |
| FR-04 | Support one HTTP or verified HTTPS upstream | Successful tests for localhost HTTP and TLS-verified origin connection |
| FR-05 | Derive client identity from TCP peer by default | Spoofed `X-Forwarded-For` does not alter the effective IP or its bucket |
| FR-06 | Load a TOML configuration; fail on invalid configuration | Clear startup validation errors; no listener starts with invalid policies |
| FR-07 | Support expression-based match rules | Tests cover fields, operators, parentheses, and type validation |
| FR-08 | Support `block` and non-terminal `log` actions | Matching block returns 403; log-only rules still forward |
| FR-09 | Support IP-scoped token-bucket rate limits | A burst is allowed up to capacity; excess requests return 429 |
| FR-10 | Apply all matching rate limits after checking block rules | A request exceeding any matching limit is rejected; precedence is deterministic |
| FR-11 | Bound rate-limit memory with idle eviction | State never grows beyond configured entry limit under high-cardinality tests |
| FR-12 | Atomically reload policy and limit rules on SIGHUP | Valid reload takes effect; invalid reload leaves existing rules active |
| FR-13 | Expose configuration validation command | `spirl check --config <path>` returns nonzero for invalid configuration |
| FR-14 | Produce structured operational logs and basic counters | Tests demonstrate forwarded, blocked, limited, and upstream-error events |
| FR-15 | Package the Linux executable with setup instructions | Clean Linux installation guide and systemd example work in a VM |
| FR-16 | Handle failures with appropriate HTTP errors and timeouts | Upstream disconnect and timeout produce bounded, non-sensitive responses |

## 4. Required v0.1 rule language

### Supported fields

- `ip.src`
- `http.request.method`
- `http.request.uri.path`
- `http.request.headers["header-name"]`

Header names are case-insensitive. The first matching header value is used for `eq`, `ne`, and `contains`; document repeated-header limitations. Path evaluation excludes the query string and does not automatically decode percent escapes.

### Operators

- Comparisons: `eq`, `ne`, `contains` for compatible string fields.
- Address membership: `in` on `ip.src`, with an IP or CIDR set literal.
- Boolean logic: `and`, `or`, `not`, parentheses.
- Expressions cannot access request bodies or execute arbitrary functions.

### Actions

| Action | Behavior |
| --- | --- |
| `block` | Return HTTP 403 when matched |
| `rate_limit` | Apply a token bucket with a documented refill and burst capacity; return HTTP 429 when exceeded |
| `log` | Record a rule match but continue processing |

Any matching block overrides rate limiting. Otherwise, every matching rate-limit rule is applied. Multiple matching limits are independent and are not charged transactionally. No bypassing `allow` action is part of the beta.

### Examples

```text
http.request.uri.path eq "/login" and http.request.method eq "POST"
ip.src in {203.0.113.0/24 198.51.100.17}
not (http.request.headers["user-agent"] contains "KnownClient")
```

These are **SPIRL design examples**, not a commitment to accept every Cloudflare expression.

## 5. Rate-limiting semantics

For a rule with `requests = 10`, `period = "60s"`, and `burst = 10`, a new per-IP bucket begins with capacity 10 and refills at 10 tokens per 60 seconds. Requests consume one available token each. It can initially allow 10 rapid requests, but it does **not** enforce a strict count of 10 over every rolling minute.

The effective key is `(rule_id, trusted_client_ip)`. Use monotonic elapsed time for refill. Concurrency must not permit material over-admission due to races. On the configured state-capacity limit, reject requests requiring new bucket entries with 429 and record the cause; never silently disable the limiter. State resets when the process restarts.

When an unchanged rule is reloaded, its limiter state should remain valid. A changed or removed rule must not accidentally inherit incompatible state.

## 6. Non-functional and security requirements (P0)

| ID | Requirement | Verification |
| --- | --- | --- |
| NFR-01 | Rust memory-safe application code; avoid `unsafe` in SPIRL-owned modules unless justified | Source review and linting |
| NFR-02 | No mandatory Redis, SQL database, AI service, or cloud account | Startup and end-to-end tests on an offline local VM |
| NFR-03 | Apply finite request, connection, and upstream timeouts | Slow-client and slow-origin tests terminate predictably |
| NFR-04 | Bound memory for request parsing, concurrency, and limiter entries | Stress tests and configured ceilings |
| NFR-05 | Do not trust client-supplied forwarding headers by default | Forged-header integration tests |
| NFR-06 | Reject invalid security expressions; preserve active policy on invalid reload | Parser, startup, and reload tests |
| NFR-07 | Keep private keys outside logs and source control; document restricted permissions | Install guide review |
| NFR-08 | Verify upstream TLS certificates when HTTPS upstream is configured | Valid-cert and invalid-cert integration tests |
| NFR-09 | Do not log credentials, body data, cookies, or query strings by default | Structured log capture tests |
| NFR-10 | CI runs formatting, lints, build, and automated tests | GitHub Actions green on default branch |
| NFR-11 | Document operational limitations and support boundaries | README and release-notes review |

### Performance targets and measurement

The requirement is **repeatable measurement**, not an unsupported promise to outperform Nginx or Cloudflare. Capture requests per second, median and p95 proxy overhead, CPU utilization, and resident memory under an identical local workload, compared with direct-to-origin requests. Publish the test machine, traffic generator, connection count, payload size, rule set, and procedure. Set numeric optimization targets after the first working proxy baseline.

## 7. Configuration usability

- An example TOML file must be checked into `examples/`.
- Error messages must identify the invalid key or rule and, where feasible, line and column.
- Startup must fail safely on unreadable, malformed, or semantically invalid configuration.
- Runtime policy reload must either fully succeed or leave the prior snapshot active; partial update is forbidden.
- Only policies and rate-limiting configuration are guaranteed hot-reloadable. Listener, TLS certificate, and upstream configuration require restart for beta.
- Document default timeouts, memory caps, logging behavior, and limits in the example file.

## 8. Required test scenarios

1. Proxied GET and POST preserve expected path, method, request body, and response status.
2. HTTPS connects with valid certificate; invalid certificate and key paths fail safely.
3. Match and block `/admin` with HTTP 403 and verify the origin was not contacted.
4. Match a login POST limit: a fresh bucket permits its configured burst, then returns HTTP 429.
5. Two different client identities use independent buckets, with identity tests performed through a controlled trusted-proxy fixture rather than spoofed headers.
6. An untrusted forwarded-IP header has no effect.
7. An invalid expression fails `spirl check` and startup, and a bad reload preserves existing rules.
8. `log` plus `block` precedence behaves as documented; multiple limits are exercised independently.
9. High-cardinality IP attempts never exceed the configured state-capacity limit.
10. A down, slow, or TLS-invalid upstream returns a bounded failure without leaking internal details.
11. A valid SIGHUP reload changes the policy without interrupting an already active request.
12. A fresh Linux VM can install and start the released binary using documented instructions.

## 9. Excluded from the beta (P1 or later)

WebSockets, multiple upstreams and load balancing, HTTP/2 guarantees, HTTP/3, distributed counters, Redis, geolocation rules, regex matching, challenge pages, behavioral bot detection, AI, admin web UI, public remote management API, automatic certificate issuance, and comprehensive WAF features. Do not promise network-layer DDoS protection.

## 10. Beta definition of done

The beta can be tagged only when every P0 requirement passes its acceptance check; representative functional tests run automatically in CI; Linux installation is reproduced on a clean machine; documentation clearly identifies security limitations; and open defects are classified and documented. Performance results must be published even if no speed claim is made.

## 11. Change management

Any feature added to beta scope must identify what it displaces from the roadmap. Keep the scope small and review architectural changes against `ARCHITECTURE.md` before implementation.
