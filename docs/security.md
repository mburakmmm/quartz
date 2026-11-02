# Quartz Security & Safety Guardrails

> **Framework:** Quartz for NOX  
> **Status:** Security Policy & Implementation Spec  
> **Package:** `quartz`

Quartz is designed from the ground up with defensive security principles to prevent unintended network traffic, memory exhaustion, denial of service, and credential leakage during local, CI, and performance testing.

---

## 1. Network Safety & Target Authorization

### Loopback-by-Default Policy

By default, Quartz strictly disallows external network communication. All contract tests run in-process using direct `(HttpRequest) -> HttpResponse` handler dispatch without opening network sockets.

When network transport is explicitly opted into via `contract/http_network.nox` or `load/executor.nox`:
1. **Loopback Only:** Only `http://127.0.0.1`, `http://[::1]`, and `http://localhost` are permitted without authorization.
2. **Strict Host Authorization:** Target URLs pointing to non-loopback domains (e.g., `https://api.internal.company.com`) are unconditionally rejected with `TargetAuthorizationError` or `LoadSafetyError` unless an exact matching prefix is provided via `authorized_target`.
3. **No Redirect Traversal:** Network transports disable automatic HTTP 3xx redirect following to prevent SSRF (Server-Side Request Forgery) attacks and unexpected egress.

---

## 2. Hard Resource Limits & Pacing

To ensure stability on shared CI runners and local developer machines, Quartz enforces immutable hard caps on all load tests:

| Parameter | Default Safety Limit | Hard Cap Limit | Behavior on Exceeded |
|---|---|---|---|
| **Virtual Users (VU)** | `5` | `20` | Clamped to `HARD_CAP_VIRTUAL_USERS` |
| **Duration** | `5s` | `60s` | Clamped to `HARD_CAP_DURATION_SECONDS` |
| **Rate (RPS)** | `10 req/s` | `100 req/s` | Clamped to `HARD_CAP_RATE_RPS` |
| **In-Flight Requests** | `10` | `50` | Excess arrivals rejected with backpressure |
| **Task Timeout** | `2000ms` | `10000ms` | Clamped to `HARD_CAP_TIMEOUT_MS` (504 status) |

### Circuit Breaker Protection

Every load run includes an active `CircuitBreaker`. If `max_consecutive_errors` (default 5) consecutive failures occur (e.g., connection refused, repeated 500s, or timeouts), the circuit breaker immediately trips, aborting the workload to prevent cascading failures on the target service.

---

## 3. Secret Redaction & Token Masking

Quartz reports, logs, and failure artifacts automatically redact sensitive HTTP headers and credential parameters before persisting them to disk or outputting to JUnit XML / JSON:

- **Redacted Headers:** `Authorization`, `Proxy-Authorization`, `Cookie`, `Set-Cookie`, `X-Api-Key`, `X-Auth-Token`.
- **Masking Format:** Value replaced with `[REDACTED]` or partial leading character retention (e.g., `Bearer abc***`).
- **Disk Artifacts:** Counterexample serialization strips raw authorization payloads from captured requests.

---

## 4. Subprocess Isolation & Watchdog Orchestration

Fuzzing native binaries, untrusted parsers, or C-extensions in-process risks crashing the test runner or hanging indefinitely.

- **Out-of-Process Target Harness:** Fuzz targets can run out-of-process via `nox.process.Command`.
- **Watchdog Deadlines:** Processes exceeding the target execution timeout are terminated with `ProcessError` and classified as `TIMEOUT` rather than hanging the test runner.
- **Corpus Retention Caps:** Fuzz corpus directories enforce `max_files` (default 100) and `max_total_bytes` (default 1MB) to prevent disk space exhaustion.

---

## 5. Streaming & Memory Exhaustion Prevention

Untrusted HTTP responses or fuzz mutator inputs could theoretically trigger Out-Of-Memory (OOM) faults:

- **Chunked Network Streaming:** In `contract/http_network.nox`, HTTP responses are consumed in chunks up to `max_body_bytes` (default 1MB). If a server returns an oversized payload, consumption halts and the payload is truncated with a safety flag.
- **Histogram Memory Bounds:** Latency tracking uses a fixed 246-bucket histogram with $O(1)$ memory, regardless of how many millions of requests are recorded.
