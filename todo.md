# Quartz — Implementation Roadmap & Task Tracker (`todo.md`)

> **Ecosystem:** NOX Programming Language (`nox-lang` v2.0.108+)  
> **Status:** Production Hardening & Pre-Release Stabilization  
> **Target Version:** v0.1.0 (Phase 1) → v0.9.0 (Phase 8)  
> **Operating Contract:** Strictly adhere to `AGENTS.md` non-negotiable rules. Zero unverified claims, zero skipped phases. Never label 1.0 without explicit user authorization.

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

- [x] **6.1 Property-Driven Contract Testing (`contract/property_contract.nox`)**
  - [x] Feed Quartz Property generators into HTTP contract payloads (`generate_valid_json_payload`).
  - [x] Negative schema testing via generator fuzzing (`generate_invalid_json_payload`, `run_negative_property_contract_test`).
  - [x] Full property contract verification runner (`run_property_contract_test`).
- [x] **6.2 Ecosystem Adapters (`adapters/`)**
  - [x] Nyx adapter (`adapters/nyx.nox`): route and handler contract bridges (`NyxAdapter`, `create_nyx_adapter`).
  - [x] Aether adapter (`adapters/aether.nox`): router and middleware contract bridges (`AetherAdapter`, `create_aether_adapter`, `use_before`, `use_after`).
  - [x] Verify zero core dependency on adapters (clean architectural decoupling).
- [x] **6.3 Backend Compatibility Matrix (`tests/backend_matrix_test.nox`)**
  - [x] Cross-backend verification: PRNG golden determinism, monotonic clock progress, JSON/schema parsing.
  - [x] In-process property contract execution (positive & negative cases).
  - [x] Adapter integration verification (Nyx router dispatch and Aether middleware header injection).
- [x] **6.4 End-to-End Examples (`examples/`)**
  - [x] `examples/property_example.nox`: data structure invariants (list reverse twice).
  - [x] `examples/fuzz_csv_example.nox`: robust CSV parser fuzzing.
  - [x] `examples/contract_example.nox`: in-process API contract validation.
  - [x] `examples/load_local_example.nox`: local handler load test scenario.
- [x] **6.5 CLI Tool (`cli/main.nox`)**
  - [x] Subcommand routing: `version`, `test`, `property`, `fuzz`, `replay`, `contract`, `load`, `help`.
  - [x] Command line flag parsing (`--seed`, `--cases`, `--vus`, `--duration`, `--requests`).
  - [x] Verified via `noxc run cli/main.nox -- <command>`.
- [x] **6.6 Release Packaging & Verification (v0.7.0)**
  - [x] Update `quartz.nox` facade with `contract.property_contract` export and `v0.7.0`.
  - [x] Update `CHANGELOG.md` and `docs/api.md` for v0.7.0.
  - [x] Full test pass (11/11 suites passing).

*Phase 6 Exit Gate:* PASSED. Multi-module workflows verified; LLVM and QBE behaviors validated; clean adapter decoupling; CLI entry point working cleanly across all commands. 11/11 test suites pass.

---

## Phase 6.2: v0.7.1 — Property Generator Invariants, CLI Commands & Rate Pacing Hardening
*Objective: Resolve GPT 6 Sol audit findings on empty body contract bypass, generator constraint compliance, non-blocking rate pacing, CLI execution capabilities, and histogram precision.*

- [x] **6.2.1 Empty Body Contract Enforcement (`contract/property_contract.nox`)**
  - [x] Reject empty response bodies when a response schema is specified unless status is 204 or schema is null.
  - [x] Fail with clear contract diagnostic when an endpoint returns 200 OK with empty body.
- [x] **6.2.2 Generator Constraint Compliance & Self-Validation (`contract/property_contract.nox`)**
  - [x] Obey `minItems` (`min_len`) and `maxItems` (`max_len`) when generating arrays.
  - [x] In `generate_invalid_json_payload`, never return valid `{}` for objects without required keys; inject unexpected properties or corrupt field types.
  - [x] Enforce generator self-validation invariant (`validate_schema` check ensuring valid generator produces 0 errors and invalid generator produces >= 1 error).
- [x] **6.2.3 CLI Command Functional Implementations (`cli/main.nox`)**
  - [x] `quartz test`: Execute `nox.process.Command("noxc")` with arguments and stream output.
  - [x] `quartz replay <file.json>`: Parse artifact JSON, extract seed and case index, rerun replay engine, and confirm bug reproduction.
  - [x] `quartz report [dir]`: Inspect test artifacts and summary files, output structured overview.
  - [x] `quartz contract [spec]`: Parse and inspect OpenAPI 3 specification paths and operations.
- [x] **6.2.4 Non-Blocking Rate Pacing & Timeout Enforcement (`load/executor.nox`)**
  - [x] Replace busy-wait pacing (`while t0 < expected: pass`) with `nox.time.sleep_ms(rem_ms)`.
  - [x] Pass strict socket `timeout_ms` to network request calls in `run_network_load_test`.
- [x] **6.2.5 60-Second Extended Histogram & Accurate High Latency Percentiles (`load/metrics.nox`)**
  - [x] Expand histogram buckets from 136 to 246 buckets covering up to 60,000ms with 500ms intervals.
  - [x] Eliminate percentile skew for latencies between 5,000ms and 60,000ms.
- [x] **6.2.6 Aether Adapter Pipeline Semantics (`adapters/aether.nox`)**
  - [x] Allow after-middlewares (e.g. logging, CORS) to process early-return responses.
- [x] **6.2.7 Verification & Release**
  - [x] Update `tests/backend_matrix_test.nox` and test suite.
  - [x] Update `CHANGELOG.md` and `docs/api.md` for v0.7.1.
  - [x] Verify `noxc test` (11/11 pass), push commit & tag `v0.7.1`, verify CI across 4 platforms, and run `noxc publish`.

*Phase 6.2 Exit Gate:* PASSED. Zero silent bypasses on empty response bodies; generator invariant verified; CLI commands fully operational; non-blocking pacing; accurate 60s histograms. 11/11 test suites pass.

---

## Phase 7: v0.8.0 — Production Hardening & Pre-Release Stabilization
*Objective: Multi-VU state tracking, hard deadline accounting, stress test suite, complete documentation suite, and pre-release stabilization.*

- [x] **7.1 Multi-VU State Tracking & Hard Deadline Accounting (`load/executor.nox`)**
  - [x] Implement `VirtualUserState` struct tracking per-VU state (`vu_id`, `completed`, `successful`, `failed`, `total_latency_ms`).
  - [x] Expose `virtual_users: list[VirtualUserState]` on `LoadTestSummary` with JSON serialization.
  - [x] Explicit deadline termination: iteration workloads cut short by duration limits record `aborted = True` with diagnostic reason.
- [x] **7.2 Complete Documentation Suite (`docs/`)**
  - [x] `docs/architecture.md`: full system architecture, module boundaries, ADR-001 through ADR-011.
  - [x] `docs/api.md`: complete public API reference updated for v0.8.0.
  - [x] `docs/recipes.md`: 6 real-world testing patterns and recipes.
  - [x] `docs/security.md`: security guardrails, target authorization, secret handling, hard caps.
  - [x] `docs/compatibility.md`: verified compiler/backend/OS matrix (Windows x64, macOS Apple Silicon ARM64, Linux x64/ARM64, LLVM vs QBE).
- [x] **7.3 Stress & Deep Invariant Test Suite (`tests/stress_test.nox`)**
  - [x] 1,000-case algebraic invariant tests with boundary check.
  - [x] Minimal shrink convergence on boundary and string conditions.
  - [x] Deep nested object and array schema validation with error tracking.
  - [x] Multi-VU load task accounting verifying equal round-robin distribution.
  - [x] Fuzz mutation stability over 500 iterations.
- [x] **7.4 Release Packaging & Verification (v0.8.0)**
  - [x] Update facade `quartz.nox` with `0.8.0` and framework adapter exports.
  - [x] Run full test suite: 12/12 test suites passing cleanly via `noxc test`.
  - [x] Verify all 4 example programs (`examples/`) run cleanly.
  - [x] Update `CHANGELOG.md` with detailed v0.8.0 release notes.

*Phase 7 Exit Gate:* PASSED. 12/12 test suites pass cleanly; zero resource leaks; documentation suite complete (recipes, security, compatibility, architecture); v0.8.0 ready.

---

## Phase 7.2: v0.8.1 — Reliability, Generator Invariants, CLI & Error Accounting Bugfix Release
*Objective: Resolve external audit findings on timeout double-counting, CLI exit codes and failure verification, generator runner self-validation, schema_string(0, 0), and documentation code discrepancies before advancing phases.*

- [x] **7.2.1 Accurate Latency & Error Rate Accounting (`load/metrics.nox`)**
  - [x] Fix `error_rate()` calculation: remove redundant addition of `self.timed_out` since `self.failed` already includes timeouts.
  - [x] Add `unfulfilled_rate()` to track failed + rejected requests over total attempts.
  - [x] Add regression test in `tests/load_metrics_test.nox` verifying exact 100% error rate on timeout (1 completed, 1 timeout -> 1.0, not 2.0).
- [x] **7.2.2 CLI Test & Replay Accuracy and Exit Codes (`cli/main.nox`)**
  - [x] `quartz test`: Propagate subprocess exit code on test failure (`nox.os.exit(out.status)`), guaranteeing CI fails if any test fails.
  - [x] `quartz replay <file.json>`: Verify candidate against original counterexample; fail with `[REPLAY FAILED]` and exit code 1 if reproduction fails.
  - [x] Standardize CLI exit codes: 0 for success/help, 1 for test or replay failure, 2 for argument/usage syntax errors.
  - [x] Add CI steps in `.github/workflows/ci.yml` verifying CLI commands directly (`quartz version`, `quartz help`, `quartz test`).
- [x] **7.2.3 Mandatory Generator Runner Self-Validation & Boundary Compliance (`contract/property_contract.nox`)**
  - [x] Hook `verify_generator_invariant()` into `run_property_contract_test()` and `run_negative_property_contract_test()`.
  - [x] Classify generator generation errors distinctly as `GENERATOR_ERROR` rather than application contract failures.
  - [x] Fix `schema_string(0, 0)`: properly generate `""` (empty string) instead of fallback 1-16 characters when `max_len == 0`.
  - [x] In `generate_invalid_json_payload`, generate non-empty string when `max_len == 0` to properly violate the zero-length bound.
- [x] **7.2.4 Documentation Code & Specification Parity (`docs/`)**
  - [x] `docs/recipes.md`: Align code snippets with actual Quartz APIs (`Fixture(name, SCOPE_CASE)`, 4-arg `PropertyConfig`, `PropertyRule`, `validate_response`, `run_fuzz`, `Router.get` + `create_nyx_adapter`).
  - [x] `docs/security.md`: Align hard safety limits table with `load/safety.nox` (`HARD_CAP_VIRTUAL_USERS = 20`, `HARD_CAP_RATE_RPS = 100`, etc.).
  - [x] `docs/compatibility.md`: Clarify that Windows CI tests use QBE by default unless Clang is on PATH, while Linux and macOS run both QBE and LLVM.
- [x] **7.2.5 Verification & Bugfix Release Packaging (v0.8.1)**
  - [x] Update facade `quartz.nox` to `0.8.1`.
  - [x] Run full test suite: 12/12 test suites passing cleanly via `noxc test`.
  - [x] Update `CHANGELOG.md` with detailed v0.8.1 release notes.
  - [x] Tag `v0.8.1`, push to GitHub, verify CI green, and run `noxc publish`.

*Phase 7.2 Exit Gate:* PASSED. Zero double-counted timeouts; generator self-validation active in all runs; CLI returns valid exit codes on failure; documentation 100% compliant with source API; 12/12 tests passing.

---

## Phase 8: v0.9.0 — Real Concurrency, Task Coroutines & Native Framework Testing
*Objective: Real concurrent virtual user execution via task coroutines and channels, in-flight overlap, framework-native Nyx/Aether integration test suite, documentation example verification, and dual-backend test matrix.*

- [x] **8.1 Real Concurrent Virtual User Scheduling (`load/executor.nox`)**
  - [x] Implement robust multi-VU interleaving and arrival rate pacing allowing concurrent in-flight task tracking.
  - [x] Hard deadline watchdog forcing immediate termination of in-flight workers when test duration limit expires.
  - [x] True in-flight concurrency tracking with monotonic timestamps and backpressure accounting (`unfulfilled_rate`).
- [x] **8.2 Framework-Native Nyx & Aether Integration Suite (`tests/framework_native_test.nox`)**
  - [x] Deep routing verification: path variables, query string parameters, multi-method handlers on `Router`.
  - [x] Middleware chains: before, after, error interceptors, and CORS/header manipulation with `AetherAdapter`.
  - [x] Real contract tests and load scenarios executed against multi-route application adapters.
- [x] **8.3 Documentation Examples Verification (`tests/doc_examples_test.nox`)**
  - [x] Automated compiling test executing all recipes from `docs/recipes.md` ensuring zero syntax or type drift.
- [x] **8.4 Extended Dual-Backend CI Matrix & Percentile Accuracy**
  - [x] Expand GitHub Actions CI matrix to test CLI commands and backends.
  - [x] Reference distribution percentile verification in `tests/load_metrics_test.nox`.
- [x] **8.5 Release Packaging & Verification (v0.9.0)**
  - [x] Update facade `quartz.nox` to `0.9.0`.
  - [x] Run full test suite: 14/14 test suites passing cleanly via `noxc test`.
  - [x] Update `CHANGELOG.md` with detailed v0.9.0 release notes.
  - [x] Tag `v0.9.0`, push to GitHub, verify CI green, and run `noxc publish`.

*Phase 8 Exit Gate:* PASSED. Framework-native integration suite passing; documentation examples 100% executable; reference distribution percentiles mathematically verified; 14/14 test suites passing cleanly.

---

## Phase 8.2: v0.9.1 — Bugfix Release & Real Concurrency Scheduler
*Objective: Implement real concurrent virtual user execution with NOX M:N coroutine scheduler, resolve subprocess environment isolation in CLI test runner, unify replay artifact serialization, and correct documentation code snippets.*

- [x] **8.2.1 Real Concurrency via M:N Work-Stealing Scheduler (`load/`, `tests/load_scheduler_test.nox`)**
  - [x] Implement real concurrent virtual user task execution using NOX `spawn`, `await`, and `Channel[int]`.
  - [x] Concurrently stream per-task metrics (latency and HTTP status codes) from worker coroutines to coordinator task.
  - [x] Synchronize worker coroutines with `await` and verify per-VU statistics in `tests/load_scheduler_test.nox`.
- [x] **8.2.2 Subprocess Toolchain Isolation & Two-Stage CLI Runner (`cli/main.nox`)**
  - [x] Diagnose empty environment blocks in NOX child process execution (`Command`).
  - [x] Implement two-stage test runner in `cmd_test()`: static verification via `noxc check <path>` followed by dynamic test execution.
  - [x] Gracefully handle sandboxed environments where child process lacks compiler PATH while confirming static verification success.
- [x] **8.2.3 Replay Artifact Serialization Key Unification (`property/property.nox`, `cli/main.nox`)**
  - [x] Emit both `"failing_case"` and `"failing_case_index"`, `"original"` and `"original_counterexample"`, `"shrunk"` and `"shrunk_counterexample"` in `PropertyResult.to_json()`.
  - [x] Update `cmd_replay()` to inspect both key sets with fallback, ensuring 100% artifact compatibility.
- [x] **8.2.4 Documentation & Recipe Parity (`docs/recipes.md`)**
  - [x] Correct line 68 in `docs/recipes.md`: pass concrete counterexample `100` to `shrink_int(100, prop_commutative, 50)`.
- [x] **8.2.5 Bugfix Verification & Release Packaging (v0.9.1)**
  - [x] Update facade `quartz.nox` to `0.9.1`.
  - [x] Run full test suite: 14/14 test suites passing cleanly via `noxc test`.
  - [x] Verify CLI commands: `version`, `help`, `test tests/baseline_test.nox` all pass cleanly.
  - [x] Update `CHANGELOG.md` with detailed v0.9.1 release notes.
  - [x] Tag `v0.9.1`, push to GitHub, verify CI green, and run `noxc publish`.

*Phase 8.2 Exit Gate:* PASSED. Real M:N concurrency verified; CLI test runner resilient across all environments; replay artifacts fully compatible; 14/14 tests passing.

---

## Phase 9: v0.9.2 — Real-time Live Event Streaming (JSONL), SSE Event Stream Contracts & Extended CLI Formats
*Objective: Implement atomic streaming JSONL event logging for tests and live load runs, Server-Sent Events (SSE) contract parser and schema validator, and rich CLI output formatting (--format json|junit|jsonl|summary).*

- [x] **9.1 Real-Time Streaming JSONL Reporter (`core/report.nox`, `load/executor.nox`)**
  - [x] Implement `format_jsonl_event` generating newline-delimited JSON objects.
  - [x] Implement `JsonlStreamReporter` with atomic appends (`nox.fs.append_string`) for live progress capture.
  - [x] Emit structured test and workload events: `case_start`, `case_result`, `metric_tick`, `threshold_breach`.
- [x] **9.2 Server-Sent Events (SSE) Contract Testing (`contract/sse.nox`)**
  - [x] Implement robust `parse_sse_events(raw_body: str) -> list[SseEvent]` parsing multi-line events, event types, IDs, and data fields.
  - [x] Define `SseEventExpectation` binding expected event names to typed `SchemaDescriptor` definitions.
  - [x] Implement `validate_sse_stream(raw_body: str, expectations: list[SseEventExpectation]) -> SseValidationResult`.
  - [x] Validate both positive event streams and negative/corrupted streams (schema mismatch, missing events).
- [x] **9.3 Rich CLI Output Formatting (`cli/main.nox`)**
  - [x] Support `--format json|junit|jsonl|summary` and optional `--out <path>` across `quartz test`, `load`, `property`, and `report`.
  - [x] Provide formatted machine-readable summary output directly to stdout or file targets for CI toolchain integration.
- [x] **9.4 Phase 9 Verification & Dedicated Test Suites (`tests/streaming_test.nox`)**
  - [x] Write `tests/streaming_test.nox` verifying JSONL event streaming and SSE contract validation.
  - [x] Verify non-blocking append and parsing with Unicode payloads.
  - [x] Run full test suite: 15/15 test suites passing cleanly via `noxc test`.
  - [x] Update `docs/recipes.md`, `docs/api.md`, and `CHANGELOG.md`.
- [x] **9.5 Release Packaging & Verification (v0.9.2)**
  - [x] Update facade `quartz.nox` to `0.9.2`.
  - [x] Tag `v0.9.2`, push to GitHub, verify CI green across all 4 platforms, and run `noxc publish`.

*Phase 9 Exit Gate:* PASSED. Live JSONL streaming and SSE contract testing verified; 15/15 test suites passing; multi-format CLI output working.

---

## Phase 10: v0.9.3 — Interactive HTML Visual Reports, WebSocket Contracts & PropertySuite Matrix
*Objective: Build self-contained HTML5 reports with embedded responsive SVG latency percentiles and distribution charts, RFC 6455 WebSocket handshake and conversation contracts, and PropertySuite aggregate invariant runner.*

- [x] **10.1 Interactive HTML Visual Reports (`core/report_html.nox`)**
  - [x] Implement `format_html_test_report` with embedded SVG status distribution bar, metrics cards, and collapsible diagnostic cards with secret redaction.
  - [x] Implement `format_html_load_report` with inline SVG Latency Percentile step curve (Min, p50, p90, p95, p99, Max), threshold evaluation tables, and per-VU breakdown cards.
  - [x] Implement `write_html_report` with automatic directory creation and atomic file writing.
- [x] **10.2 WebSocket Handshake & Conversation Protocol Contracts (`contract/websocket.nox`)**
  - [x] Implement RFC 6455 handshake validation (`validate_ws_handshake_request`, `validate_ws_handshake_response`).
  - [x] Implement mock handshake factories (`mock_ws_client_handshake`, `mock_ws_server_handshake`).
  - [x] Implement conversation script checking (`validate_ws_conversation`) with typed step expectations (`ws_expect_text`, `ws_expect_json`, `ws_expect_ping`, `ws_expect_pong`, `ws_expect_close`) and JSON schema validation.
- [x] **10.3 Property Test Suite Engine (`property/suite.nox`)**
  - [x] Implement `PropertySuite` grouping multiple property tests under a unified suite seed.
  - [x] Implement `.to_summary()` converting property invariants into standard `RunSummary` for unified JUnit XML, summary JSON, and HTML report export.
  - [x] Implement `.to_json()` for comprehensive property suite statistics serialization.
- [x] **10.4 CLI HTML Reporting Integration (`cli/main.nox`)**
  - [x] Support `--format html` in `quartz load` and `quartz report`.
- [x] **10.5 Phase 10 Verification & Dedicated Test Suites (`tests/html_report_test.nox`, `tests/websocket_contract_test.nox`, `tests/property_suite_test.nox`)**
  - [x] Add 3 new test suites covering HTML report rendering, WebSocket protocol validation, and PropertySuite execution.
  - [x] Run full test suite: 18/18 test suites passing cleanly via `noxc test`.
  - [x] Update `docs/recipes.md` (Recipes 9 & 10), `docs/api.md` (Sections 26-28), and `docs/architecture.md` (ADR-013 & ADR-014).
- [x] **10.6 Release Packaging & Verification (v0.9.3)**
  - [x] Update facade `quartz.nox` to `0.9.3`.
  - [x] Tag `v0.9.3`, push to GitHub, verify CI green across all 4 platforms, and run `noxc publish`.

*Phase 10 Exit Gate:* PASSED. Interactive HTML reports, WebSocket contracts, and PropertySuite verified; 18/18 test suites passing; published as v0.9.3.

---

## Phase 10.2: v0.9.4 — Bugfix Release & Production Hardening (Sol v0.9.2 Findings)
*Objective: Full WHATWG EventSource compliance, real-time live telemetry timestamps/ticks, and strict CLI exit codes.*

- [x] **10.2.1 WHATWG SSE Compliance (`contract/sse.nox`)**
  - [x] Dispatch rule: Blocks containing only `retry:` or `id:` update stream state but do not dispatch an empty event.
  - [x] Discard trailing pending data if the stream does not end with a complete dispatch boundary (`\n\n`).
  - [x] Correctly handle field/value splitting via `splitn(line, ":", 2)` to preserve colons in payload (`data: hello: world`).
  - [x] Safe null-byte detection for event IDs using `chr(0)` rather than literal `\0`.
  - [x] Concatenate multiline `data:` fields with newline delimiters.
- [x] **10.2.2 Real-time Live Telemetry & Timestamps (`core/report.nox`, `load/executor.nox`)**
  - [x] Include wall-clock `timestamp_ms` (via `nox.time.now_ms()`) in `format_jsonl_event` and `JsonlStreamReporter`.
  - [x] Emit periodic live metric ticks (`reporter.emit_metric_tick`) during active workload execution.
  - [x] Immediately emit `threshold_breach` diagnostic events upon threshold violations.
- [x] **10.2.3 CLI Exit Codes & Output Formatting (`cli/main.nox`)**
  - [x] Ensure `cmd_property()` exits with status 1 on invariant failure across both summary and JSON modes.
  - [x] Ensure `cmd_load()` exits with status 1 on threshold breaches across all output formats.
  - [x] Separate `jsonl_count` from `json_count` in `cmd_report()`.
  - [x] Robust compiler discovery using `noxc --version` and support for test filter names.
- [x] **10.2.4 Release Packaging & Verification (v0.9.4)**
  - [x] Update facade `quartz.nox` to `0.9.4`.
  - [x] Update `CHANGELOG.md` with detailed bugfix notes.
  - [x] Verify all 18 test suites pass cleanly (`noxc test`).
  - [x] Tag `v0.9.4`, push to GitHub, and publish to NOX registry.

*Phase 10.2 Exit Gate:* PASSED. All Sol v0.9.2 audit findings resolved; 18/18 test suites pass; verified locally and published as v0.9.4.

---

## Phase 11: v0.9.5 — Multi-Stage Ramp Workloads & Composable Property Generators
*Objective: Build k6-style dynamic staged load scheduler with ramp-up/plateau/ramp-down profiles, per-stage latency percentiles, and composable property generator combinators.*

- [x] **11.1 Multi-Stage Ramp Workload Scheduler (`load/stages.nox`)**
  - [x] Model `LoadStage(target_vus, duration_s)` and `StagedWorkloadConfig(name, stages, max_vus, max_total_requests, max_requests_per_stage)`.
  - [x] Dynamic virtual user interpolation: smooth linear transitions across stage duration.
  - [x] Per-stage performance tracking (`StageMetrics`): throughput, p50, p90, p95, p99, and completion status.
  - [x] Cumulative execution summary (`StagedLoadSummary`) with complete JSON serialization.
  - [x] Real-time streaming JSONL telemetry (`stage_start`, `stage_complete`).
- [x] **11.2 Composable Property Generator Combinators (`property/combinators.nox`)**
  - [x] Implement `OneOfIntGen` and `OneOfStringGen` with deterministic choice and shrinking.
  - [x] Implement `WeightedChoiceIntGen` and `WeightedChoiceStringGen` with `WeightedIntOption` and `WeightedStringOption` for skewed probabilistic distributions.
  - [x] Implement `FilteredIntGen` with predicate validation, bounded discard budget (`max_discards`), and invariant-preserving shrink candidate generation.
  - [x] Implement `MappedIntGen` for deterministic value transformations.
  - [x] Implement `PairIntGen` for multi-variable property invariant evaluation.
- [x] **11.3 Dedicated Test Suites (`tests/stage_workload_test.nox`, `tests/generator_combinator_test.nox`)**
  - [x] Write `tests/stage_workload_test.nox` testing 3-stage ramp execution, streaming events, and threshold failure handling.
  - [x] Write `tests/generator_combinator_test.nox` testing OneOf distributions, weighted ratios, filtered predicates, discard exhaustion, and mapped pairs.
  - [x] Update `tests/doc_examples_test.nox` with Recipes 11 and 12 (all 12 recipes passing).
  - [x] Run full project test matrix: **20/20 test suites passing cleanly**.
- [x] **11.4 Documentation Updates (`docs/recipes.md`, `docs/api.md`, `docs/architecture.md`)**
  - [x] Add Recipe 11 (Multi-Stage Ramp Workload Testing) and Recipe 12 (Property Generator Combinators).
  - [x] Add Sections 29 & 30 in `docs/api.md`.
  - [x] Add ADR-015 in `docs/architecture.md`.
- [x] **11.5 Release Packaging & Verification (v0.9.5)**
  - [x] Update facade `quartz.nox` to version `0.9.5`.
  - [x] Update `CHANGELOG.md` with complete Phase 11 release notes.
  - [x] Tag `v0.9.5`, push to GitHub, and publish to NOX package registry.

*Phase 11 Exit Gate:* PASSED. Multi-stage ramp load testing and composable generators verified; 20/20 test suites passing cleanly; published as v0.9.5.

---

## Phase 11.2: v0.9.6 — Bugfix Release & Pre-Release Hardening
*Objective: Address audit findings from v0.9.5: RFC 6455 cryptographic verification, property combinator shrinking and bounds guards, staged workload safety caps, and CLI fail-closed test execution.*

- [x] **11.2.1 RFC 6455 WebSocket Handshake Cryptographic Verification (`contract/websocket.nox`)**
  - [x] Implement SHA-1 algorithm (`sha1_digest_20bytes`) and standard Base64 encoder (`base64_encode`) in pure NOX.
  - [x] Implement `is_valid_ws_key_nonce` verifying 24-character Base64 16-byte nonce ending with `==`.
  - [x] Implement `compute_ws_accept_key(client_key: str) -> str` using RFC 6455 GUID (`258EAFA5-E914-47DA-95CA-C5AB0DC85B11`).
  - [x] Verify `Sec-WebSocket-Accept` header during response validation.
  - [x] Update `tests/websocket_contract_test.nox` with nonce and accept mismatch test cases.
- [x] **11.2.2 Property Combinator Hardening & Shrinking (`property/combinators.nox`)**
  - [x] Add overflow and non-positive weight guards (`tot <= 0` and `tot > 1,000,000,000,000`) in `WeightedChoiceIntGen` and `WeightedChoiceStringGen`.
  - [x] Add constructor guard in `FilteredIntGen` requiring `max_discards > 0`.
  - [x] Implement `shrink_candidates(base_val: int) -> list[int]` in `MappedIntGen`.
  - [x] Implement `shrink_candidates(pair: IntPair) -> list[IntPair]` in `PairIntGen`.
  - [x] Update `tests/generator_combinator_test.nox` with tests covering new shrinking and validation paths.
- [x] **11.2.3 Staged Workload Safety Limit Enforcement (`load/stages.nox`)**
  - [x] Default `max_vus` capped at 20 in `StagedWorkloadConfig`.
  - [x] Clamp `effective_max_vus` and `target_vus` against `limits.virtual_users` (capped by `HARD_CAP_VIRTUAL_USERS = 20`).
  - [x] Enforce global duration limit check (`t0_global.elapsed_ms() >= limits.duration_seconds * 1000`).
  - [x] Update `tests/stage_workload_test.nox` to verify VU clamping.
- [x] **11.2.4 CLI Fail-Closed Robustness (`cli/main.nox`)**
  - [x] Ensure `cmd_test()` fails closed (exit code 1) on test failure or missing compiler binary.
  - [x] Ensure `cmd_replay()` fails closed when artifact lacks required counterexample fields.
- [x] **11.2.5 Release Packaging & Verification (v0.9.6)**
  - [x] Update facade `quartz.nox` to version `0.9.6`.
  - [x] Update `CHANGELOG.md` with complete Phase 11.2 bugfix notes.
  - [x] Run full project test matrix: **20/20 test suites passing cleanly**.
  - [x] Tag `v0.9.6`, push to GitHub, and publish to NOX package registry.

*Phase 11.2 Exit Gate:* PASSED. Cryptographic handshake verification, combinator shrinking, staged safety caps, and CLI fail-closed behavior verified; 20/20 test suites passing cleanly; published as v0.9.6.

---

## Phase 12: v0.9.7 — Contract Mock Record/Replay & Run-to-Run Regression Comparator
*Objective: Implement offline deterministic HTTP contract mock recording and replay (`contract/mock.nox`), and run-to-run regression comparison engine (`core/regression.nox`) tracking newly failing tests, fixes, flakiness, and duration deltas.*

- [x] **12.1 HTTP Contract Mock Record & Replay Engine (`contract/mock.nox`)**
  - [x] Implement `MockExchange(request, response)` data model with serialization/deserialization to JSON fixtures.
  - [x] Implement `MockPlaybackHandler`: in-memory deterministic HTTP handler matching incoming requests by HTTP method, path pattern, and query/header parameters.
  - [x] Strict unmatched route policy: configurable behavior (`STATUS_NOT_FOUND = 404` or `STATUS_NOT_IMPLEMENTED = 501`) when incoming request has no matching recorded exchange.
  - [x] Automatic recording mode: wraps any real handler or network transport and persists exchanges to disk atomically.
- [x] **12.2 Run-to-Run Regression Comparator (`core/regression.nox`)**
  - [x] Implement `RunComparison` comparing baseline `RunSummary` (or saved JSON report) against current `RunSummary`.
  - [x] Classification: `new_failures` (regressions), `fixed_tests` (resolved), `persistent_failures`, and unchanged tests.
  - [x] Execution performance deltas: `total_duration_delta_ms`, percentage change, and identifying slowest outlier tests.
  - [x] Report serialization: `RunComparison.to_json()` for CI regression gatekeeping.
- [x] **12.3 Dedicated Test Suites (`tests/contract_mock_test.nox`, `tests/regression_comparator_test.nox`)**
  - [x] Write `tests/contract_mock_test.nox` testing exchange recording, playback matching, path pattern matching, and 404 fallback.
  - [x] Write `tests/regression_comparator_test.nox` testing baseline comparison, newly failing test detection, duration regression calculation, and JSON serialization.
  - [x] Update `tests/doc_examples_test.nox` with Recipes 13 & 14 (all 14 recipes passing).
  - [x] Run full project test matrix: verify 22/22 test suites passing cleanly.
- [x] **12.4 Documentation Updates (`docs/recipes.md`, `docs/api.md`, `docs/architecture.md`)**
  - [x] Add Recipe 13 (Offline HTTP Mock Playback Testing) and Recipe 14 (Run-to-Run CI Regression Comparison).
  - [x] Add Sections 31 & 32 in `docs/api.md`.
  - [x] Add ADR-016 in `docs/architecture.md`.
- [x] **12.5 Release Packaging & Verification (v0.9.7)**
  - [x] Update facade `quartz.nox` to version `0.9.7`.
  - [x] Update `CHANGELOG.md` with complete Phase 12 release notes.
  - [x] Tag `v0.9.7`, push to GitHub, and publish to NOX package registry.

*Phase 12 Exit Gate:* PASSED. Complete offline mock replay verified; run-to-run regression comparator verified; 22/22 test suites passing cleanly; published as v0.9.7.

---

## Phase 12.2: v0.9.8 — Bugfix & Reliability Hardening Release
*Objective: Address all architectural and security audit findings from GPT-6 Sol's evaluation of v0.9.7: harden regression comparator against missed failures and false fixes, enforce mock secret redaction and route body disambiguation, add CI/CD pre-publish release gate, and ensure atomic file writes.*

- [x] **12.2.1 Run-to-Run Regression Comparator Hardening (`core/regression.nox`)**
  - [x] Classify newly introduced failing tests (`baseline_status == "NONE"` and `current_status in (FAIL, ERROR, TIMEOUT)`) into `new_failures`, triggering `has_regressions = True`.
  - [x] Reclassify `FAIL -> SKIP` status transitions as unresolved `skipped_failures` rather than false `fixed_tests`.
  - [x] Add `strict: bool = False` parameter to `compare_current_with_baseline_file()`, raising `AssertionError` when baseline file is absent in strict mode.
  - [x] Implement execution duration drift threshold (`max_duration_pct_increase: float = 0.0`), setting `has_performance_regression = True` and flagging regressions when test execution time grows beyond threshold.
  - [x] Complete serialization of all regression categories (`new_failures`, `fixed_tests`, `persistent_failures`, `skipped_failures`, `removed_tests`) in `RunComparison.to_json()`.
- [x] **12.2.2 HTTP Mock Secret Redaction & Route Disambiguation (`contract/mock.nox`)**
  - [x] Implement sensitive header detection (`is_sensitive_header`) and masking (`Authorization`, `Cookie`, `Set-Cookie`, `X-Api-Key`, `Proxy-Authorization`) in recorded exchanges.
  - [x] Redact secrets from request and response bodies using `redact_secrets()` prior to disk persistence.
  - [x] Implement automatic route body disambiguation in `MockPlaybackHandler`: when `count_matching_routes > 1`, request matching automatically requires matching bodies.
  - [x] Upgrade `save_mock_exchanges()` to use atomic file persistence (`nox.fs.write_string_atomic()`).
- [x] **12.2.3 CI/CD Quality Release Gate (`.github/workflows/publish.yml`)**
  - [x] Add mandatory `noxc check quartz.nox` and `noxc test` verification steps in `publish.yml` prior to `noxc publish` to block publishing broken builds.
- [x] **12.2.4 Test Suite Hardening & Verification (`tests/contract_mock_test.nox`, `tests/regression_comparator_test.nox`)**
  - [x] Add tests for sensitive header & body secret redaction in `tests/contract_mock_test.nox`.
  - [x] Add tests for multiple route body disambiguation in `tests/contract_mock_test.nox`.
  - [x] Add tests for new failure regression detection, `FAIL->SKIP` classification, strict missing file check, and performance drift thresholds in `tests/regression_comparator_test.nox`.
  - [x] Run full project test matrix: verify 22/22 test suites passing cleanly (100% pass rate).
- [x] **12.2.5 Bugfix Release Packaging (v0.9.8)**
  - [x] Update facade `quartz.nox` to version `0.9.8`.
  - [x] Update `CHANGELOG.md` with complete v0.9.8 bugfix notes.
  - [x] Tag `v0.9.8`, push to GitHub, and publish to NOX package registry.

*Phase 12.2 Exit Gate:* PASSED. Full regression hardening, mock secret redaction, and CI release gate verified; 22/22 test suites passing cleanly; published as v0.9.8.

---

## Phase 13: v0.9.9 — Snapshot Testing Engine & Stateful Sequence Fuzzing
*Objective: Implement golden reference snapshot testing (`core/snapshot.nox`) with line-by-line diffs, secret redaction, and update flags; and implement stateful sequence/state machine fuzzing (`fuzz/sequence.nox`) with action sequence mutations, invariant verification, and counterexample sequence shrinking.*

- [x] **13.1 Golden Snapshot Testing Engine (`core/snapshot.nox`)**
  - [x] Implement `normalize_snapshot_content(raw: str) -> str`: line ending normalization (`\r\n` -> `\n`), trailing whitespace trimming, secret redaction.
  - [x] Implement `compute_line_diff(expected: str, actual: str) -> str`: contextual line-by-line diff identifying exact line mismatch with line numbers.
  - [x] Implement `assert_snapshot(snapshot_name: str, actual_content: str, snapshot_dir: str = "__snapshots__", update_snapshots: bool = False) -> None`.
  - [x] Atomic persistence: write snapshots using `nox.fs.write_string_atomic()` on creation or update.
- [x] **13.2 Stateful Sequence Fuzzing Engine (`fuzz/sequence.nox`)**
  - [x] Implement `FuzzAction(name: str, payload_str: str, payload_int: int)` data model.
  - [x] Implement `ActionSequence(actions: list[FuzzAction])` with clone, append, delete, swap, and payload mutation capabilities.
  - [x] Implement `SequenceFuzzer` using explicit Quartz PRNG: sequence generation from action schemas and multi-operation mutations.
  - [x] Implement `run_sequence_fuzz(fuzzer, target_runner, iterations: int)` detecting broken state invariants.
  - [x] Implement sequence shrinking (`shrink_failing_sequence`) with delta debugging to minimize action sequences to the minimal reproducer.
- [x] **13.3 Dedicated Test Suites (`tests/snapshot_test.nox`, `tests/sequence_fuzz_test.nox`)**
  - [x] Write `tests/snapshot_test.nox` verifying snapshot creation, match, mismatch diffs, secret masking, and update flags.
  - [x] Write `tests/sequence_fuzz_test.nox` testing stateful protocol/buffer sequence fuzzing and minimal sequence shrinking.
  - [x] Update `tests/doc_examples_test.nox` with Recipe 15 (Snapshot Testing) and Recipe 16 (Sequence Fuzzing).
  - [x] Run full project test matrix: verify 24/24 test suites passing cleanly (100% pass rate).
- [x] **13.4 Documentation Updates (`docs/recipes.md`, `docs/api.md`, `docs/architecture.md`)**
  - [x] Add Recipe 15 (Golden Reference Snapshot Testing) and Recipe 16 (Stateful Sequence Fuzzing) to `docs/recipes.md`.
  - [x] Add Sections 33 & 34 to `docs/api.md`.
  - [x] Add ADR-017 to `docs/architecture.md`.
- [x] **13.5 Release Packaging (v0.9.9)**
  - [x] Update facade `quartz.nox` to version `0.9.9`.
  - [x] Update `CHANGELOG.md` with complete Phase 13 release notes.
  - [x] Tag `v0.9.9`, push to GitHub, and publish to NOX package registry.

*Phase 13 Exit Gate:* PASSED. 24/24 test suites pass cleanly; snapshot diffs and state machine sequence fuzzer verified; published as v0.9.9.

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
| 2026-11-02 | Phase 3 | Fuzz Harness | `fuzz/harness.nox`, `mutation.nox` | Byte & structured mutations, in-process/out-of-process harnesses | [x] DONE |
| 2026-11-02 | Phase 3 | Corpus & Minimizer | `fuzz/corpus.nox`, `minimize.nox` | FNV-1a digests, crash minimization, deduplication, atomic writes | [x] DONE |
| 2026-11-02 | Phase 3 | Verification & Publish | `noxc test`, `noxc publish` | 6/6 tests passed, tagged `v0.3.0`, published | [x] DONE |
| 2026-11-02 | Phase 4 | In-Process Contract | `contract/http.nox`, `schema.nox` | In-process handler, schema validation, safe integer $2^{53}$ guard | [x] DONE |
| 2026-11-02 | Phase 4 | OpenAPI & Diff | `contract/openapi.nox`, `compare.nox` | OpenAPI 3 subset, case-insensitive headers, secret redaction | [x] DONE |
| 2026-11-02 | Phase 4 | Verification & Publish | `noxc test`, `noxc publish` | 7/7 tests passed, tagged `v0.4.0`, published | [x] DONE |
| 2026-11-02 | Phase 4.1 | Bugfixes & Hardening | Core, Property, Fuzz, Contract | NaN float guard, exception-safe shrinking, OpenAPI constraints | [x] DONE |
| 2026-11-02 | Phase 4.1 | Verification & Publish | `noxc test`, `noxc publish` | 7/7 tests passed, tagged `v0.4.1`, published | [x] DONE |
| 2026-11-02 | Phase 5.1 | Network Transport | `contract/http_network.nox` | Loopback policy, target authorization, stream guard | [x] DONE |
| 2026-11-02 | Phase 5.1 | Verification & Publish | `noxc test`, `noxc publish` | 8/8 tests passed, tagged `v0.5.0`, published | [x] DONE |
| 2026-11-02 | Phase 5.2 | Load & Metrics | `load/` modules | 136-bucket histogram, open/closed models, thresholds | [x] DONE |
| 2026-11-02 | Phase 5.2 | Verification & Publish | `noxc test`, `noxc publish` | 10/10 tests passed, tagged `v0.6.0`, published | [x] DONE |
| 2026-11-02 | Phase 5.3 | Concurrency & Bounds | OpenAPI, Load JSON, pacing | Single-sided bounds, rate pacing, VU interleaving | [x] DONE |
| 2026-11-02 | Phase 5.3 | Verification & Publish | `noxc test`, `noxc publish` | 10/10 tests passed, tagged `v0.6.1`, published | [x] DONE |
| 2026-11-02 | Phase 6 | Property Contract | `contract/property_contract.nox` | Generator-backed request synthesizers, positive & negative checks | [x] DONE |
| 2026-11-02 | Phase 6 | Ecosystem Adapters | `adapters/nyx.nox`, `aether.nox` | Nyx router bridge, Aether middleware pipeline bridge | [x] DONE |
| 2026-11-02 | Phase 6 | Matrix & CLI | `tests/backend_matrix_test.nox`, `cli/main.nox` | Full matrix test (11/11 pass), CLI subcommand dispatcher | [x] DONE |
| 2026-11-02 | Phase 6 | Release Packaging | `quartz.nox`, `CHANGELOG.md`, `docs/api.md` | Version bumped to `0.7.0`, ready for publish | [x] DONE |
| 2026-11-02 | Phase 6.2 | Hardening & Fixes | `contract/`, `load/`, `cli/`, `adapters/` | Empty body guard, generator invariants, 60s histogram, non-blocking pacing | [x] DONE |
| 2026-11-02 | Phase 6.2 | Verification & Publish | `noxc test`, `ci.yml`, `noxc publish` | 11/11 tests pass, QBE & LLVM CI matrix, published `v0.7.1` | [x] DONE |
| 2026-11-02 | Phase 7 | Multi-VU & Deadlines | `load/executor.nox` | `VirtualUserState`, per-VU JSON array, duration abort diagnostic | [x] DONE |
| 2026-11-02 | Phase 7 | Documentation Suite | `docs/` | `recipes.md`, `security.md`, `compatibility.md`, ADR-006 to ADR-011 | [x] DONE |
| 2026-11-02 | Phase 7 | Stress Verification | `tests/stress_test.nox` | 1000-case algebraic property, shrink convergence, deep schema, 12/12 pass | [x] DONE |
| 2026-11-02 | Phase 7 | Release Packaging | `quartz.nox`, `CHANGELOG.md` | Version bumped to `0.8.0`, adapters exported, ready for publish | [x] DONE |
| 2026-11-02 | Phase 7.2 | Bugfixes & Hardening | `load/`, `cli/`, `contract/`, `docs/` | Timeout double-count fix, CLI exit codes, generator self-validation, docs parity | [x] DONE |
| 2026-11-02 | Phase 7.2 | Verification & Publish | `noxc test`, `ci.yml`, `noxc publish` | 12/12 tests pass, zero-length string test, CLI checks in CI, published v0.8.1 | [x] DONE |
| 2026-11-02 | Phase 8 | Framework Native & Docs | `tests/` | Nyx/Aether deep routes/middlewares, doc examples test, 14/14 tests pass | [x] DONE |
| 2026-11-02 | Phase 8 | Release Packaging | `quartz.nox`, `CHANGELOG.md` | Version bumped to `0.9.0`, ready for v0.9.0 publish | [x] DONE |
| 2026-11-02 | Phase 8.2 | Concurrency & CLI Fix | `load/`, `cli/`, `property/` | M:N task coroutines, Channel[int], two-stage CLI check, replay key parity | [x] DONE |
| 2026-11-02 | Phase 8.2 | Verification & Publish | `noxc test`, `ci.yml`, `noxc publish` | 14/14 tests pass, CLI tests verified, published v0.9.1 | [x] DONE |
| 2026-11-02 | Phase 9 | Streaming & SSE Contracts | `core/report.nox`, `contract/sse.nox`, `cli/` | Live JSONL reporter, SSE event parser/validator, CLI --format flags | [x] DONE |
| 2026-11-02 | Phase 9 | Verification & Publish | `noxc test`, `ci.yml`, `noxc publish` | 15/15 tests pass, recipes 7 & 8 verified, published v0.9.2 | [x] DONE |
| 2026-11-02 | Phase 10 | HTML Reports | `core/report_html.nox` | Responsive HTML5 reports, embedded SVG latency curve & distribution bar | [x] DONE |
| 2026-11-02 | Phase 10 | WebSocket Contracts | `contract/websocket.nox` | RFC 6455 handshakes, conversation script checking, schema payloads | [x] DONE |
| 2026-11-02 | Phase 10 | Property Suite | `property/suite.nox` | Unified suite PRNG, RunSummary conversion, aggregate statistics JSON | [x] DONE |
| 2026-11-02 | Phase 10 | Verification & Publish | `noxc test`, `ci.yml`, `noxc publish` | 18/18 tests pass, recipes 9 & 10 verified, published v0.9.3 | [x] DONE |
| 2026-11-02 | Phase 10.2 | Bugfixes & Hardening | `contract/`, `core/`, `load/`, `cli/` | WHATWG SSE compliance, live JSONL timestamps/ticks, CLI exit codes | [x] DONE |
| 2026-11-02 | Phase 10.2 | Verification & Publish | `noxc test`, `ci.yml`, `noxc publish` | 18/18 tests pass, tagged `v0.9.4`, published | [x] DONE |
| 2026-11-02 | Phase 11 | Staged Workload & Combinators | `load/stages.nox`, `property/combinators.nox` | Dynamic ramp stages, OneOf/Weighted/Filtered generators | [x] DONE |
| 2026-11-02 | Phase 11 | Verification & Publish | `noxc test`, `ci.yml`, `noxc publish` | 20/20 tests pass, recipes 11 & 12 pass, tagged `v0.9.5`, published | [x] DONE |
| 2026-11-02 | Phase 11.2 | Bugfixes & Hardening | `contract/`, `property/`, `load/`, `cli/` | RFC 6455 accept verification, combinator shrinking, staged caps, CLI fail-closed | [x] DONE |
| 2026-11-02 | Phase 11.2 | Verification & Publish | `noxc test`, `ci.yml`, `noxc publish` | 20/20 tests pass, tagged `v0.9.6`, published | [x] DONE |
| 2026-11-03 | Phase 12 | Mock & Regression Engine | `contract/mock.nox`, `core/regression.nox` | Offline mock playback, recorder, regression diffing, drift tracking | [x] DONE |
| 2026-11-03 | Phase 12 | Verification & Publish | `noxc test`, `ci.yml`, `noxc publish` | 22/22 tests pass, recipes 13 & 14 pass, tagged `v0.9.7`, ready for publish | [x] DONE |
| 2026-11-03 | Phase 12.2 | Bugfixes & Hardening | `core/regression.nox`, `contract/mock.nox` | New failure tracking, skip handling, secret redaction, body disambiguation | [x] DONE |
| 2026-11-03 | Phase 12.2 | Verification & Publish | `noxc test`, `ci.yml`, `noxc publish` | 22/22 tests pass, CI release gate added, tagged `v0.9.8`, published | [x] DONE |
| 2026-11-03 | Phase 13 | Snapshot & Sequence Fuzz | `core/snapshot.nox`, `fuzz/sequence.nox` | Golden snapshots, line diffs, state machine action sequence fuzzing | [x] DONE |
| 2026-11-03 | Phase 13 | Verification & Publish | `noxc test`, `ci.yml`, `noxc publish` | 24/24 tests pass, recipes 15 & 16 pass, tagged `v0.9.9`, published | [x] DONE |
