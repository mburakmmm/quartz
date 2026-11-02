# Quartz — Implementation Roadmap & Task Tracker (`todo.md`)

> **Ecosystem:** NOX Programming Language (`nox-lang` v2.0.108+)  
> **Status:** Phase 0 Active / Foundation  
> **Target Version:** v0.1.0 (Phase 1) → v1.0.0 (Phase 7)  
> **Operating Contract:** Strictly adhere to `AGENTS.md` non-negotiable rules. Zero unverified claims, zero skipped phases.

---

## Task Status Legend
- `[ ]` **TODO**: Planned, not yet started.
- `[-]` **IN PROGRESS**: Currently being executed.
- `[x]` **DONE**: Completed, compiler-checked, tests executed and passing, docs updated, failure paths covered.
- `[!]` **BLOCKED / NOT RUN**: Blocked by compiler limitation or environment constraint (documented explicitly).

---

## Phase 0: Technical Probes, Capability Matrix & Baseline Skeleton
*Objective: Audit NOX compiler/stdlib capabilities, verify language constructs via compiling probes, establish package skeleton and initial test contract.*

- [x] **0.1 Environment & Toolchain Audit**
  - [x] Verify `noxc --version`, check LLVM and QBE backend availability on Windows.
  - [x] Inspect available stdlib modules (`nox.test`, `nox.fs`, `nox.time`, `nox.json`, `nox.random`, `nox.http`, `nox.process`).
  - [x] Document target platforms (`windows-x64` local, QBE fallback without clang/zig, hosted profile).
- [x] **0.2 Language Capability Mini-Probes (Technical Spikes)**
  - [x] Probe 1: Class inheritance, methods, fields, and constructors (`probes/probe_1_classes.nox`).
  - [x] Probe 2: Generics syntax (`[T]`), method restrictions, function-value callbacks `(A) -> B` (`probes/probe_2_*.nox`).
  - [x] Probe 3: Exception handling (`try / except Exception as e / finally`) (`probes/probe_3_exceptions.nox`).
  - [x] Probe 4: Concurrency primitives (`spawn`, `await`, `Channel[T]`) under QBE (`probes/probe_4_*.nox`).
  - [x] Probe 5: Monotonic timer precision (`nox.time.instant_now()`) (`probes/probe_5_time.nox`).
  - [x] Probe 6: File I/O (`nox.fs.write_string`, atomic writes, directory creation) (`probes/probe_6_fs.nox`).
  - [x] Probe 7: JSON parser float behavior, large integer precision (>2^53 limits) (`probes/probe_7_json.nox`).
  - [x] Probe 8: Process execution with timeouts (`nox.process.Command`) (`probes/probe_8_*.nox`).
- [x] **0.3 Project Layout & Package Initialization**
  - [x] Create `nox.json` with package metadata, entry points, `BinSpec`, and `nox.lock`.
  - [x] Create base directory structure (`core/`, `tests/`, `docs/`, `fixtures/`, `cli/`).
  - [x] Create public facade `quartz.nox`.
  - [x] Create initial `README.md`, `CHANGELOG.md`, `LICENSE`.
- [x] **0.4 Baseline Test & Initial Result Contract**
  - [x] Write minimal `tests/baseline_test.nox` runnable via `noxc test`.
  - [x] Define initial JSON execution result schema contract (`schema_version: "1"` in `core/types.nox`).
  - [x] Verify intentional failure handling and non-zero exit codes (tested via deliberate failure assertion).
  - [x] Document Architectural Decision Records (ADRs) for Phase 0 in `docs/architecture.md`.

*Phase 0 Exit Gate:* PASSED. `noxc test` discovers and runs `baseline_test.nox` cleanly; deliberate failure exits non-zero; compiler capabilities and limitations recorded in `docs/architecture.md`.

---

## Phase 1: v0.1.0 — Quartz Core MVP
*Objective: Production-grade test runner, assertion suite, fixture lifecycle, and machine-readable reports (JSON/JUnit).*

- [x] **1.1 Core Types & Data Models (`core/types.nox`)**
  - [x] Define test status enumeration: `PASS`, `FAIL`, `SKIP`, `ERROR`, `TIMEOUT`.
  - [x] Define `TestCase`, `TestResult`, `Failure`, `RunSummary`, `ResourceLimits`, `RunnerConfig`.
  - [x] Ensure full diagnostic capture: test ID, duration, status, failure message.
- [x] **1.2 Assertion Library (`core/assert.nox`)**
  - [x] Implement `assert_eq_int`, `assert_eq_str`, `assert_eq_bool`, `assert_true`, `assert_false`.
  - [x] Implement `assert_eq_float` with explicit absolute and relative tolerances.
  - [x] Implement `assert_not_eq_int`, `assert_not_eq_str`, `assert_contains_str`, `assert_not_contains_str`.
  - [x] Negative assertion tests catching `AssertionError`.
- [x] **1.3 Test Suite & Runner (`core/suite.nox`)**
  - [x] Create `TestSuite` with explicit case registration and execution.
  - [x] Implement non-raising `check_*` methods that accumulate results without halting test flow.
  - [x] Add test filtering by name/pattern and skip accounting.
  - [x] Implement deterministic result counting and status tracking.
- [x] **1.4 Fixture Lifecycle (`core/fixture.nox`)**
  - [x] Case-level and suite-level setup and teardown lifecycle using strict `try / finally`.
  - [x] Ensure teardown always runs on pass, failure, or error.
  - [x] Context manager support with `__enter__` and `__exit__`.
  - [x] Isolation and cleanup validation under exceptions.
- [x] **1.5 Reporting & Exporters (`core/report.nox`)**
  - [x] Implement JUnit XML report generator compatible with CI parsers.
  - [x] Implement versioned summary JSON report exporter.
  - [x] Add secret redaction mechanism (masking tokens, passwords, authorization headers).
  - [x] Atomic file writes (`write_junit_report`, `write_json_report`).
- [x] **1.6 Phase 1 Verification & Tests**
  - [x] `tests/core_test.nox`: 25+ positive, negative, and edge-case assertions.
  - [x] Verify fixture cleanup under deliberate assertion failures.
  - [x] Validate JUnit XML and JSON summary output.
  - [x] Update `CHANGELOG.md` and `docs/api.md` for v0.1.0.

*Phase 1 Exit Gate:* PASSED. Full core test suite passes via `noxc test` (2 suites, 25+ checks); zero state leaks in fixtures; valid JSON and JUnit reports emitted.

---

## Phase 2: v0.2.0 — Property-Based Testing & Shrinking
*Objective: Composable typed generators, explicit-state PRNG, property runner, and deterministic counterexample shrinking.*

- [x] **2.1 Deterministic Explicit-State PRNG (`core/prng.nox`)**
  - [x] Implement standalone 64-bit Knuth MMIX LCG (independent of global `nox.random`).
  - [x] Seed mixing routine: `derive_case_seed(root_seed, test_id, case_index, rng_version)`.
  - [x] Golden vector tests ensuring cross-platform reproducible sequences (`tests/prng_test.nox`).
- [x] **2.2 Input Generators (`property/generator.nox`)**
  - [x] Integer generator: bounded ranges, boundary-biased (`min`, `max`, `0`, `1`, `-1`).
  - [x] Float generator: finite values, uniform distribution in range.
  - [x] Boolean generator.
  - [x] String generator: ASCII, Unicode (Turkish characters `ç, ğ, ı, ö, ş, ü`, etc.).
  - [x] Byte generator: raw buffers with length bounds.
  - [x] Composite generators: ListIntGen with item generators.
- [x] **2.3 Property Runner & Invariants (`property/property.nox`)**
  - [x] Configurable case count (default 100), max discards, timeout per case.
  - [x] Discard tracking and exhaustion handling.
  - [x] Capture initial failing counterexample with seed and case index before shrinking.
- [x] **2.4 Counterexample Shrinker (`property/shrink.nox`)**
  - [x] Integer shrinker: towards zero, sign flip, binary search / halving (`// 2`).
  - [x] String shrinker: empty string, delta-debugging chunk elimination, tail reduction.
  - [x] List shrinker: empty list, delta-debugging chunk deletion, element shrinking.
  - [x] Invariant-preserving validation: ensure candidate retains failure signature.
- [x] **2.5 Replay Engine (`property/replay.nox`)**
  - [x] Replay failure artifact from saved seed, test ID, and case index (`replay_int_case`, `replay_str_case`).
  - [x] Serialize failure artifacts with minimal counterexamples to JSON (`save_failure_artifact`).
- [x] **2.6 Phase 2 Verification & Tests**
  - [x] `tests/prng_test.nox`: golden vector repeatability across runs.
  - [x] `tests/property_test.nox`: 1,000-case deterministic invariant tests.
  - [x] `tests/shrink_test.nox`: verify deterministic minimal counterexample extraction.
  - [x] Update `CHANGELOG.md` and `docs/api.md` for v0.2.0.

*Phase 2 Exit Gate:* PASSED. 1,000 randomized property cases reproduce identically; seeded failures shrink to minimal counterexamples; replay reproduces failures cleanly.

---

## Phase 3: v0.3.0 — Fuzz Engine & Corpus Management
*Objective: Byte and structured mutation fuzzing, corpus persistence, crash deduplication, and watchdog harness.*

- [x] **3.1 Fuzz Target Harness (`fuzz/harness.nox`)**
  - [x] Target interface definition: `run_in_process(target_func, input_bytes) -> FuzzOutcome`.
  - [x] Classification: `NORMAL`, `ASSERTION_VIOLATION`, `HANDLED_REJECTION`, `TIMEOUT`, `CRASH`.
  - [x] In-process harness for pure decoders/parsers with monotonic duration timing.
  - [x] Out-of-process harness (`run_out_of_process`) with watchdog process timeout.
- [x] **3.2 Mutation Engine (`fuzz/mutation.nox`)**
  - [x] Byte mutations: bit flip, byte flip, byte swap, insert, delete, duplicate, splice.
  - [x] Boundary number mutation (`INT64_MIN`, `INT64_MAX`, `2^53`, `NaN`, `Infinity`).
  - [x] Structured mutations: JSON (unclosed quotes, truncated brackets, duplicate fields), CSV (delimiters, quotes).
- [x] **3.3 Corpus Management (`fuzz/corpus.nox`)**
  - [x] Directory layout: `corpus/<target>/{seeds, crashes, timeouts, regressions}`.
  - [x] Stable content-based hashing / file naming via FNV-1a hex digests.
  - [x] Hard resource caps: max corpus size, max file count, max input bytes.
  - [x] Safe atomic writes avoiding partial crash artifacts.
- [x] **3.4 Crash Minimization (`fuzz/minimize.nox`)**
  - [x] Delta-debugging minimizer for crash inputs while preserving exact error signature.
  - [x] Crash deduplication by failure signature.
- [x] **3.5 Phase 3 Verification & Tests**
  - [x] `tests/fuzz_test.nox`: parser fuzzing smoke test with known bug seed.
  - [x] Replay verified crash artifacts to guarantee reproduction.
  - [x] Verify bounded execution: timeout enforcement and length caps.
  - [x] Update `CHANGELOG.md` and `docs/api.md` for v0.3.0.

*Phase 3 Exit Gate:* PASSED. Target parser crashes discovered from seeds, minimized down to exact crash tokens, classified, and replayed; zero unhandled resource leaks.

---

## Phase 4: v0.4.0 — HTTP Contract Testing & Schema Validation
*Objective: In-process HTTP handler contracts, schema validator, OpenAPI subset importer, and secret redaction.*

- [x] **4.1 Transport Abstraction & In-Process Handler (`contract/http.nox`)**
  - [x] Transport-neutral `HttpRequest` and `HttpResponse` abstractions (`build_request`, `build_response`).
  - [x] In-process handler executor (`execute_handler`) with monotonic nanosecond timing and zero network ports.
  - [x] Default prohibition of external network connections.
- [x] **4.2 Contract & Operation Descriptors (`contract/types.nox`)**
  - [x] Define `ContractCheckResult`, case-insensitive header helpers (`get_header_ci`, `has_header_ci`).
  - [x] Secret redaction for authorization, cookie, and API key tokens (`redact_header_value`).
- [x] **4.3 Supported Schema Validator (`contract/schema.nox`)**
  - [x] Type validation: `null`, `boolean`, `string`, `number`, `integer`, `array`, `object`, `enum`.
  - [x] Large integer precision guard (>2^53 safe integer bounds check).
  - [x] Constraints: required fields, min/max values, string lengths, enum sets.
  - [x] Unknown properties policy (configurable `is_strict` mode: reject unknown keys).
  - [x] Explicit errors on unsupported JSON Schema / OpenAPI constructs.
- [x] **4.4 Comparison & Diff Engine (`contract/compare.nox`, `contract/runner.nox`)**
  - [x] Case-insensitive header matching with normalized media type / charset handling.
  - [x] Status code validation against allowed status set.
  - [x] Declarative contract test case builder (`ContractTestCase`).
- [x] **4.5 OpenAPI Subset Importer (`contract/openapi.nox`)**
  - [x] Parse JSON-formatted OpenAPI 3.x documents (`parse_openapi_json`) extracting operations and response schemas.
  - [x] Raise `UnsupportedContractFeature` on unhandled OpenAPI constructs (`$ref`, `oneOf`, `anyOf`, `allOf`, `not`).
- [x] **4.6 Phase 4 Verification & Tests**
  - [x] `tests/contract_handler_test.nox`: positive and negative in-process contract tests (7 comprehensive checks).
  - [x] Mismatch verification on missing fields, wrong types, status code discrepancies.
  - [x] Secret redaction tests (verifying tokens are masked in test reports).
  - [x] Update `CHANGELOG.md` and `docs/api.md` for v0.4.0.

*Phase 4 Exit Gate:* PASSED. In-process contract suite passes; precise mismatch errors generated; integer 2^53 limits enforced; external network access zero-traffic by default.

---

## Phase 4.1: v0.4.1 — Reliability, Correctness & Hardening Bugfix Release
*Objective: Resolve external audit findings across Core, Property, Fuzz, and Contract modules before advancing.*

- [x] **4.1.1 Core & Property Hardening**
  - [x] `core/assert.nox`: Add explicit IEEE 754 NaN handling in `assert_eq_float` (`actual != actual` or `expected != expected`).
  - [x] `property/shrink.nox`: Exception-safe shrink evaluation (`safe_eval_int`, `safe_eval_str`, `safe_eval_list_int`).
  - [x] `property/property.nox`: Escape special characters in `PropertyResult.to_json()` with `escape_json`.
- [x] **4.1.2 Fuzzing Engine Hardening**
  - [x] `fuzz/harness.nox`: Replace loose substring rejection heuristic with explicit `HandledRejection` exception class and strict prefix handling to prevent misclassifying real bugs as rejections.
  - [x] `fuzz/corpus.nox`: Enforce `max_files` and `max_total_bytes` limits in `add_seed()` and `save_crash_artifact()`.
  - [x] `fuzz/corpus.nox`: Escape fields in crash artifact JSON.
  - [x] `fuzz/fuzzer.nox`: Add `fail_on_bug: bool` and `timeout_ms` overall session deadline to `run_fuzz()`.
- [x] **4.1.3 OpenAPI Importer & Contract Hardening**
  - [x] `contract/openapi.nox`: Implement validation constraints in `parse_schema_node()`: `minLength`, `maxLength`, `minimum`, `maximum`, `minItems`, `maxItems`.
  - [x] `contract/openapi.nox`: Disallow silent keyword omissions; raise `UnsupportedContractFeature` for unsupported keywords (`discriminator`, `pattern`, etc.).
  - [x] `contract/openapi.nox`: Respect OpenAPI `additionalProperties` default (`is_strict = False` unless `additionalProperties: false`).
  - [x] `contract/openapi.nox`: Safely parse OpenAPI status codes including `"default"` and `"2XX"` wildcard patterns.
  - [x] `contract/schema.nox`: Use `>= SAFE_INT_MAX` boundary guard for integer precision loss.
  - [x] `contract/compare.nox`: Normalize Content-Type media types by trimming parameters before `;`.
- [x] **4.1.4 Verification & Tests**
  - [x] Add unit tests verifying each bugfix in existing and dedicated tests.
  - [x] Update `CHANGELOG.md` and `docs/api.md` for v0.4.1.
  - [x] Tag `v0.4.1`, push, verify CI green, and run `noxc publish`.

*Phase 4.1 Exit Gate:* PASSED. Zero unhandled resource limits in corpus, no silent OpenAPI keyword omissions, strict fuzz rejection classification, all 7 test suites passing cleanly.

---

## Phase 5: v0.5.0 & v0.6.0 — Authorized Local HTTP & Bounded Load Testing
*Objective: Authorized loopback transport, bounded load workload models, latency percentiles, and safety gates.*

- [x] **5.1 Authorized Loopback Network Transport (`contract/http_network.nox` - v0.5.0)**
  - [x] Opt-in network transport using `nox.http.request` with timeouts.
  - [x] Hard safety gate: restrict strictly to `127.0.0.1` / `::1` / `localhost` unless explicitly authorized.
  - [x] Reject all remote hosts without `--authorized-target` flag / authorized target policy.
  - [x] Inactive redirect policy by default.
  - [x] `tests/contract_http_test.nox`: 6 comprehensive checks verifying all safety gates.
- [x] **5.2 Safety Guardrails & Policy Validator (`load/safety.nox` - v0.6.0)**
  - [x] Hard caps: max virtual users (default <= 5), duration (<= 10s), rate (<= 10 req/s), in-flight (<= 10).
  - [x] Safety circuit breaker: instant abort on repeated transport errors or connection refusals.
  - [x] Strict local-only target validator.
- [x] **5.3 Workload Models & Scheduler (`load/workload.nox`, `load/executor.nox` - v0.6.0)**
  - [x] Closed workload model: N virtual users with task scheduling.
  - [x] Open workload model: target arrival rate with bounded queue and backpressure.
  - [x] Generic task workload executor (`run_task_workload`) and HTTP handler load runner (`run_handler_load_test`).
- [x] **5.4 Monotonic Latency Metrics & Histograms (`load/metrics.nox` - v0.6.0)**
  - [x] Monotonic timing with `nox.time.instant_now()`.
  - [x] Bounded latency histogram (136 buckets) to prevent unbounded memory growth.
  - [x] Percentile calculations: p50, p90, p95, p99, min, max, mean.
  - [x] Request accounting: scheduled, completed, failed, timed out, rejected.
- [x] **5.5 Performance Thresholds (`load/thresholds.nox` - v0.6.0)**
  - [x] Threshold definitions: `p95_latency_ms <= X`, `error_rate <= Y`, `throughput_rps >= Z`.
  - [x] Threshold evaluation and non-zero exit codes on breach.
  - [x] Handling edge cases (zero successful requests must never pass a latency threshold).
- [x] **5.6 Phase 5 Verification & Tests**
  - [x] `tests/contract_http_test.nox`: authorized loopback tests against local endpoints.
  - [x] `tests/load_scheduler_test.nox`: verify bounded workers, open vs closed models, circuit breaker abort.
  - [x] `tests/load_metrics_test.nox`: verify percentile monotonicity and histogram bounds.
  - [x] Update `CHANGELOG.md` and `docs/api.md` for v0.5.0 and v0.6.0.

*Phase 5 Exit Gate:* PASSED. Concurrency and memory caps verified under slow targets; latency percentiles strictly monotonic; circuit breaker trips on repeated errors; unauthorized targets unconditionally blocked. 10/10 test suites pass.

---

## Phase 5.3: v0.6.1 — Concurrency, OpenAPI Boundaries & Load Engine Hardening
*Objective: Resolve external audit findings on OpenAPI single-sided bounds, load engine concurrency/pacing, JSON compliance, and histogram overflow.*

- [x] **5.3.1 OpenAPI Single-Sided Bounds (`contract/schema.nox`, `contract/openapi.nox`)**
  - [x] Separate `has_min` and `has_max` flags in `schema_integer` and `schema_number`.
  - [x] Support single-sided bounds (e.g. `{ "type": "integer", "minimum": 5 }` without setting maximum to 0).
  - [x] Add regression test in `tests/contract_handler_test.nox` verifying single-sided minimum and maximum schemas.
- [x] **5.3.2 Load Test JSON Compliance & Escaping (`load/executor.nox`)**
  - [x] Serialize booleans as lowercase `"true"` / `"false"` per JSON standard.
  - [x] Sanitize strings (`workload_name`, `abort_reason`) using `escape_json`.
- [x] **5.3.3 Load Metrics Overflow & High-Latency Accuracy (`load/metrics.nox`)**
  - [x] If latency exceeds 5000ms (bucket 135), calculate percentiles using `max_latency_ms` rather than capping at 5000ms.
  - [x] Track `overflow_count` and serialize in `LoadMetrics.to_json()`.
- [x] **5.3.4 Virtual User Interleaving & Rate Pacing (`load/executor.nox`, `load/workload.nox`)**
  - [x] Interleaved multi-VU execution simulating N concurrent virtual users in closed workload model.
  - [x] Arrival rate pacing in open workload model (`target_rps` interval scheduling).
  - [x] Real in-flight concurrency tracking with backpressure rejection.
  - [x] Enforce per-task `timeout_ms` watchdog marking tasks as `is_timeout=True` on breach.
- [x] **5.3.5 Streaming Body Protection (`contract/http_network.nox`)**
  - [x] Chunked streaming via `nox.http.stream()` up to `max_body_bytes` to prevent memory exhaustion before truncation.
- [x] **5.3.6 Verification & Release**
  - [x] Update `tests/load_metrics_test.nox`, `tests/load_scheduler_test.nox`, and `tests/contract_handler_test.nox`.
  - [x] Update `CHANGELOG.md` and `docs/api.md` for v0.6.1.
  - [x] Tag `v0.6.1`, push, verify CI across Windows x64, Linux x64/arm64, macOS arm64, and run `noxc publish`.

*Phase 5.3 Exit Gate:* PASSED. Zero false rejections on single-sided OpenAPI bounds; valid lowercase boolean JSON; accurate percentiles on high latencies; multi-VU pacing verified; all 10 test suites pass.

---

## Phase 6: v0.7.0 & v0.8.x — Cross-Module Hardening & Integrations
*Objective: Property-driven contracts, framework adapters, LLVM/QBE backend matrix, and CI hardening.*

- [ ] **6.1 Property-Driven Contract Testing**
  - [ ] Feed Quartz Property generators into HTTP contract payloads.
  - [ ] Negative schema testing via generator fuzzing.
- [ ] **6.2 Ecosystem Adapters (`adapters/`)**
  - [ ] Nyx adapter (`adapters/nyx.nox`): route and handler contract bridges.
  - [ ] Aether adapter (`adapters/aether.nox`): router and middleware contract bridges.
  - [ ] Verify zero core dependency on adapters (clean isolation).
- [ ] **6.3 Backend Compatibility Matrix (`tests/backend_matrix_test.nox`)**
  - [ ] Execute test suite under LLVM backend.
  - [ ] Execute test suite under QBE backend; document any scheduling divergence.
  - [ ] Mark unsupported target combinations explicitly in docs.
- [ ] **6.4 End-to-End Examples (`examples/`)**
  - [ ] `examples/property_example.nox`: data structure invariants.
  - [ ] `examples/fuzz_csv_example.nox`: robust CSV parser fuzzing.
  - [ ] `examples/contract_example.nox`: in-process API contract validation.
  - [ ] `examples/load_local_example.nox`: local handler load test scenario.
- [ ] **6.5 CLI Tool (`cli/main.nox`)**
  - [ ] Subcommand routing: `test`, `property`, `fuzz`, `replay`, `contract`, `load`, `report`, `version`.
  - [ ] Command line argument parsing and validation.
- [ ] **6.6 Release Candidate Hardening (v0.8.x)**
  - [ ] Extended test runs (10,000 property iterations, extended fuzzing).
  - [ ] Memory leak checks and performance benchmarking.
  - [ ] Update `CHANGELOG.md` for v0.7.0 and v0.8.x.

*Phase 6 Exit Gate:* Multi-module workflows verified; LLVM and QBE behaviors documented; clean adapter decoupling; CLI entry point working.

---

## Phase 7: v1.0.0 — Production Stabilization & Release
*Objective: Interface freeze, complete documentation, security audit, and formal v1.0 release.*

- [ ] **7.1 Interface Freeze & SemVer Contract**
  - [ ] Finalize public API signatures in `quartz.nox` and core modules.
  - [ ] Document semver stability guarantees.
- [ ] **7.2 Documentation Suite (`docs/`)**
  - [ ] `docs/architecture.md`: full system architecture and ADRs.
  - [ ] `docs/api.md`: complete public API reference.
  - [ ] `docs/recipes.md`: real-world testing patterns and recipes.
  - [ ] `docs/security.md`: security guardrails, target authorization, secret handling.
  - [ ] `docs/compatibility.md`: verified compiler/backend/OS matrix.
- [ ] **7.3 Release Packaging**
  - [ ] Verify `nox.lock` integrity and dependencies.
  - [ ] Verify package builds via `noxc build` and `noxc check`.
  - [ ] Run full release validation checklist against Section 12 quality gates.
  - [ ] Tag v1.0.0 and prepare release notes in `CHANGELOG.md`.

*Phase 7 Exit Gate:* 100% of quality gates pass; zero known P0/P1 defects; documentation complete; v1.0.0 release ready.

---

## Execution Log & Audit Trail

| Date | Phase | Task | Action / Command | Result / Evidence | Status |
|---|---|---|---|---|---|
| 2026-11-02 | Setup | Audit | `noxc --version` | `noxc 2.0.108` confirmed | [x] DONE |
| 2026-11-02 | Setup | Planning | Create `todo.md` | Comprehensive phased roadmap established | [x] DONE |
| 2026-11-02 | Phase 0 | Audit & Spikes | `noxc build --backend llvm/qbe` | Windows auto-fallback to QBE (without clang/zig on PATH) | [x] DONE |
| 2026-11-02 | Phase 0 | Spikes | Run probes 1 to 8 | Classes, callbacks, exceptions, concurrency, fs, json, process verified | [x] DONE |
| 2026-11-02 | Phase 0 | Discovery | Manifest parsing | `nox.json` requires `BinSpec` struct for `"bin"`; enables local imports | [x] DONE |
| 2026-11-02 | Phase 0 | Foundation | Create project layout | `core/`, `tests/`, `docs/`, `cli/`, `quartz.nox`, `nox.json`, `nox.lock` | [x] DONE |
| 2026-11-02 | Phase 0 | Baseline | `noxc test` | `tests/baseline_test.nox` passes; deliberate failure exits code 1 | [x] DONE |
| 2026-11-02 | Phase 0 | Docs | `docs/architecture.md` | ADR-001 to ADR-005 written with Windows/cross-platform specifics | [x] DONE |
| 2026-11-02 | Phase 1 | Core Types | `core/types.nox` | Statuses, `Failure`, `TestResult`, `RunSummary`, `ResourceLimits` | [x] DONE |
| 2026-11-02 | Phase 1 | Assertions | `core/assert.nox` | Built-in `AssertionError`, float tol, string contains, booleans | [x] DONE |
| 2026-11-02 | Phase 1 | TestSuite | `core/suite.nox` | Non-raising `check_*`, filtering, counting, deterministic runner | [x] DONE |
| 2026-11-02 | Phase 1 | Fixtures | `core/fixture.nox` | `try/finally` lifecycle, `with` context manager, cleanup verified | [x] DONE |
| 2026-11-02 | Phase 1 | Reports | `core/report.nox` | JUnit XML, JSON summary, secret redaction, atomic disk writes | [x] DONE |
| 2026-11-02 | Phase 1 | Verification | `noxc test` | `tests/core_test.nox` (25+ checks) and baseline passed cleanly | [x] DONE |
| 2026-11-02 | Phase 1 | Docs & Release | `docs/api.md`, `CHANGELOG.md` | Complete v0.1.0 public API reference and changelog updated | [x] DONE |
| 2026-11-02 | Phase 1 | Git & CI | `.github/workflows/ci.yml` | Multi-arch CI (Linux x64/arm64, macOS arm64, Windows x64) | [x] DONE |
| 2026-11-02 | Phase 1 | Registry Publish | `noxc publish` | `submitted: quartz (id=dac20c43-f400-4a74-a954-2d1b0f800fad)` | [x] DONE |
| 2026-11-02 | Phase 2 | PRNG | `core/prng.nox` | 64-bit Knuth LCG, FNV-1a derivation, golden vector sequence | [x] DONE |
| 2026-11-02 | Phase 2 | Generators | `property/generator.nox` | `IntGen` (boundary-biased), `FloatGen`, `BoolGen`, `StringGen`, `ListIntGen` | [x] DONE |
| 2026-11-02 | Phase 2 | Shrinker | `property/shrink.nox` | Minimal counterexamples (`shrink_int`, `shrink_str`, `shrink_list_int`) | [x] DONE |
| 2026-11-02 | Phase 2 | Runner & Replay | `property/property.nox`, `replay.nox` | `run_*_property`, `replay_*_case`, failure artifact JSON serialization | [x] DONE |
| 2026-11-02 | Phase 2 | Verification | `noxc test` | 5/5 test modules passed (1,000 cases deterministic invariant passed) | [x] DONE |
| 2026-11-02 | Phase 2 | Docs & Release | `docs/api.md`, `CHANGELOG.md` | Documented v0.2.0 API, tagged `v0.2.0`, ready for `noxc publish` | [x] DONE |
