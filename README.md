# 💎 Quartz

<p align="center">
  <strong>The Modern, High-Precision Test Framework & Reliability Toolkit for the NOX Ecosystem</strong>
</p>

<p align="center">
  <a href="https://github.com/mburakmmm/quartz/actions/workflows/ci.yml"><img src="https://github.com/mburakmmm/quartz/actions/workflows/ci.yml/badge.svg" alt="CI Status" /></a>
  <a href="https://github.com/mburakmmm/quartz/releases/latest"><img src="https://img.shields.io/badge/release-v0.9.20-0969da.svg" alt="Latest Release" /></a>
  <a href="https://github.com/mburakmmm/nox-lang"><img src="https://img.shields.io/badge/nox--lang-v2.1.186+-8250df.svg" alt="NOX Language" /></a>
  <a href="https://github.com/mburakmmm/quartz/blob/main/LICENSE"><img src="https://img.shields.io/badge/license-MIT-success.svg" alt="License: MIT" /></a>
  <img src="https://img.shields.io/badge/platforms-Linux%20|%20macOS%20|%20Windows-2ea44f.svg" alt="Multi-Platform Support" />
  <img src="https://img.shields.io/badge/backends-LLVM%20|%20QBE-bf8700.svg" alt="Dual Backend" />
</p>

---

## 📖 Overview

**Quartz** is an industrial-strength, zero-dependency testing engine built from first principles for the [NOX programming language](https://github.com/mburakmmm/nox-lang). It seamlessly complements `nox.test` and `noxc test` without patching compiler internals or requiring runtime reflection.

Designed to scale from unit assertions to mission-critical infrastructure verification, Quartz delivers:
- **Quartz Core** — Non-raising assertions (including regex & order-independent dict equality), explicit test suites, `TempDirFixture` / `MemoryGuardFixture`, tagged lazy & `--pattern` glob test execution, safe-confined snapshot testing, run-to-run regression comparators, and streaming JSONL / responsive HTML5 reporting.
- **Quartz Property** — Composable typed generators, explicit-state 64-bit Knuth LCG determinism, automated delta-debugging shrinking to minimal counterexamples, full-artifact deterministic replay, and `nox.property` interoperability (`property.nox_interop`).
- **Quartz Fuzz** — Byte and structure-aware mutations, BLAKE3-indexed corpus management with `set[str]` $O(1)$ deduplication, crash signature clustering, stateful sequence fuzzing with transition rules, and System-Under-Test (SUT) state models.
- **Quartz Contract** — Zero-overhead in-process HTTP & **native gRPC/Protobuf** (`contract.grpc`) handler contracts, HTTP/2 & connection pooling transport options, exact `schema_int64` ($> 2^{53}$) & regex pattern constraints, OpenAPI 3 subset validation, WHATWG SSE, RFC 6455 WebSockets, and offline HTTP Mock record & playback.
- **Quartz Load** — Safe bounded concurrent load testing (`ConcurrentWaveScope` & `TaskGroupRunner`), reference-counted atomic lifetime safety (`AtomicInt` / `AtomicRef`), nanosecond sub-ms precision metrics (`monotonic_ns`), `nox.mem.stats` allocation telemetry, honest `peak_active_vus` vs overlapping-span `peak_concurrency`, closed VU and open rate pacing, 336-bucket microsecond HDR-style + 246-bucket ms latency histograms (nearest-rank p50, p90, p95, p99), and safety circuit breakers.
- **Opt-In Desktop Accelerators (`accel/`)** — Isolated C-backed high-throughput adapters for `nox.yyjson` (relaxed JSON/JSON5 schema validation), `nox.pcre2` (PCRE2 + JIT named-group regex matching), and `nox.xxhash` (XXH3-64 / XXH3-128 corpus indexing) without adding native link dependencies to portable `quartz.nox`.
- **Ecosystem Adapters** — Zero-coupling reference models and verification bridges for **Atlas** (tabular DataFrames & CSV) and **Feather** (ORM & SQL transaction rollbacks with SQLite / PostgreSQL), plus **Nyx** and **Aether**.

---

## 🏛️ Architecture

```
                       ┌────────────────────────────────────────┐
                       │               quartz CLI               │
                       │     dynamic runner, tags & replay      │
                       └───────────────────┬────────────────────┘
                                           │
                       ┌───────────────────▼────────────────────┐
                       │              quartz.core               │
                       │    runner, suites, fixtures, report    │
                       └───┬───────────────┬────────────────┬───┘
                           │               │                │
                 ┌─────────▼─────┐   ┌─────▼───────┐   ┌────▼──────────┐
                 │    property   │   │    fuzz     │   │   contract    │
                 │   generators  │   │  mutations  │   │  HTTP/schema  │
                 │    shrinker   │   │   corpus    │   │  SSE/WebSocket│
                 │    registry   │   │  clustering │   │  mock engine  │
                 └───────────────┘   └─────────────┘   └────┬──────────┘
                                                            │
                                                   ┌────────▼──────────┐
                                                   │    load engine    │
                                                   │  bounded VU waves │
                                                   │  histograms & p95 │
                                                   └───────────────────┘
                                           ▲
                                           │ (zero core coupling)
                        ┌──────────────────┴──────────────────┐
                        │         Ecosystem Adapters          │
                        │    Atlas · Feather · Nyx · Aether   │
                        └─────────────────────────────────────┘
```

---

## ⚡ Feature Matrix

| Module | Core Capabilities | Highlights |
|---|---|---|
| **`quartz.core`** | Assertions, TestSuites, Fixtures, Tag Filtering, Snapshots, Regression Engine | Non-raising `check_*`, `--tag` and `--exclude-tag` lazy execution, `with` context manager fixtures, golden snapshot diffs, run-to-run drift tracking, HTML5 visual dashboard with SVG latency distribution. |
| **`quartz.property`** | Composable Generators, PRNG Determinism, Counterexample Shrinking, Replay Registry | Boundary-biased integers, floats, Turkish & emoji Unicode, combinators (`OneOf`, `WeightedChoice`, `Filtered`, `Mapped`, `PairIntGen`), 64-bit Knuth LCG, delta-debugging shrinker, `PropertyRegistry` with Windows backslash-safe JSON artifacts. |
| **`quartz.fuzz`** | Byte/Structure Mutation, Corpus Storage, Deduplication, Sequence Fuzzing, SUT Models | In-process and watchdog harnesses, FNV-1a content addressing, stack trace & address normalization, `CrashDeduplicator`, stateful sequence fuzzer with transition rules & initial actions, `StateModel` / `StateModelStr`. |
| **`quartz.contract`** | In-Process & Network HTTP, OpenAPI 3, SSE, WebSocket, Mock Engine | Direct in-process handler invocation (sub-millisecond, zero open ports), loopback safety gates, WHATWG SSE parser, RFC 6455 WebSocket handshakes, offline HTTP record & playback with secret redaction. |
| **`quartz.load`** | Closed VU & Open Arrival Models, Measured Overlap Concurrency, Monotonic Histograms | Real OS-thread concurrent HTTP Virtual User execution (`run_network_load_test`) & `TaskGroupRunner`, reference-counted atomic safety under timeouts, 336-bucket microsecond + 246-bucket ms monotonic histograms (`p50_us`/`p95_us`/`p99_us`), multi-stage ramp schedulers, circuit breakers. |
| **`quartz.adapters`** | Atlas DataFrames, Feather ORM, Nyx, Aether | `AtlasTable` CSV roundtrip & fuzzing, sort idempotence, ground-truth join & groupby contracts, SQL `NULL` vs `""` preservation, `FeatherRecord` CRUD invariants, transaction rollback verification, real SQLite (`libsqlite3`) & `nox.orm` tests. |

---

## 🚀 Quickstart

### 1. Add Quartz to your `nox.json`

```json
{
  "name": "my_nox_service",
  "version": "1.0.0",
  "dependencies": {
    "github.com/mburakmmm/quartz": "v0.9.20"
  }
}
```

### 2. Standard Toolchain Commands

Quartz uses the standard, sanctioned NOX workflow:

```bash
# Verify type correctness
noxc check quartz.nox

# Run all test suites
noxc test

# Run a specific test suite
noxc test tests/core_test.nox

# Execute the Quartz CLI entry point
noxc run cli/main.nox -- version
```

---

## 💡 Code Showcases

### 1. Unit & Fixture Testing (`core`)

```nox
from core.suite import TestSuite
from core.fixture import Fixture, SCOPE_CASE
from core.assert import assert_eq_int, assert_true

suite: TestSuite = TestSuite("payment_service")

# Tagged lazy tests: skipped tests are never invoked
def test_fast_calculation() -> bool:
    return (2 + 2) == 4

suite.run_test("math_check", test_fast_calculation, ["smoke", "fast"])
suite.check_eq_int("balance_check", 100, 100)

assert_true(suite.all_passed(), "all checks must pass")
```

### 2. Property-Based Testing with Shrinking & Replay (`property`)

```nox
from property.generator import IntGen
from property.property import PropertyConfig, run_int_property, PropertyResult
from property.replay import PropertyRegistry, save_failure_artifact

gen: IntGen = IntGen(0, 500, with_boundaries=True)
cfg: PropertyConfig = PropertyConfig(cases=100, max_discard_ratio=5, seed=9999, timeout_ms=5000)

def invariant_under_150(x: int) -> bool:
    return x < 150

# Finds counterexample, shrinks via delta-debugging & halving to minimal 150
result: PropertyResult = run_int_property("bounded_check", cfg, gen, invariant_under_150)

# Save deterministic JSON artifact and register in PropertyRegistry
save_failure_artifact("artifacts/failure.json", result, "my_test.nox")

reg: PropertyRegistry = PropertyRegistry()
reg.register_int("bounded_check", gen, invariant_under_150)
assert reg.replay_result(result) == True
```

### 3. Stateful Sequence Fuzzing & SUT Model (`fuzz`)

```nox
from fuzz.sequence import FuzzAction, SequenceFuzzer, StateModelStr, evaluate_sut_against_str_model

fuzzer: SequenceFuzzer = SequenceFuzzer(seed=42, action_names=["login", "add_item", "checkout"], min_length=2, max_length=10)
fuzzer.add_transition_rule("login", ["add_item"])
fuzzer.add_transition_rule("add_item", ["add_item", "checkout"])
fuzzer.set_initial_actions(["login"])

# Model-based verification: verifies SUT state against expected model step-by-step
outcome = evaluate_sut_against_str_model(cart_model, sut_reset, sut_step, fuzzer.generate_random_sequence(), fail_on_precondition=True)
if not outcome.passed:
    print("Violation kind:", outcome.violation_kind, "at step:", outcome.step_index)
```

### 4. Zero-Overhead In-Process HTTP Contracts & Mocking (`contract`)

```nox
from contract.http import build_request, empty_headers
from contract.schema import schema_object, schema_string, schema_integer, validate_schema
from contract.mock import HttpMockPlayer, HttpMockRecorder
from nox.http import HttpRequest, HttpResponse

# Test your HTTP handlers directly in-memory (no open ports, no OS network overhead)
req: HttpRequest = build_request("POST", "/v1/users", "{\"name\":\"Ada\",\"age\":36}", empty_headers())
resp: HttpResponse = user_handler(req)

assert resp.status == 201
```

### 5. High-Concurrency Bounded Load Testing (`load`)

```nox
from load.workload import closed_workload, WorkloadConfig
from load.safety import default_safety_limits, SafetyLimits
from load.thresholds import Threshold
from load.executor import run_task_workload, LoadTestSummary

wl: WorkloadConfig = closed_workload("api_benchmark", virtual_users=4, duration_seconds=5, max_iterations=100)
limits: SafetyLimits = default_safety_limits()
thresholds: list[Threshold] = [
    Threshold("p95_latency_ms", "<=", 250.0),
    Threshold("error_rate", "<=", 0.0)
]

def benchmark_task(case_idx: int) -> bool:
    return process_order(case_idx)

# Measures active VU waves, execution span overlap, and 336-bucket microsecond + 246-bucket ms histograms
summary: LoadTestSummary = run_task_workload(wl, benchmark_task, thresholds, limits)
print("p95 latency:", summary.metrics.p95(), "ms (", summary.metrics.p95_us(), "us); Throughput:", summary.metrics.throughput_rps(summary.total_duration_ms), "req/s")
```

### 6. Ecosystem Adapters: Atlas DataFrames & Feather ORM (`adapters`)

```nox
import nox.sqlite
from adapters.atlas import AtlasTable, from_db_rows, verify_sort_contract, verify_inner_join_contract
from adapters.feather import run_orm_transaction, TX_COMMITTED, TX_ROLLED_BACK_ON_ERROR

# Atlas: Preserves SQL NULL vs empty-string distinction in canonical signatures
db_rows = conn.query("SELECT id, name, bio FROM users")
table: AtlasTable = from_db_rows(db_rows)
assert table.is_null_cell(0, "bio") == True

# Feather: Guarantees automatic recovery rollback on COMMIT or statement errors
res = run_orm_transaction(conn, users_table, "id", mutation_fn, commit=True)
assert res.outcome == TX_COMMITTED
```

---

## 🛠️ CLI Reference

Quartz includes a self-contained CLI entry point (`cli/main.nox`):

```bash
# Run tests with tag & glob pattern filtering
noxc run cli/main.nox -- test tests/core_test.nox --tag smoke
noxc run cli/main.nox -- test tests --pattern "*contract*"
noxc run cli/main.nox -- test fixtures/tag_demo.nox --exclude-tag slow

# Run property-based testing against custom target or demo
noxc run cli/main.nox -- property --target tests/property_test.nox --cases 500 --seed 12345

# Replay a failure artifact against a target property test
noxc run cli/main.nox -- replay artifacts/failures/case_37.json --target tests/property_test.nox

# Run bounded fuzzer on a user or sample target
noxc run cli/main.nox -- fuzz --target my_fuzz_target.nox --max-cases 1000

# Validate OpenAPI contracts or live loopback endpoints
noxc run cli/main.nox -- contract --spec openapi.json --base-url http://127.0.0.1:8080 --allow-network

# Run authorized bounded load tests against a target URL or user script
noxc run cli/main.nox -- load --url http://127.0.0.1:8080/health --allow-network --vus 4 --duration 5 --requests 100

# Stream test events as real-time JSON Lines or generate visual HTML reports
noxc run cli/main.nox -- test tests --format jsonl
noxc run cli/main.nox -- test tests --format html --out reports/summary.html
```

---

## 🌐 Platform & Backend Compatibility

Quartz is continuously tested across a 4-platform matrix with dual NOX backends on every push:

| Platform | Architecture | Runner | QBE Backend | LLVM Backend |
|---|---|---|:---:|:---:|
| **Linux** | `x64` | `ubuntu-latest` | ✅ | ✅ (M:N Work-Stealing) |
| **Linux** | `arm64` | `ubuntu-24.04-arm` | ✅ | ✅ (M:N Work-Stealing) |
| **macOS** | `arm64` | `macos-latest` (Apple Silicon) | ✅ | ✅ (M:N Work-Stealing) |
| **Windows** | `x64` | `windows-latest` | ✅ | ✅ (Auto-Fallback) |

---

## 📚 Deep Documentation

For full architectural records, formal specifications, and complete recipes, explore the documentation suite:

| Guide | Description |
|---|---|
| [**Architecture & ADRs**](docs/architecture.md) | Architectural Decision Records ADR-001 through ADR-018 detailing design choices and boundaries. |
| [**Complete API Reference**](docs/api.md) | Detailed function signatures and type contracts across all 50 Quartz modules. |
| [**Recipes & Cookbook**](docs/recipes.md) | 18 step-by-step production recipes covering property tests, fuzzers, contracts, load, and adapters. |
| [**Security & Guardrails**](docs/security.md) | Loopback policies, secret redaction, safe integer bounds ($2^{53}$), and sandbox limits. |
| [**Compatibility Matrix**](docs/compatibility.md) | Platform capabilities, timer precision, and compiler backend behavioral parity. |
| [**Changelog**](CHANGELOG.md) | Full historical log of all releases from `v0.1.0` through `v0.9.20`. |

---

## 📄 License

Quartz is licensed under the [MIT License](LICENSE).  
Copyright © 2026 Quartz Contributors & NOX Ecosystem.
