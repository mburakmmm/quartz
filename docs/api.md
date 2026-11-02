# Quartz API Reference — v0.9.0

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
- `VirtualUserState`
  - `.vu_id: int`
  - `.completed: int`
  - `.successful: int`
  - `.failed: int`
  - `.total_latency_ms: int`
  - `.record(latency_ms: int, success: bool) -> None`
  - `.to_json() -> str`
- `LoadTestSummary`
  - `.all_passed: bool`
  - `.aborted: bool`
  - `.abort_reason: str`
  - `.metrics: LoadMetrics`
  - `.threshold_results: list[ThresholdResult]`
  - `.virtual_users: list[VirtualUserState]`
  - `.to_json() -> str`
- `run_task_workload(workload: WorkloadConfig, task_func: (int) -> bool, thresholds: list[Threshold], limits: SafetyLimits) -> LoadTestSummary`
- `run_handler_load_test(workload: WorkloadConfig, handler: (HttpRequest) -> HttpResponse, req: HttpRequest, thresholds: list[Threshold], limits: SafetyLimits) -> LoadTestSummary`
- `run_network_load_test(workload: WorkloadConfig, req: HttpRequest, base_url: str, policy: NetworkPolicy, thresholds: list[Threshold], limits: SafetyLimits) -> LoadTestSummary`

---

## 21. Property-Driven Contract Testing (`contract/property_contract.nox`)

- `PropertyContractResult`
  - `.total_cases: int`
  - `.passed_cases: int`
  - `.failed_cases: int`
  - `.counterexample_input: str`
  - `.counterexample_error: str`
  - `.seed: int`
  - `.is_success() -> bool`
  - `.to_json() -> str`
- `generate_valid_json_payload(rng: Prng, schema: SchemaDescriptor) -> str`: Synthesizes JSON string conforming to schema.
- `generate_invalid_json_payload(rng: Prng, schema: SchemaDescriptor) -> str`: Synthesizes JSON string violating schema boundaries/types.
- `run_property_contract_test(handler: (HttpRequest) -> HttpResponse, method: str, path: str, request_schema: SchemaDescriptor | None, expected_status_codes: list[int], response_schema: SchemaDescriptor | None, seed: int, cases: int) -> PropertyContractResult`
- `run_negative_property_contract_test(handler: (HttpRequest) -> HttpResponse, method: str, path: str, request_schema: SchemaDescriptor, expected_error_status: int, seed: int, cases: int) -> PropertyContractResult`

---

## 22. Ecosystem Adapters (`adapters/nyx.nox`, `adapters/aether.nox`)

Optional bridges connecting NOX web frameworks to Quartz Contract and Load testing. Core does not depend on adapters.

### Nyx Adapter (`adapters/nyx.nox`)
- `create_nyx_adapter(router: Router) -> NyxAdapter`
- `NyxAdapter`
  - `.dispatch(req: HttpRequest) -> HttpResponse`
  - `.test_contract(test_case: ContractTestCase) -> ContractCheckResult`
  - `.run_load_test(req: HttpRequest, workload: WorkloadConfig, thresholds: list[Threshold], safety: SafetyLimits) -> LoadTestSummary`

### Aether Adapter (`adapters/aether.nox`)
- `create_aether_adapter(router: Router) -> AetherAdapter`
- `AetherAdapter`
  - `.use_before(mw: (HttpRequest) -> HttpResponse | None) -> None`
  - `.use_after(mw: (HttpRequest, HttpResponse) -> HttpResponse) -> None`
  - `.dispatch(req: HttpRequest) -> HttpResponse`
  - `.test_contract(test_case: ContractTestCase) -> ContractCheckResult`
  - `.run_load_test(req: HttpRequest, workload: WorkloadConfig, thresholds: list[Threshold], safety: SafetyLimits) -> LoadTestSummary`

---

## 23. CLI Entry Point (`cli/main.nox`)

Published as the `quartz` binary via `nox.json` and runnable via `noxc run cli/main.nox -- <command>`:
- `quartz version`: Prints framework identity and version.
- `quartz help`: Displays formatted command usage and flags.
- `quartz property [--seed <int>] [--cases <int>] [--format json|summary] [--out <file>]`: Executes property-based invariant check.
- `quartz fuzz [--seed <int>] [--max-cases <int>]`: Executes mutation fuzzer against target.
- `quartz load [--vus <int>] [--duration <int>] [--requests <int>] [--format json|summary] [--out <file>]`: Runs bounded local load test scenario with latency metrics.
- `quartz replay <file.json>`: Inspects and replays saved failure artifact.
- `quartz contract [spec.json]`: Validates API contracts and inspects OpenAPI specs.
- `quartz test [path]`: Runs test suite execution with static and dynamic verification.
- `quartz report [dir] [--format json|summary] [--out <file>]`: Inspects test result artifacts and exports summaries.

---

## 24. Server-Sent Events (SSE) Stream Contracts (`contract/sse.nox`)

- `SseEvent(event: str, data: str, id: str, retry: int)`
  - `.is_json() -> bool`: Returns true if data payload starts and ends with `{}` or `[]`.
- `parse_sse_events(raw_body: str) -> list[SseEvent]`: Parses chunked/multiline SSE text streams into structured event records, ignoring comments.
- `SseEventExpectation(expected_event: str, schema: SchemaDescriptor, has_schema: bool, require_id: bool)`
- `SseValidationResult(is_valid: bool, total_events: int, errors: list[str])`
  - `.error_summary() -> str`
- `validate_sse_stream(raw_body: str, expectations: list[SseEventExpectation]) -> SseValidationResult`: Validates event stream ordering, event names, required IDs, and typed payload schemas.

---

## 25. Real-Time Streaming JSONL Reporting (`core/report.nox`, `load/executor.nox`)

- `format_jsonl_event(event_type: str, test_id: str, status: str, duration_ms: int, payload_json: str) -> str`: Formats single newline-delimited JSON event.
- `JsonlStreamReporter(path: str)`
  - `.emit(event_type: str, test_id: str, status: str, duration_ms: int, payload_json: str) -> None`: Appends live event to file.
  - `.emit_case_start(test_id: str, case_name: str) -> None`
  - `.emit_case_result(test_id: str, case_name: str, status: str, duration_ms: int, message: str) -> None`
  - `.emit_metric_tick(workload: str, completed: int, throughput_rps: float, p50: int, p95: int) -> None`
  - `.emit_threshold_breach(workload: str, metric_name: str, target_val: float, observed_val: float) -> None`
- `run_task_workload_streaming(workload: WorkloadConfig, task_func: (int) -> bool, thresholds: list[Threshold], limits: SafetyLimits, reporter: JsonlStreamReporter) -> LoadTestSummary`: Runs workload with real-time JSONL telemetry logging.

---

## 26. Interactive HTML Visual Reports (`core/report_html.nox`)

- `escape_html(s: str) -> str`: Escapes HTML entity characters (`&`, `<`, `>`, `"`, `'`).
- `format_html_test_report(title: str, summary: RunSummary, run_id: str, compiler_ver: str, backend: str) -> str`: Produces a complete, dependency-free HTML5 test execution dashboard with embedded SVG distribution bar, status pill badges, summary metrics, and collapsible diagnostic details with secret redaction.
- `format_html_load_report(title: str, summary: LoadTestSummary) -> str`: Produces a complete HTML5 load test report with embedded SVG Latency Percentile step curve (Min, p50, p90, p95, p99, Max), threshold evaluation tables, and per-VU breakdown cards.
- `write_html_report(path: str, html_content: str) -> None`: Atomically saves HTML report to disk, automatically creating parent directories.

---

## 27. WebSocket Handshake & Conversation Protocol Contracts (`contract/websocket.nox`)

- Opcode Constants: `WS_TEXT`, `WS_BINARY`, `WS_PING`, `WS_PONG`, `WS_CLOSE`.
- `WsMessage(msg_type: str, payload: str, timestamp_ms: int = 0)`
- `WsStepExpectation(direction: str, msg_type: str, exact_payload: str, schema: SchemaDescriptor, has_schema: bool, close_code: int)`
  - `ws_expect_text(direction: str, exact_text: str) -> WsStepExpectation`
  - `ws_expect_json(direction: str, schema: SchemaDescriptor) -> WsStepExpectation`
  - `ws_expect_ping(direction: str = "server") -> WsStepExpectation`
  - `ws_expect_pong(direction: str = "client") -> WsStepExpectation`
  - `ws_expect_close(direction: str = "server", close_code: int = 1000) -> WsStepExpectation`
- `WsValidationResult(is_valid: bool, step_index: int, error_message: str)`
- `mock_ws_client_handshake(target: str = "/ws", key: str = "...", subprotocol: str = "") -> HttpRequest`: Constructs compliant RFC 6455 upgrade request.
- `mock_ws_server_handshake(subprotocol: str = "") -> HttpResponse`: Constructs compliant 101 Switching Protocols response.
- `validate_ws_handshake_request(req: HttpRequest, required_subprotocol: str = "") -> WsValidationResult`: Verifies method, Upgrade headers, Sec-WebSocket-Key, Sec-WebSocket-Version: 13, and optional subprotocol.
- `validate_ws_handshake_response(res: HttpResponse, expected_subprotocol: str = "") -> WsValidationResult`: Verifies 101 status code, Upgrade headers, and negotiated subprotocol.
- `validate_ws_conversation(messages: list[WsMessage], expected_steps: list[WsStepExpectation]) -> WsValidationResult`: Verifies message step sequence, opcodes, exact string payloads, and JSON schema constraints.

---

## 28. Property Test Suite & Aggregate Invariant Runner (`property/suite.nox`)

- `PropertySuite(name: str, seed: int = 147291, cases_per_property: int = 100, timeout_ms: int = 5000)`
  - `.check_int(test_id: str, gen: IntGen, predicate: (int) -> bool) -> PropertyResult`
  - `.check_str(test_id: str, gen: StringGen, predicate: (str) -> bool) -> PropertyResult`
  - `.check_list_int(test_id: str, gen: ListIntGen, predicate: (list[int]) -> bool) -> PropertyResult`
  - `.is_successful() -> bool`: Returns true if all registered property invariants passed.
  - `.to_summary() -> RunSummary`: Converts suite results into standard `RunSummary` for unified JUnit XML, summary JSON, and HTML report export.
  - `.to_json() -> str`: Serializes comprehensive suite statistics and counterexample details to JSON.



