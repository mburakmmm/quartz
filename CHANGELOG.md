# Changelog

All notable changes to the Quartz test framework will be documented in this file.
This project adheres to [Semantic Versioning](https://semver.org/).

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
