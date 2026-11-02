# Changelog

All notable changes to the Quartz test framework will be documented in this file.
This project adheres to [Semantic Versioning](https://semver.org/).

## [0.4.1] - 2026-11-02 - Phase 4.1: Reliability, Correctness & Hardening Bugfix Release

### Fixed & Hardened
- **Core Assertions (`core/assert.nox`)**: Added IEEE 754 NaN guard to `assert_eq_float` preventing false passes when comparing NaN values.
- **Property JSON Diagnostics (`property/property.nox`)**: Sanitized counterexample strings and failure messages using `escape_json` in `PropertyResult.to_json()`.
- **Shrinking Resilience (`property/shrink.nox`)**: Wrapped predicate evaluations in exception-safe handlers (`safe_eval_int`, `safe_eval_str`, `safe_eval_list_int`), preventing unexpected target crashes from interrupting candidate shrinking.
- **Fuzz Exception Classification (`fuzz/harness.nox`)**: Replaced loose keyword checks with dedicated `HandledRejection` exception class and strict prefix matching (`reject:`, `validation_error:`, `bad_input:`) to eliminate false-negative crash classifications.
- **Corpus Resource Limits (`fuzz/corpus.nox`)**: Enforced `max_files` and `max_total_bytes` in `CorpusManager.add_seed()`, added crash artifact limit enforcement in `save_crash_artifact()`, and escaped artifact JSON fields.
- **Fuzz CI Gating & Session Timeout (`fuzz/fuzzer.nox`)**: Added `fail_on_bug: bool` parameter to `run_fuzz()` to fail test suites when bugs are found, and added `timeout_ms` session deadlines.
- **OpenAPI Schema Constraints (`contract/openapi.nox`)**: Implemented parsing and propagation for `minLength`, `maxLength`, `minimum`, `maximum`, `minItems`, and `maxItems`. Unsupported keywords now raise explicit `UnsupportedContractFeature` errors instead of being silently skipped.
- **OpenAPI Defaults (`contract/openapi.nox`)**: Aligned `additionalProperties` default with OpenAPI 3 spec (permissive by default; strict only when explicitly declared `additionalProperties: false`). Added support for `2XX`..`5XX` range codes and `default` response status.
- **Integer Precision Guard (`contract/schema.nox`)**: Adjusted precision guard check to `>= SAFE_INT_MAX` and `<= SAFE_INT_MIN` ensuring rounded boundary numbers at $2^{53}$ are detected.
- **Content-Type Normalization (`contract/compare.nox`)**: Normalized `Content-Type` media-type headers by stripping trailing parameters (e.g. `charset=utf-8`) before matching against expected media types.
- **Test Suites (`tests/`)**: Added comprehensive regression tests in `core_test.nox`, `fuzz_test.nox`, and `contract_handler_test.nox`.

## [0.4.0] - 2026-11-02 - Phase 4: In-Process HTTP Contract Testing & Schema Validation

### Added
- **Transport Abstraction (`contract/http.nox`)**: In-process HTTP handler executor (`execute_handler`) with nanosecond timing, `build_request`, `build_response`, and strict prohibition of default external network connections.
- **Contract & Operation Descriptors (`contract/types.nox`)**: `ContractCheckResult`, case-insensitive HTTP header lookup/verification (`get_header_ci`, `has_header_ci`), and secret redaction for sensitive auth tokens and cookies (`redact_header_value`).
- **Schema Evaluation Engine (`contract/schema.nox`)**: Typed JSON schema validator for `null`, `boolean`, `string`, `integer` (with $2^{53}$ precision guard), `number`, `array`, `object`, and `enum`. Configurable strict vs tolerant unknown property policies.
- **Response Verification (`contract/compare.nox`)**: Validation of status codes, required response headers, Content-Type media-type/charset normalization, and response body JSON schema checks (`validate_response`).
- **OpenAPI Subset Importer (`contract/openapi.nox`)**: Importer for JSON OpenAPI 3.x subset (`parse_openapi_json`) extracting operations and response schemas, raising explicit `UnsupportedContractFeature` errors on unhandled keywords.
- **Test Case Runner (`contract/runner.nox`)**: Declarative test case builder (`ContractTestCase`) executing directly against application handler callbacks.
- **Example (`examples/contract_example.nox`)**: Runnable example demonstrating in-process endpoint contract verification.
- **Test Suite (`tests/contract_handler_test.nox`)**: Comprehensive test suite covering positive handler verification, negative status codes, missing headers, schema violations, $2^{53}$ integer precision guards, header case-insensitivity, and OpenAPI parsing.

## [0.3.0] - 2026-11-02 - Phase 3: Fuzz Engine & Corpus Management

### Added
- **Fuzz Target Harness (`fuzz/harness.nox`)**: `FuzzOutcome` descriptor, execution status classification (`NORMAL`, `HANDLED_REJECTION`, `ASSERTION_VIOLATION`, `TIMEOUT`, `CRASH`), in-process target harness (`run_in_process`) with monotonic nanosecond timing, and out-of-process harness (`run_out_of_process`) with bounded process timeouts.
- **Mutation Engine (`fuzz/mutation.nox`)**: Bounded bit flips, byte flips, insertions, deletions, duplications, byte swaps, splicing, boundary integer/float injections (`INT64_MIN`, `INT64_MAX`, `2^53`, `NaN`, `Infinity`), delimiter injections, and structure-aware mutations for JSON and CSV.
- **Corpus Management (`fuzz/corpus.nox`)**: Deterministic directory hierarchy (`corpus/<target>/{seeds, crashes, timeouts, regressions}`), content-derived FNV-1a hexadecimal stable IDs, atomic seed/crash writing, and hard resource bounds (`CorpusConfig`).
- **Crash Minimization (`fuzz/minimize.nox`)**: Bounded delta-debugging crash minimizer preserving the original failure signature (`minimize_crash`).
- **Fuzzer Driver (`fuzz/fuzzer.nox`)**: Iterative bounded fuzz loop (`run_fuzz`), automatic crash capture, deduplication by signature, crash minimization, and summary statistics (`FuzzSummary`).
- **Example (`examples/fuzz_csv_example.nox`)**: Runnable example demonstrating CSV parser fuzzing with custom seeds.
- **Test Suite (`tests/fuzz_test.nox`)**: Comprehensive test suite verifying target classification, mutation determinism and length bounds, corpus lifecycle, delta-debugging crash minimization down to exact crash tokens, and parser fuzzing sessions.


## [0.2.0] - 2026-11-02 - Phase 2: Property Engine & Deterministic Replay

### Added
- **Explicit-State PRNG (`core/prng.nox`)**: Knuth MMIX 64-bit LCG algorithm, FNV-1a seed mixing routine `derive_case_seed(root_seed, test_id, case_index, rng_version)`, range generation, and golden vector reproducibility.
- **Typed Generators (`property/generator.nox`)**: `IntGen` with boundary bias (`INT_MIN`, `INT_MAX`, `0`, `1`, `-1`), `FloatGen`, `BoolGen`, `StringGen` (with ASCII, Alphanumeric, and Turkish Unicode sets), `ByteGen`, and `ListIntGen`.
- **Shrinking Engine (`property/shrink.nox`)**: Bounded minimal counterexample search preserving invariant failure (`shrink_int`, `shrink_str`, `shrink_list_int`).
- **Property Runner (`property/property.nox`)**: `run_int_property`, `run_str_property`, `run_list_int_property` with case execution loops, discard tracking, and structured JSON failure diagnostics.
- **Replay Engine (`property/replay.nox`)**: Exact case reproduction via `replay_int_case`, `replay_str_case`, and atomic failure artifact serialization (`save_failure_artifact`).
- **Test Coverage**:
  - `tests/prng_test.nox`: Golden vector sequences, ranges, and seed derivation independence.
  - `tests/shrink_test.nox`: Bounded shrink searches for int boundaries, zero reduction, string delta-debugging, and list chunk elimination.
  - `tests/property_test.nox`: 1,000-case deterministic runs, failure shrinking to minimal counterexamples, and replay verification.
- **CI / Workflows**: Cross-platform GitHub Actions workflow covering Linux (x64 and ARM64), macOS (Apple Silicon ARM64), and Windows x64.
- **Publish Workflow**: Added automated `noxc publish` workflow for new tags.

## [0.1.0] - 2026-11-02 - Phase 1: Quartz Core MVP

### Added
- **Core Types (`core/types.nox`)**: `STATUS_PASS`, `STATUS_FAIL`, `STATUS_SKIP`, `STATUS_ERROR`, `STATUS_TIMEOUT`, `TestResult`, `Failure`, `ResourceLimits`, `RunnerConfig`, and `RunSummary`.
- **Assertion Library (`core/assert.nox`)**: `assert_true`, `assert_false`, `assert_eq_int`, `assert_not_eq_int`, `assert_eq_str`, `assert_not_eq_str`, `assert_eq_bool`, `assert_eq_float` (with abs/rel tolerances), `assert_contains_str`, and `assert_not_contains_str`.
- **Test Suite & Runner (`core/suite.nox`)**: `TestSuite` with non-raising `check_*` methods that accumulate results, test filtering by pattern/tag, and result counting.
- **Fixture Lifecycle (`core/fixture.nox`)**: `Fixture` base class with `setup()` / `teardown()`, `__enter__` / `__exit__` context management, and `run_with_fixture` functional wrapper guaranteeing cleanup on exceptions.
- **Reporting & Redaction (`core/report.nox`)**: JUnit XML formatter, version 1 summary JSON formatter, secret redaction (`redact_secrets`), and atomic report writers (`write_junit_report`, `write_json_report`).
- **Comprehensive Verification (`tests/core_test.nox`)**: 25+ assertions testing all core types, suites, fixtures, filters, and reports.

## [0.1.0-dev] - 2026-11-02 - Phase 0: Technical Probes & Baseline Skeleton

### Added
- Initial package manifest `nox.json` and lockfile `nox.lock`.
- Foundation types and public facade in `quartz.nox`.
- Baseline test in `tests/baseline_test.nox` verifying `noxc test` execution.
- Mini-probes across compiler backends, classes, generics, exceptions, concurrency, timing, fs, json, and process execution.
- Architectural Decision Records in `docs/architecture.md`.
