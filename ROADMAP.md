# SPIRL Development Roadmap

**Starting point:** 2026-10-20  
**Target beta:** 2026-12-14 (approximately eight weeks)  
**Plan status:** Proposed delivery sequence; adjust based on prototype results

## Guiding principle

Ship a **small, tested, installable** reverse proxy rather than a large collection of unfinished security features. Finish a functional vertical slice early, validate risky components, and make later milestones additive.

## Eight-week milestones

| Week | Dates (2026) | Focus | Demonstrable exit criterion |
| --- | --- | --- | --- |
| 1 | Oct 20–26 | Foundations and feasibility | Architecture and requirements agreed; repository bootstrapped; Pingora listener and TLS APIs experimentally verified |
| 2 | Oct 27–Nov 2 | Minimum working proxy | GET and POST proxy to one local upstream; integration tests run locally and in CI |
| 3 | Nov 3–9 | Client identity and rate limiter | Concurrent, bounded per-IP token bucket with independent keys and reliable 429 behavior |
| 4 | Nov 10–16 | Expression language | Parser, typed AST, validator, and evaluator pass unit tests |
| 5 | Nov 17–23 | Full rule pipeline and reload | `block`, `log`, and `rate_limit` actions enforce precedence; invalid SIGHUP reload rolls back safely |
| 6 | Nov 24–30 | HTTPS and hardening | TLS termination, verified HTTPS upstream, error paths, timeouts, and spoofed-header tests |
| 7 | Dec 1–7 | Reliability and benchmarks | End-to-end suite, concurrency and memory stress tests, published baseline benchmarks |
| 8 | Dec 8–14 | Linux beta release | Reproducible release build, systemd service, installation instructions, versioned beta tag |

## Week 1 — Preparation and technical spike

**Tasks**

- Commit `docs/ARCHITECTURE.md`, `docs/REQUIREMENTS.md`, and `docs/ROADMAP.md`.
- Create or update README, LICENSE, `.gitignore`, and a basic issues board.
- Initialize one Rust Cargo binary package in the existing Git repository; commit `Cargo.lock`.
- Pin a Rust toolchain and initial dependencies. Verify current Pingora release compatibility instead of relying on unreviewed examples.
- Add GitHub Actions with `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, `cargo build`, and `cargo test`.
- Implement a disposable Pingora proof-of-concept: one HTTP listener, one upstream. Validate that TLS setup is feasible.
- Decide numeric bounds for HTTP headers, timeouts, connection concurrency, and rate-state entries before accepting public traffic.

**Done when:** the project builds in a fresh Linux environment, CI passes, and there is no unresolved framework blocker. If Pingora blocks core requirements, evaluate Hyper as a fallback now rather than mid-project.

## Week 2 — Reverse proxy vertical slice

**Tasks**

- Implement a single upstream target and basic HTTP/1.1 forwarding.
- Stream request and response bodies.
- Preserve expected methods, paths, statuses, and relevant end-to-end headers.
- Adopt the framework's safe hop-by-hop header handling.
- Handle offline upstream and timeout conditions.
- Add local test-origin fixtures and integration tests.

**Done when:** `curl` through SPIRL returns the test origin's content for GET and POST, and automated tests prove it.

## Week 3 — Identity and in-memory rate limiter

**Tasks**

- Implement trusted client-IP extraction; direct TCP peer is authoritative by default.
- Implement token bucket with monotonic timestamps.
- Key buckets by stable rule ID and client IP.
- Enforce bounded capacity, idle eviction, and safe high-cardinality behavior.
- Add concurrent tests and initial measurements.

**Done when:** a controlled burst behaves predictably, excess traffic gets 429, spoofed headers fail to alter the limit, and entry count remains bounded.

## Week 4 — Rules parser and evaluator

**Tasks**

- Implement a tokenizer and precedence-aware parser for the agreed expression subset.
- Build a typed AST and compile it during configuration validation.
- Add supported fields and IP or CIDR-set matching.
- Reject unknown fields, invalid operands, long expressions, and excessive nesting.
- Test precedence, nested expressions, header handling, IP parsing, and invalid input.

**Done when:** a rule can be compiled and evaluated against an HTTP request representation without networking dependencies.

## Week 5 — Policy integration and reload

**Tasks**

- Define immutable compiled policy snapshots.
- Integrate matching `log`, `block`, and `rate_limit` actions into the proxy filter stage.
- Enforce any-block-first behavior and all matching rate limits.
- Implement `spirl check --config <path>`.
- Implement SIGHUP validation and atomic policy replacement, retaining last known-good state on error.
- Test state handling when rate-limit rules are changed or removed.

**Done when:** the full rule pipeline works and policy reloads do not require terminating request processing.

## Week 6 — HTTPS and security hardening

**Tasks**

- Enable frontend HTTPS with user-provided certificate and private key.
- Support verified TLS to an HTTPS upstream, keeping a localhost HTTP upstream as a common deployment.
- Review connection timeouts, header limits, request limits, and resource ceilings.
- Test malformed HTTP, forged forwarding headers, large headers, unavailable origins, and invalid certificates.
- Ensure logs do not reveal credentials, cookies, or request bodies.

**Done when:** representative HTTPS deployments and negative security tests pass.

## Week 7 — Quality, reliability, and benchmarks

**Tasks**

- Build end-to-end integration tests covering all P0 requirements.
- Run high-cardinality state tests, concurrent traffic tests, and long-running stability tests.
- Compare direct-to-origin traffic against proxy-only, proxy-plus-rule, and proxy-plus-rate-limit workloads.
- Document test hardware, workloads, p50 and p95 latency, throughput, RSS, and CPU.
- Fix correctness bugs before performance tuning.

**Done when:** the test suite is green and the repository contains reproducible benchmarks with no invented speed claims.

## Week 8 — Packaging and release

**Tasks**

- Produce and checksum a release build for `x86_64-unknown-linux-gnu`.
- Provide a systemd service example, sample configuration, installation guide, and upgrading notes.
- Test deployment on a clean Linux VM.
- Update README and record known limitations and any missing P0 features.
- Create a release candidate, complete acceptance checks, then tag the beta (for example, `v0.1.0-beta.1`).

**Done when:** another user can install SPIRL, configure a protected origin and HTTPS certificate, enable rules, and verify a 403 or 429 without developer assistance.

## Scope protection

The following items must **not** displace P0 work before the beta: WebSockets, dashboards, multiple upstreams, distributed rate limits, optional AI, regex rules, automated certificate management, or additional Linux architectures.

If time is lost, prioritize in this order:

1. Correct and safe proxying.
2. Reliable rate limiting and client identity.
3. Core expression rules and deterministic actions.
4. HTTPS, bounded resources, and negative tests.
5. Safe reload, operational tooling, and polished release assets.

A release without a mandatory security control must be labeled an **early preview**, not an architecture-complete beta. Do not silently weaken correctness requirements to meet a date.

## Project management suggestions

- Use GitHub Issues for each deliverable with a clear acceptance check.
- Use labels `area:proxy`, `area:rules`, `area:limiter`, `area:security`, `area:docs`, and `priority:P0`.
- Merge small pull requests with automated tests; protect the default branch once CI works.
- Update the roadmap weekly and record major architecture changes in a short decision note.

## First five GitHub issues

1. **Bootstrap the Rust binary and CI** — `cargo build`, format, Clippy, and tests pass on Linux.
2. **Build a single-upstream Pingora proof-of-concept** — proxy a local `curl` GET and POST.
3. **Introduce versioned TOML configuration** — validate a minimal listener and upstream file.
4. **Implement a tested token-bucket library module** — prove burst, refill, concurrent correctness, and memory bound.
5. **Define v0.1 expression grammar and golden tests** — cover precedence, typed fields, CIDR membership, and invalid inputs.

## Release decision checklist

- [ ] All FR-01 through FR-16 passed or explicitly deferred with a reclassification of the release.
- [ ] All NFR-01 through NFR-11 verified.
- [ ] No known rule bypass or client-identity spoofing defect.
- [ ] TLS and upstream verification tested.
- [ ] Valid and invalid reload scenarios tested.
- [ ] Memory and concurrency tests completed.
- [ ] CI green and Linux installation independently reproduced.
- [ ] README, sample configuration, release notes, and performance measurements published.
