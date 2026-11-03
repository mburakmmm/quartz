# Quartz Architecture & Technical Decision Records (ADRs)

> **Toolchain:** `noxc 2.0.108`  
> **Environment:** Windows x86_64 (Tested for ecosystem cross-platform compatibility)  
> **Date:** 2026-11-02  

---

## 1. System Overview

Quartz is designed as a standalone, zero-dependency testing framework for the NOX ecosystem. It provides property testing, structure-aware fuzzing, HTTP contract validation, and bounded load testing without relying on third-party application frameworks.

```text
                  +-----------------------------------+
                  |            quartz CLI             |
                  |          (cli/main.nox)           |
                  +-----------------+-----------------+
                                    |
                  +-----------------v-----------------+
                  |            quartz.nox             |
                  |          (public facade)          |
                  +-----------------+-----------------+
                                    |
          +-------------------------+-------------------------+
          |                         |                         |
+---------v---------+     +---------v---------+     +---------v---------+
|    quartz.core    |     |  quartz.property  |     |    quartz.fuzz    |
| runner, asserts,  |     | generators, PRNG, |     | mutations, corpus,|
| fixtures, reports |     | shrinker, replay  |     | watchdog harness  |
+---------+---------+     +-------------------+     +-------------------+
          |
+---------v---------+
|  quartz.contract  |
| HTTP / in-process |
| schema validator  |
+---------+---------+
          |
+---------v---------+
|    quartz.load    |
| bounded scheduler |
| latency histogram |
+-------------------+
```

---

## 2. Capability Matrix (Validated on Windows x86_64)

| Feature / Subsystem | Status | Verified Behavior & Notes |
|---|---|---|
| **Derleme Backend** | `QBE: VERIFIED`<br>`LLVM: UNVERIFIED (Needs Clang/Zig)` | On Windows without Clang/Zig on PATH, `noxc` automatically and successfully builds executables using the built-in QBE backend. |
| **Class & Inheritance** | `VERIFIED` | Classes, constructors, fields, and method overriding work reliably. |
| **Generics** | `PARTIAL / RESTRICTED` | Generic functions (`def id[T]`) and generic classes (`class Box[T]`) are supported, but generic classes cannot inherit from a base class (`err_class_generic_with_base`), and class methods cannot have generic parameters (`err_generic_method_rejected`). |
| **Callbacks** | `VERIFIED` | Function types `(A, B) -> C` can be passed as arguments and invoked indirectly. |
| **Exceptions** | `VERIFIED` | `try / except Exception as e / finally` executes `finally` on both success and error paths. Note: `except` requires imported class names (`from mod import Err`), dotted names are rejected. |
| **Concurrency** | `VERIFIED` | `spawn`, `await`, and `Channel[T]` operate under cooperative single-threaded fiber scheduling on QBE. Rate limiters must not rely on blocking `sleep_ms`. |
| **Monotonic Timing** | `VERIFIED` | `nox.time.instant_now().elapsed_ms()` measures monotonic elapsed time cleanly. |
| **File I/O** | `VERIFIED` | `nox.fs.write_string`, `write_string_atomic`, `read_to_string`, and directory operations work with standard `/` paths on Windows. |
| **JSON Parser** | `VERIFIED` | `nox.json.parse` parses into `JsonValue`. Numbers are IEEE-754 float64; integers > 2^53 lose lower-bit precision. |
| **Process Execution** | `VERIFIED` | `nox.process.Command` runs subprocesses, captures status/stdout/stderr, and correctly raises `ProcessError` when `set_timeout` is exceeded. |
| **Local Imports** | `VERIFIED` | Project-internal multi-file imports (`import core.types` or `from core.types import ...`) require `nox.json` in the project root. |

---

## 3. Architectural Decision Records (ADRs)

### ADR-001: Concrete Generator Implementations over Universal `Gen[T]` Hierarchy
- **Context:** `AGENTS.md` posed the open question of whether to use a typed `Gen[T]` hierarchy or concrete generators. Our technical probe revealed that in NOX 2.0.108:
  1. Generic classes cannot inherit from base classes (`class Box(Base)[T]` is rejected by the compiler).
  2. Methods cannot have their own generic type parameters (`def method[T]` is rejected).
- **Decision:** Implement explicit concrete generator classes (`IntGen`, `FloatGen`, `BoolGen`, `StringGen`, `ByteGen`, `ListGen`, `JsonGen`) with a common convention and type-erased container where necessary.
- **Consequences:** Type safety is preserved without fighting compiler monomorphization boundaries.

### ADR-002: Dedicated Explicit-State Quartz PRNG
- **Context:** `nox.random` provides per-worker state, but property-based testing and fuzzing require deterministic, reproducible input streams that can be replayed on any worker, machine, or operating system given identical `(root_seed, test_id, case_index, rng_version)`.
- **Decision:** Quartz Core and Property engines will use a dedicated explicit-state 32-bit / 64-bit PRNG (e.g. LCG/XorShift with golden vectors) that never touches the global random state.
- **Consequences:** Tests produce 100% deterministic results across Windows, macOS (M-series), and Linux.

### ADR-003: Manifest Requirement and `from ... import` Pattern
- **Context:** Without `nox.json`, `noxc` only resolves modules from `stdlib/nox/`. With `nox.json`, `noxc` resolves project-internal paths (e.g. `core/types.nox`). Furthermore, top-level constants like `STATUS_PASS` are accessed cleanest via `from core.types import STATUS_PASS` rather than attribute lookups on non-variable namespace identifiers.
- **Decision:** Maintain `nox.json` with valid `BinSpec` structure and use `from module import ...` for cross-file imports.

### ADR-004: Preservation Strategy for JSON Integers > 2^53 in Contract Testing
- **Context:** Probe 7 demonstrated that `9007199254740993` is parsed as `9007199254740992.0` due to IEEE-754 float conversion in `nox.json`.
- **Decision:** When evaluating contract bodies, integer identity fields exceeding 2^53 must be detected or validated via raw string matching or schema assertions, warning/failing rather than silently rounding precision.

### ADR-005: Cross-Platform Windows & macOS Scheduling Awareness
- **Context:** The NOX language author develops on macOS M4 (LLVM / M:N work-stealing / kqueue), while this test bed runs on Windows x86_64 (currently using QBE cooperative single-threaded fiber runtime).
- **Decision:** Quartz scheduling paths must strictly avoid blocking calls (`sleep_ms`) in workers and must structure channels and tasks so they run deterministically and safely on both single-threaded cooperative QBE and multi-threaded LLVM backends.

### ADR-006: In-Process HTTP Handler Contracts vs. Opt-in Loopback
- **Context:** Integration testing against network sockets introduces port binding collisions, OS firewall prompts, flakiness in CI, and SSRF risks.
- **Decision:** Default to direct in-process `(HttpRequest) -> HttpResponse` handler dispatch for API contract tests. Only activate real socket requests via `contract/http_network.nox` when network transport is explicitly opted into, enforcing loopback-only constraints by default.

### ADR-007: Delta-Debugging Crash Minimization with Signature Preservation
- **Context:** Fuzz testing produces crashes, but raw crashing payloads are often hundreds of bytes with irrelevant noise. Minimizing indiscriminately can shrink a payload to trigger an unrelated earlier exception (e.g., syntax error rather than buffer overflow).
- **Decision:** `minimize_crash` iteratively reduces payload chunks using delta-debugging while asserting that every candidate candidate continues to produce the exact failure signature / error message of the original crash.

### ADR-008: Bounded Latency Histograms and Zero-Allocation Quantiles
- **Context:** Recording individual latency floats for millions of requests causes unbounded memory growth and GC pressure. Capping at 5,000ms distorts percentiles for slow endpoints.
- **Decision:** Implement a fixed 246-bucket monotonic latency histogram covering up to 60,000ms with 500ms intervals above 5,000ms. Quantile computations (p50, p90, p95, p99) run in $O(1)$ memory with mathematical interpolation.

### ADR-009: Decoupled Framework Adapters (Nyx & Aether)
- **Context:** NOX ecosystem frameworks like Nyx and Aether offer routing and middleware paradigms. Direct dependencies on these frameworks in Quartz Core would create circular dependencies or bloat.
- **Decision:** Quartz Core remains completely pure and zero-dependency. Framework adapters reside in `adapters/nyx.nox` and `adapters/aether.nox`, translating framework routers and middleware pipelines into standard `(HttpRequest) -> HttpResponse` handlers.

### ADR-010: Generator Self-Validation and Schema Invariant Preservation
- **Context:** Property-driven contract testing generates request bodies from schema descriptors. If the generator produces an invalid payload when claiming to generate a valid one, false negative contract violations occur.
- **Decision:** `contract/property_contract.nox` enforces a strict self-validation invariant: every generated payload is immediately evaluated against `validate_schema()`. Valid payloads must produce zero errors; negative/invalid payloads must produce at least one error.

### ADR-011: Channel Primitive Parameterization under QBE Runtime
- **Context:** In NOX 2.0.108 with the QBE backend on Windows, parameterized channels with complex custom classes (`Channel[CustomClass]`) can cause exit code 1 during fiber cleanup.
- **Decision:** Internal worker channels use primitive payloads (`Channel[int]`, `Channel[str]`) or serialized string frames. Complex state aggregation is handled via deterministic accumulators and structured structs outside the channel fiber.

### ADR-012: Real M:N Concurrency, Channel[int] Value Streams & Subprocess Toolchain Isolation
- **Context:** Load testing requires real concurrent virtual user execution running simultaneously across CPU cores on LLVM and cooperative fibers on QBE. However, in NOX 2.0.108:
  1. Coroutine task parameters (`async def`) are strictly restricted by the compiler to primitive types (`int/float/bool/str/None/Task[T]/Channel[T]`); custom classes and function callbacks cannot be packed into coroutine frames.
  2. Module exports do not preserve the `async` metadata across files (`from module import async_func` causes the compiler to treat the imported function as synchronous, rejecting `spawn`).
  3. `nox.process.Command` starts child processes with a minimal/empty environment block (missing `PATH`, `TEMP`, `SYSTEMROOT`), causing nested compiler invocations (`noxc test`) in sandboxes to report `FileNotFound`.
- **Decision:**
  1. Virtual user workers are spawned concurrently using `Channel[int]` for non-blocking lock-free metric streaming (`vu_id`, `dur_ms`, `status_code`), completely avoiding heap allocations and memory corruption.
  2. Coroutine task functions are scoped within the module where `spawn` is executed.
  3. The CLI test runner implements a two-stage execution strategy: static type verification via `noxc check <path>` (immune to missing C compiler toolchain in child process) followed by dynamic test execution, providing informative diagnostics when child process compiler execution is restricted by environment sandboxing.

### ADR-013: RFC 6455 WebSocket Handshake & Conversation Protocol Contracts
- **Context:** Real-time applications (streaming chats, metrics tickers, notifications) communicate over WebSockets. Existing HTTP contract testing tools verify single request/response cycles but cannot validate upgrade handshakes or bidirectional message conversations.
- **Decision:** Implement `contract/websocket.nox` with transport-agnostic message verification. Validate RFC 6455 client handshake headers (`Upgrade: websocket`, `Connection: Upgrade`, `Sec-WebSocket-Key`, `Sec-WebSocket-Version: 13`), server 101 status code responses, and multi-step conversation dialogues against typed `SchemaDescriptor` definitions.

### ADR-014: Standalone Zero-Dependency Embedded SVG HTML Reporting
- **Context:** CI systems and developers need rich visual reports (percentile latency curves, distribution charts) without depending on external JavaScript libraries, CDNs, or rendering engines.
- **Decision:** Implement `core/report_html.nox` with zero third-party dependencies, generating clean HTML5 documents with embedded inline SVG vector graphics (`<polyline>`, `<svg width="100%" height="220">`, `<circle>`). Reports open offline in any browser and format properly across light and dark modes.

### ADR-015: Multi-Stage Ramp Workload Scheduling & Composable Generators
- **Context:** Real-world traffic does not jump instantaneously to maximum concurrency; production systems experience traffic ramp-up, steady plateau, and ramp-down. Furthermore, property testing requires skewed probabilistic input distributions (e.g., 80% GET vs 20% POST).
- **Decision:**
  1. Implement `load/stages.nox` with `StagedWorkloadConfig` and `LoadStage(target_vus, duration_s)`. Dynamically interpolate active virtual user count across each stage duration while maintaining O(1) memory metrics and streaming JSONL telemetry (`stage_start`, `stage_complete`).
  2. Implement `property/combinators.nox` providing `OneOfIntGen`, `OneOfStringGen`, `WeightedChoiceIntGen`, `WeightedChoiceStringGen`, and `FilteredIntGen` with bounded discard budgets and invariant-preserving shrink candidates.

### ADR-016: HTTP Mock Record & Replay Fixtures and Run-to-Run Regression Comparison
- **Context:**
  1. Microservice consumer contracts often require testing handlers against external HTTP APIs that cannot be spun up inside isolated CI build containers or without network credentials.
  2. Large test suites accumulate tests across versions; developers and CI gates need an automatic mechanism to compare current run results with a baseline run to immediately flag newly introduced test failures (regressions) and duration drift.
- **Decision:**
  1. Implement `contract/mock.nox` providing `MockExchange`, `MockPlaybackHandler`, and `MockRecorder`. Requests are matched deterministically in-memory by HTTP method and target path (with optional header and body matching), falling back to a strict 404 response when unmatched. Exchanges roundtrip cleanly to disk as standard JSON fixtures.
  2. Implement `core/regression.nox` providing `RunComparison` and `compare_runs()`. Automatically computes newly failing tests (`new_failures`), resolved tests (`fixed_tests`), persistent failures, and total execution duration delta percentages, serializing to machine-readable JSON and console alerts.


