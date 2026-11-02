# Changelog

All notable changes to the Quartz test framework will be documented in this file.
This project adheres to [Semantic Versioning](https://semver.org/).

## [0.1.0] - 2026-11-02 - Phase 1: Quartz Core MVP

### Added
- **Core Types (`core/types.nox`)**: `STATUS_PASS`, `STATUS_FAIL`, `STATUS_SKIP`, `STATUS_ERROR`, `STATUS_TIMEOUT`, `TestResult`, `Failure`, `ResourceLimits`, `RunnerConfig`, and `RunSummary`.
- **Assertion Library (`core/assert.nox`)**: `assert_true`, `assert_false`, `assert_eq_int`, `assert_not_eq_int`, `assert_eq_str`, `assert_not_eq_str`, `assert_eq_bool`, `assert_eq_float` (with abs/rel tolerances), `assert_contains_str`, and `assert_not_contains_str`.
- **Test Suite & Runner (`core/suite.nox`)**: `TestSuite` with non-raising `check_*` methods that accumulate results, test filtering by pattern/tag, and result counting.
- **Fixture Lifecycle (`core/fixture.nox`)**: `Fixture` base class with `setup()` / `teardown()`, `__enter__` / `__exit__` context management, and `run_with_fixture` functional wrapper guaranteeing cleanup on exceptions.
- **Reporting & Redaction (`core/report.nox`)**: JUnit XML formatter, version 1 summary JSON formatter, secret redaction (`redact_secrets`), and atomic report writers (`write_junit_report`, `write_json_report`).
- **Comprehensive Verification (`tests/core_test.nox`)**: 25+ assertions testing all core types, suites, fixtures, filters, and reports.
- **Documentation**: Public API reference in `docs/api.md`.

## [0.1.0-dev] - 2026-11-02 - Phase 0: Technical Probes & Baseline Skeleton

### Added
- Initial package manifest `nox.json` and lockfile `nox.lock`.
- Foundation types and public facade in `quartz.nox`.
- Baseline test in `tests/baseline_test.nox` verifying `noxc test` execution.
- Mini-probes across compiler backends, classes, generics, exceptions, concurrency, timing, fs, json, and process execution.
- Architectural Decision Records in `docs/architecture.md`.
