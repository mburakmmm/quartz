# Quartz API Reference — v0.1.0 Core MVP

## 1. Core Types (`core/types.nox`)

### Status Constants
- `STATUS_PASS: str = "PASS"`
- `STATUS_FAIL: str = "FAIL"`
- `STATUS_SKIP: str = "SKIP"`
- `STATUS_ERROR: str = "ERROR"`
- `STATUS_TIMEOUT: str = "TIMEOUT"`

### Classes
- `Failure(message: str, expected: str, actual: str)`
- `TestResult(id: str, name: str, status: str, duration_ms: int, message: str, failure: Failure | None)`
  - `.is_passed() -> bool`
  - `.is_failed() -> bool`
  - `.is_skipped() -> bool`
  - `.is_error() -> bool`
- `ResourceLimits(timeout_ms: int, max_memory_mb: int)`
- `RunnerConfig(filter_pattern: str, tag_filter: str, stop_on_failure: bool, backend: str)`
- `RunSummary()`
  - `.record(result: TestResult) -> None`
  - `.is_successful() -> bool`

---

## 2. Assertions (`core/assert.nox`)

All assertion functions raise the built-in `AssertionError` on failure.

- `assert_true(cond: bool, msg: str) -> None`
- `assert_false(cond: bool, msg: str) -> None`
- `assert_eq_int(actual: int, expected: int, msg: str) -> None`
- `assert_not_eq_int(actual: int, unexpected: int, msg: str) -> None`
- `assert_eq_str(actual: str, expected: str, msg: str) -> None`
- `assert_not_eq_str(actual: str, unexpected: str, msg: str) -> None`
- `assert_eq_bool(actual: bool, expected: bool, msg: str) -> None`
- `assert_eq_float(actual: float, expected: float, abs_tol: float, rel_tol: float, msg: str) -> None`
  - Validates `|actual - expected| <= max(abs_tol, |expected| * rel_tol)`.
- `assert_contains_str(haystack: str, needle: str, msg: str) -> None`
- `assert_not_contains_str(haystack: str, needle: str, msg: str) -> None`

---

## 3. Test Suite & Runner (`core/suite.nox`)

`TestSuite(name: str)` provides non-raising assertion checks that accumulate results into `RunSummary` without prematurely halting test execution.

### Methods
- `.set_filter(pattern: str) -> None`
- `.check_true(case_name: str, cond: bool) -> None`
- `.check_false(case_name: str, cond: bool) -> None`
- `.check_eq_int(case_name: str, actual: int, expected: int) -> None`
- `.check_not_eq_int(case_name: str, actual: int, unexpected: int) -> None`
- `.check_eq_str(case_name: str, actual: str, expected: str) -> None`
- `.check_not_eq_str(case_name: str, actual: str, unexpected: str) -> None`
- `.check_eq_bool(case_name: str, actual: bool, expected: bool) -> None`
- `.check_eq_float(case_name: str, actual: float, expected: float, abs_tol: float, rel_tol: float) -> None`
- `.check_contains_str(case_name: str, haystack: str, needle: str) -> None`
- `.total_count() -> int`
- `.pass_count() -> int`
- `.fail_count() -> int`
- `.skip_count() -> int`
- `.all_passed() -> bool`

---

## 4. Fixtures (`core/fixture.nox`)

Provides deterministic lifecycle management with `try / finally` guarantees.

- `SCOPE_CASE: str = "case"`
- `SCOPE_SUITE: str = "suite"`
- `Fixture(name: str, scope: str)`
  - `.setup() -> None`
  - `.teardown() -> None`
  - `__enter__() -> Fixture`
  - `__exit__() -> None`
- `run_with_fixture(fixture: Fixture, action: () -> None) -> None`

---

## 5. Reporting & Redaction (`core/report.nox`)

- `escape_xml(s: str) -> str`
- `escape_json(s: str) -> str`
- `redact_secrets(text: str) -> str`
  - Replaces sensitive tokens (`Bearer ...`) with `[REDACTED]`.
- `format_junit_xml(suite_name: str, summary: RunSummary) -> str`
- `format_summary_json(run_id: str, backend: str, compiler_version: str, summary: RunSummary) -> str`
- `write_junit_report(path: str, suite_name: str, summary: RunSummary) -> None`
- `write_json_report(path: str, run_id: str, backend: str, compiler_version: str, summary: RunSummary) -> None`
