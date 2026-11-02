# Quartz API Reference — v0.6.1

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
- `ResourceLimits(timeout_ms: int, max_memory_mb: int)`
- `RunnerConfig(filter_pattern: str, tag_filter: str, stop_on_failure: bool, backend: str)`
- `RunSummary()`

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
- `assert_contains_str(haystack: str, needle: str, msg: str) -> None`
- `assert_not_contains_str(haystack: str, needle: str, msg: str) -> None`

---

## 3. Test Suite & Runner (`core/suite.nox`)

`TestSuite(name: str)` provides non-raising assertion checks accumulating into `RunSummary`.

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
- `run_with_fixture(fixture: Fixture, action: () -> None) -> None`

---

## 5. Explicit-State PRNG (`core/prng.nox`)

Deterministic PRNG independent of global random state.

- `Prng(seed: int)`
  - `.next_int() -> int` (positive 63-bit integer)
  - `.next_int_range(min_val: int, max_val: int) -> int` (inclusive range)
  - `.next_float() -> float` (in `[0.0, 1.0)`)
  - `.next_bool() -> bool`
  - `.choose_index(count: int) -> int`
- `derive_case_seed(root_seed: int, test_id: str, case_index: int, rng_version: int) -> int`

---

## 6. Property Generators (`property/generator.nox`)

- `IntGen(min_val: int, max_val: int, with_boundaries: bool)`
  - Boundary bias includes `min_val`, `max_val`, `0`, `1`, `-1`.
- `FloatGen(min_val: float, max_val: float)`
- `BoolGen()`
- `StringGen(min_len: int, max_len: int, charset: str)`
  - Character sets: `CHARSET_ALPHA`, `CHARSET_ALPHANUMERIC`, `CHARSET_ASCII_PRINTABLE`, `CHARSET_TURKISH_UNICODE`.
- `ByteGen(min_len: int, max_len: int)`
- `ListIntGen(min_len: int, max_len: int, item_gen: IntGen)`

---

## 7. Shrinking Engine (`property/shrink.nox`)

Minimal counterexample extraction preserving invariant violation.

- `shrink_int(initial: int, predicate: (int) -> bool, max_attempts: int) -> ShrinkResultInt`
  - Tries zero, sign flip, halving (`// 2`), and boundary steps.
- `shrink_str(initial: str, predicate: (str) -> bool, max_attempts: int) -> ShrinkResultStr`
  - Tries empty string, delta-debugging chunk elimination, and tail reduction.
- `shrink_list_int(initial: list[int], predicate: (list[int]) -> bool, max_attempts: int) -> ShrinkResultListInt`
  - Tries empty list, chunk halving, and element dropping.

---

## 8. Property Runner & Replay (`property/property.nox`, `property/replay.nox`)

- `PropertyConfig(cases: int, max_discard_ratio: int, seed: int, timeout_ms: int)`
- `default_property_config(seed: int) -> PropertyConfig`
- `PropertyResult`
  - `.is_passed() -> bool`
  - `.to_json() -> str`
- `run_int_property(test_id: str, config: PropertyConfig, gen: IntGen, predicate: (int) -> bool) -> PropertyResult`
- `run_str_property(test_id: str, config: PropertyConfig, gen: StringGen, predicate: (str) -> bool) -> PropertyResult`
- `run_list_int_property(test_id: str, config: PropertyConfig, gen: ListIntGen, predicate: (list[int]) -> bool) -> PropertyResult`
- `replay_int_case(test_id: str, seed: int, case_index: int, gen: IntGen, predicate: (int) -> bool) -> bool`
- `replay_str_case(test_id: str, seed: int, case_index: int, gen: StringGen, predicate: (str) -> bool) -> bool`
- `save_failure_artifact(path: str, result: PropertyResult) -> None`

---

## 9. Fuzz Target Harness (`fuzz/harness.nox`)

- `OUTCOME_NORMAL: str = "NORMAL"`
- `OUTCOME_HANDLED_REJECTION: str = "HANDLED_REJECTION"`
- `OUTCOME_ASSERTION_VIOLATION: str = "ASSERTION_VIOLATION"`
- `OUTCOME_TIMEOUT: str = "TIMEOUT"`
- `OUTCOME_CRASH: str = "CRASH"`

### Classes & Functions
- `HandledRejection(Exception)`: Distinct exception indicating input rejection without triggering crash classification.
- `FuzzOutcome(status: str, message: str, signature: str, duration_ms: int)`
  - `.is_failure() -> bool`
  - `.is_bug() -> bool`
- `run_in_process(target_func: (str) -> None, input_data: str) -> FuzzOutcome`
- `run_out_of_process(program: str, args: list[str], timeout_ms: int) -> FuzzOutcome`

---

## 10. Mutation Engine (`fuzz/mutation.nox`)

Bounded deterministic byte and structured mutations:
- `mutate(data: str, prng: Prng, max_len: int) -> str`
- `mutate_byte_flip(data: str, prng: Prng) -> str`
- `mutate_byte_insert(data: str, prng: Prng, max_len: int) -> str`
- `mutate_byte_delete(data: str, prng: Prng) -> str`
- `mutate_byte_duplicate(data: str, prng: Prng, max_len: int) -> str`
- `mutate_byte_swap(data: str, prng: Prng) -> str`
- `mutate_splice(data1: str, data2: str, prng: Prng, max_len: int) -> str`
- `mutate_boundary_number(data: str, prng: Prng, max_len: int) -> str`
- `mutate_delimiters(data: str, prng: Prng, max_len: int) -> str`
- `mutate_json(data: str, prng: Prng, max_len: int) -> str`
- `mutate_csv(data: str, prng: Prng, max_len: int) -> str`

---

## 11. Corpus Management & Crash Minimization (`fuzz/corpus.nox`, `fuzz/minimize.nox`, `fuzz/fuzzer.nox`)

- `CorpusConfig(max_files: int, max_total_bytes: int, max_input_len: int)`
- `default_corpus_config() -> CorpusConfig`
- `CorpusManager(target_name: str, root_dir: str, config: CorpusConfig)`
  - `.ensure_directories() -> None`
  - `.add_seed(content: str) -> bool`
  - `.load_seeds() -> list[str]`
  - `.is_new_signature(signature: str) -> bool`
  - `.save_crash_artifact(input_data: str, outcome: FuzzOutcome, seed: int, case_index: int) -> str`
- `minimize_crash(target_func: (str) -> None, failing_input: str, expected_signature: str, max_steps: int) -> MinimizationResult`
- `run_fuzz(target_name: str, target_func: (str) -> None, initial_seeds: list[str], iterations: int, seed: int, corpus: CorpusManager, max_len: int, timeout_ms: int = 0, fail_on_bug: bool = False) -> FuzzSummary`

---

## 12. HTTP Transport Abstraction (`contract/http.nox`)

- `build_request(method: str, target: str, body: str, headers: dict[str, str]) -> HttpRequest`
- `build_response(status: int, body: str, headers: dict[str, str]) -> HttpResponse`
- `empty_headers() -> dict[str, str]`
- `execute_handler(handler: (HttpRequest) -> HttpResponse, req: HttpRequest) -> HandlerExecutionResult`
  - In-process direct execution with zero network sockets.

---

## 13. Schema Descriptors & Validation (`contract/schema.nox`)

- `PropertyRule(name: str, schema: SchemaDescriptor)`
- `SchemaDescriptor(type_name: str)`
- Factory functions:
  - `schema_string(min_len: int = -1, max_len: int = -1) -> SchemaDescriptor`
  - `schema_enum(enum_vals: list[str]) -> SchemaDescriptor`
  - `schema_integer(min_val: int = 0, max_val: int = 0, has_min: bool = False, has_max: bool = False) -> SchemaDescriptor`
  - `schema_number(min_val: float = 0.0, max_val: float = 0.0, has_min: bool = False, has_max: bool = False) -> SchemaDescriptor`
  - `schema_boolean() -> SchemaDescriptor`
  - `schema_null() -> SchemaDescriptor`
  - `schema_array(item_schema: SchemaDescriptor, min_len: int = -1, max_len: int = -1) -> SchemaDescriptor`
  - `schema_object(properties: list[PropertyRule], required_keys: list[str], is_strict: bool = True) -> SchemaDescriptor`
- `validate_schema(schema: SchemaDescriptor, val: JsonValue, path: str) -> list[str]`

---

## 14. Response Verification & Runner (`contract/compare.nox`, `contract/runner.nox`, `contract/types.nox`)

- `ContractCheckResult(is_valid: bool, errors: list[str], warnings: list[str], status_code: int, duration_ms: int)`
  - `.has_errors() -> bool`
  - `.error_summary() -> str`
- `get_header_ci(headers: dict[str, str], target_name: str) -> str`
- `has_header_ci(headers: dict[str, str], target_name: str) -> bool`
- `redact_header_value(header_name: str, value: str) -> str`
- `validate_response(res: HttpResponse, expected_statuses: list[int], required_headers: list[str], expected_content_type: str, body_schema: SchemaDescriptor | None, duration_ms: int) -> ContractCheckResult`
- `ContractTestCase(name: str, request: HttpRequest, expected_statuses: list[int], required_headers: list[str], expected_content_type: str, body_schema: SchemaDescriptor | None)`
  - `.run_on_handler(handler: (HttpRequest) -> HttpResponse) -> ContractCheckResult`

---

## 15. OpenAPI Subset Importer (`contract/openapi.nox`)

- `UnsupportedContractFeature(Exception)`
- `parse_openapi_json(doc_json_str: str) -> ParsedOpenApiDoc`
  - `.title: str`
  - `.version: str`
  - `.operations: list[ParsedOperation]`
  - Supports `minLength`, `maxLength`, `minimum`, `maximum`, `minItems`, `maxItems`.
  - Supports `additionalProperties` (defaults to non-strict per OpenAPI 3 spec).
  - Supports status codes and ranges (`2XX`..`5XX`, `default`).
  - Rejects unsupported keywords (`$ref`, `oneOf`, `anyOf`, `allOf`, `not`, `pattern`, `discriminator`, etc.) by raising `UnsupportedContractFeature`.

---

## 16. Authorized Network Transport & Safety Gate (`contract/http_network.nox`)

- `NetworkSafetyError(Exception)`
- `NetworkNotAllowedError(NetworkSafetyError)`
- `UnauthorizedTargetError(NetworkSafetyError)`

### Classes & Policies
- `NetworkPolicy(allow_network: bool, authorized_target: str, timeout_ms: int, max_body_bytes: int)`
- `default_network_policy() -> NetworkPolicy` (strictly disables network)
- `loopback_network_policy(timeout_ms: int = 2000) -> NetworkPolicy` (allows loopback targets only)
- `authorized_network_policy(authorized_target: str, timeout_ms: int = 2000) -> NetworkPolicy` (allows loopback + exact matching origin)
- `parse_url(url: str) -> ParsedUrl`
  - `.scheme: str`, `.host: str`, `.port: int`, `.path: str`, `.origin: str`
- `is_loopback_host(host: str) -> bool`
- `validate_network_target(url: str, policy: NetworkPolicy) -> ParsedUrl`
- `execute_network_request(req: HttpRequest, base_url: str, policy: NetworkPolicy) -> NetworkExecutionResult`
  - `.responses: list[HttpResponse]`
  - `.duration_ms: int`
  - `.status_code: int`
  - `.error_message: str`
  - `.is_timeout: bool`
  - `.is_success: bool`
- `run_network_contract_check(req: HttpRequest, base_url: str, policy: NetworkPolicy, expected_statuses: list[int], required_headers: list[str], expected_content_type: str, body_schema: SchemaDescriptor | None) -> ContractCheckResult`
- `ContractTestCase.run_on_network(base_url: str, policy: NetworkPolicy) -> ContractCheckResult`

---

## 17. Load Safety Guardrails & Circuit Breaker (`load/safety.nox`)

- `LoadSafetyError(Exception)`
- `CircuitBreakerTrippedError(LoadSafetyError)`
- Hard safety limits:
  - `HARD_CAP_VIRTUAL_USERS: int = 20`
  - `HARD_CAP_DURATION_SECONDS: int = 60`
  - `HARD_CAP_RATE_RPS: int = 100`
  - `HARD_CAP_IN_FLIGHT: int = 50`
  - `HARD_CAP_TIMEOUT_MS: int = 10000`
- `SafetyLimits(virtual_users: int = 5, duration_seconds: int = 5, rate_rps: int = 10, max_in_flight: int = 10, timeout_ms: int = 2000, max_consecutive_errors: int = 5, max_error_ratio: float = 0.5)`
- `default_safety_limits() -> SafetyLimits`
- `CircuitBreaker(max_consecutive: int = 5)`
  - `.record_success() -> None`
  - `.record_failure(reason: str) -> None`
  - `.check_healthy() -> None` (raises `CircuitBreakerTrippedError` when tripped)
- `validate_load_target(target: str, authorized_target: str = "") -> None`

---

## 18. Bounded Histograms & Latency Metrics (`load/metrics.nox`)

- `LoadMetrics()`
  - Fixed 136-bucket logarithmic-linear histogram guaranteeing $O(1)$ memory usage.
  - `.record_scheduled() -> None`
  - `.record_rejected() -> None`
  - `.record_result(status_code: int, duration_ms: int, is_timeout: bool, is_success: bool) -> None`
  - `.p50() -> int`
  - `.p90() -> int`
  - `.p95() -> int`
  - `.p99() -> int`
  - `.percentile(p: float) -> int`
  - `.mean_latency_ms() -> float`
  - `.throughput_rps(total_duration_ms: int) -> float`
  - `.error_rate() -> float`
  - `.to_json(total_duration_ms: int) -> str`

---

## 19. Performance Threshold Criteria (`load/thresholds.nox`)

- `Threshold(metric_name: str, operator: str, target_value: float)`
  - Supported metrics: `p50`, `p90`, `p95`, `p99`, `mean_latency_ms`, `max_latency_ms`, `error_rate`, `throughput_rps`, `failed`, `timed_out`, `rejected`.
  - Supported operators: `<=`, `<`, `>=`, `>`, `==`.
- `ThresholdResult`
  - `.passed: bool`
  - `.message: str`
- `evaluate_threshold(t: Threshold, metrics: LoadMetrics, total_duration_ms: int) -> ThresholdResult`
- `evaluate_all_thresholds(thresholds: list[Threshold], metrics: LoadMetrics, total_duration_ms: int) -> list[ThresholdResult]`
- `all_thresholds_passed(results: list[ThresholdResult]) -> bool`

---

## 20. Workload Models & Scheduler (`load/workload.nox`, `load/executor.nox`)

- `MODEL_CLOSED: str = "closed"`
- `MODEL_OPEN: str = "open"`
- `WorkloadConfig(name: str, model: str = "closed", virtual_users: int = 5, duration_seconds: int = 5, target_rps: int = 10, max_in_flight: int = 10, max_iterations: int = 100, warmup_seconds: int = 0)`
- `closed_workload(name: str, virtual_users: int = 5, duration_seconds: int = 5, max_iterations: int = 100) -> WorkloadConfig`
- `open_workload(name: str, target_rps: int = 10, duration_seconds: int = 5, max_in_flight: int = 10, max_iterations: int = 100) -> WorkloadConfig`
- `LoadTestSummary`
  - `.all_passed: bool`
  - `.aborted: bool`
  - `.abort_reason: str`
  - `.metrics: LoadMetrics`
  - `.threshold_results: list[ThresholdResult]`
  - `.to_json() -> str`
- `run_task_workload(workload: WorkloadConfig, task_func: (int) -> bool, thresholds: list[Threshold], limits: SafetyLimits) -> LoadTestSummary`
- `run_handler_load_test(workload: WorkloadConfig, handler: (HttpRequest) -> HttpResponse, req: HttpRequest, thresholds: list[Threshold], limits: SafetyLimits) -> LoadTestSummary`
- `run_network_load_test(workload: WorkloadConfig, req: HttpRequest, base_url: str, policy: NetworkPolicy, thresholds: list[Threshold], limits: SafetyLimits) -> LoadTestSummary`


