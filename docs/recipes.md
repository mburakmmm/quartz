# Quartz Testing Recipes & Best Practices

> **Framework:** Quartz for NOX  
> **Status:** Production Reference  
> **Package:** `quartz`

This guide provides practical, production-ready testing recipes for applications, libraries, and HTTP services built with the NOX programming language.

---

## 1. Unit Testing & Fixture Lifecycles

Quartz Core provides standard assertions and robust `try / finally` fixture lifecycle management.

### Basic Assertion Suite

```nox
from core.assert import assert_eq_int, assert_eq_str, assert_true, assert_eq_float

def test_calculator() -> None:
    assert_eq_int(2 + 2, 4, "addition works")
    assert_eq_str("hello " + "world", "hello world", "string concatenation")
    assert_true(len("nox") == 3, "string length")
    assert_eq_float(3.14159, 3.14150, 0.001, 0.001, "float within tolerance")
```

### Setup and Teardown Fixtures

Use `Fixture` to ensure database connections, temporary files, or mock servers are reliably torn down, even when assertions fail.

```nox
from core.fixture import Fixture, SCOPE_CASE
from core.assert import assert_true

fixture: Fixture = Fixture("db_connection", SCOPE_CASE)
fixture.setup()
try:
    # Test logic with active fixture
    assert_true(fixture.is_active, "database query verified")
finally:
    fixture.teardown()
```

---

## 2. Property-Based Testing & Shrinking

Property-based testing exercises invariant properties across hundreds of boundary-biased, randomized inputs with deterministic PRNG seeds.

### Testing Invariants with `IntGen`

```nox
from property.generator import IntGen
from property.property import PropertyConfig, PropertyResult, run_int_property
from property.shrink import shrink_int, ShrinkResultInt

def test_multiplication_commutative() -> None:
    # Generate integers between -1000 and 1000, boundary-biased around 0, 1, -1
    gen: IntGen = IntGen(-1000, 1000, True)
    config: PropertyConfig = PropertyConfig(100, 5, 42, 5000)

    def prop_commutative(x: int) -> bool:
        return (x * 2) == (2 * x)

    res: PropertyResult = run_int_property("int_commutative", config, gen, prop_commutative)
    if res.status != "PASS":
        # Automatically shrinks to minimal counterexample
        shrunk: ShrinkResultInt = shrink_int(100, prop_commutative, 50)
        print("Failure shrunk to: " + str(shrunk.value))
```

### Deterministic Seed Replay

Every property failure emits a JSON artifact with the root seed and case index:

```bash
# Replay exact failure without rerunning full suite
noxc run cli/main.nox -- replay artifacts/failures/property_list_reverse.json
```

---

## 3. Parser & Decoder Fuzzing

Fuzz untrusted input parsers (JSON, CSV, protocols) with byte mutations, boundary injections, and delta-debugging crash minimization.

### Fuzzing a CSV Parser

```nox
from fuzz.fuzzer import run_fuzz, FuzzSummary
from fuzz.corpus import CorpusManager, default_corpus_config

def parse_csv(data: str) -> None:
    if "TRIGGER_BUG" in data:
        raise Exception("Deliberate parser crash")

corpus: CorpusManager = CorpusManager("csv_target", "corpus", default_corpus_config())
summary: FuzzSummary = run_fuzz(
    "csv_target",
    parse_csv,
    ["name,age\nalice,30\n"],
    500,
    42,
    corpus,
    1024
)

print("Crashes found: " + str(summary.crashes_found))
```

Crashes are deduplicated by error signature and written to `corpus/<target>/crashes/<hash>.json`.

---

## 4. HTTP Contract Testing & OpenAPI 3 Schemas

Contract testing verifies that HTTP responses conform to declared OpenAPI schemas without requiring external network dependencies.

### In-Process Handler Contract

```nox
from nox.http import HttpRequest, HttpResponse
from contract.http import build_request, empty_headers
from contract.schema import schema_object, schema_string, schema_integer, PropertyRule, SchemaDescriptor
from contract.runner import ContractTestCase
from contract.types import ContractCheckResult

def my_api_handler(req: HttpRequest) -> HttpResponse:
    headers: dict[str, str] = {"Content-Type": "application/json"}
    return HttpResponse(200, "{\"id\": 101, \"username\": \"admin\"}", headers)

def test_api_contract() -> None:
    # Declare expected response schema
    fields: list[PropertyRule] = [
        PropertyRule("id", schema_integer(1, 999999, True, True)),
        PropertyRule("username", schema_string(3, 50))
    ]
    expected_schema: SchemaDescriptor = schema_object(fields, ["id", "username"], True)

    req: HttpRequest = build_request("GET", "/users/101", "", empty_headers())
    tc: ContractTestCase = ContractTestCase(
        "users_api_check",
        req,
        [200],
        ["Content-Type"],
        "application/json",
        expected_schema
    )

    result: ContractCheckResult = tc.run_on_handler(my_api_handler)
    if not result.is_valid:
        print("Contract violation: " + result.error_summary())
```

---

## 5. Bounded Local Load Testing

Quartz Load evaluates local HTTP endpoints under closed (virtual users) or open (arrival rate) models with monotonic latency histograms and pass/fail thresholds.

### Running a Closed Workload

```nox
from nox.http import HttpRequest, HttpResponse
from contract.http import build_request, empty_headers
from load.workload import closed_workload, WorkloadConfig
from load.thresholds import Threshold
from load.safety import default_safety_limits, SafetyLimits
from load.executor import run_handler_load_test, LoadTestSummary

workload: WorkloadConfig = closed_workload("api_benchmark", 5, 2, 100) # 5 VUs, 2s duration, max 100 reqs
thresholds: list[Threshold] = [
    Threshold("p95", "<=", 25.0),       # 95th percentile under 25ms
    Threshold("error_rate", "<=", 0.0)  # 0% errors allowed
]

def my_api_handler(req: HttpRequest) -> HttpResponse:
    headers: dict[str, str] = {"Content-Type": "application/json"}
    return HttpResponse(200, "{\"status\": \"ok\"}", headers)

summary: LoadTestSummary = run_handler_load_test(
    workload,
    my_api_handler,
    build_request("GET", "/ping", "", empty_headers()),
    thresholds,
    default_safety_limits()
)

print("Passed: " + str(summary.all_passed))
print("P95 Latency: " + str(summary.metrics.p95()) + "ms")
```

---

## 6. Framework Adapters (Nyx & Aether)

Quartz seamlessly tests web applications built on NOX ecosystem frameworks like Nyx and Aether.

### Nyx Router Contract Testing

```nox
from nox.router import Router, Context
from nox.http import HttpRequest, HttpResponse
from adapters.nyx import create_nyx_adapter, NyxAdapter
from contract.http import build_request, empty_headers

router: Router = Router()
def health_handler(ctx: Context) -> HttpResponse:
    return HttpResponse(200, "{\"status\":\"ok\"}", {})
router.get("/health", health_handler)

adapter: NyxAdapter = create_nyx_adapter(router)
res: HttpResponse = adapter.dispatch(build_request("GET", "/health", "", empty_headers()))
assert res.status == 200
```

### Aether Middleware Pipeline Testing

```nox
from nox.router import Router, Context
from nox.http import HttpRequest, HttpResponse
from adapters.aether import create_aether_adapter, AetherAdapter
from contract.http import build_request, empty_headers

router: Router = Router()
def secure_handler(ctx: Context) -> HttpResponse:
    return HttpResponse(200, "{\"secret\":42}", {})
router.get("/secure", secure_handler)

adapter: AetherAdapter = create_aether_adapter(router)

def auth_middleware(req: HttpRequest) -> HttpResponse | None:
    if "Authorization" not in req.headers:
        return HttpResponse(401, "{\"error\":\"unauthorized\"}", {})
    return None

adapter.use_before(auth_middleware)
res: HttpResponse = adapter.dispatch(build_request("GET", "/secure", "", empty_headers()))
assert res.status == 401
```

---

## 7. Server-Sent Events (SSE) Stream Contract Testing

Validate chunked HTTP event streams (`text/event-stream`), verifying event types, IDs, and payload schema integrity across multiple stream blocks.

```nox
from contract.sse import SseEvent, parse_sse_events, SseEventExpectation, SseValidationResult, validate_sse_stream
from contract.schema import schema_object, schema_string, schema_integer, PropertyRule, SchemaDescriptor

user_schema: SchemaDescriptor = schema_object([
    PropertyRule("user_id", schema_integer(1, 1000, True, True)),
    PropertyRule("email", schema_string(3, 50))
], ["user_id", "email"])

raw_stream: str = "event: user_created\nid: 1\ndata: {\"user_id\": 42, \"email\": \"alice@example.com\"}\n\nevent: ping\ndata: {}\n\n"

expectations: list[SseEventExpectation] = [
    SseEventExpectation("user_created", user_schema, True, True),
    SseEventExpectation("ping", schema_object([], []), True, False)
]

val_result: SseValidationResult = validate_sse_stream(raw_stream, expectations)
assert val_result.is_valid
```

---

## 8. Real-Time JSONL Live Event Logging

Stream live test outcomes and workload performance telemetry directly to append-only `.jsonl` event logs for real-time observability.

```nox
from core.report import JsonlStreamReporter

reporter: JsonlStreamReporter = JsonlStreamReporter("artifacts/live_telemetry.jsonl")
reporter.emit_case_start("api_suite", "test_user_profile")
reporter.emit_metric_tick("workload_api", 100, 2500.0, 2, 6)
reporter.emit_case_result("api_suite", "test_user_profile", "PASS", 4, "ok")
```

---

## 9. WebSocket Handshake & Conversation Protocol Contracts

Validate RFC 6455 WebSocket handshakes and full duplex message conversations with typed JSON schema validation for payloads.

```nox
from nox.http import HttpRequest, HttpResponse
from contract.schema import schema_object, schema_string, PropertyRule, SchemaDescriptor
from contract.websocket import (
    WS_TEXT, WS_PING, WS_PONG, WsMessage, WsStepExpectation, WsValidationResult,
    ws_expect_text, ws_expect_json, ws_expect_ping, ws_expect_pong,
    validate_ws_handshake_request, validate_ws_conversation, mock_ws_client_handshake
)

# 1. Handshake verification
req: HttpRequest = mock_ws_client_handshake("/chat", "dGhlIHNhbXBsZSBub25jZQ==", "chat.v1")
res: WsValidationResult = validate_ws_handshake_request(req, "chat.v1")
assert res.is_valid

# 2. Conversation contract verification
msg_schema: SchemaDescriptor = schema_object([
    PropertyRule("text", schema_string(1, 100))
], ["text"])

script: list[WsStepExpectation] = [
    ws_expect_text("client", "hello"),
    ws_expect_json("server", msg_schema),
    ws_expect_ping("server"),
    ws_expect_pong("client")
]

dialogue: list[WsMessage] = [
    WsMessage(WS_TEXT, "hello", 5),
    WsMessage(WS_TEXT, "{\"text\":\"welcome!\"}", 12),
    WsMessage(WS_PING, "", 20),
    WsMessage(WS_PONG, "", 22)
]

conv_res: WsValidationResult = validate_ws_conversation(dialogue, script)
assert conv_res.is_valid
```

---

## 10. Interactive HTML Visual Reports with Embedded SVG Charts

Generate zero-dependency, self-contained HTML5 reports with interactive SVG latency curves and test distribution visual charts.

```nox
from core.types import RunSummary, TestResult, STATUS_PASS
from core.report_html import format_html_test_report, format_html_load_report, write_html_report

# Export test execution HTML dashboard
summary: RunSummary = RunSummary()
summary.record(TestResult("t1", "test_api_ping", STATUS_PASS, 4, "", None))
html_content: str = format_html_test_report("Core API Suite", summary, "run-2026", "noxc 2.0.108", "llvm")
write_html_report("artifacts/test_report.html", html_content)
```

---

## 11. Multi-Stage Ramp Workload Testing

Execute dynamic staged load profiles with ramp-up, steady plateau, and ramp-down phases, collecting per-stage latency histograms and transition telemetry.

```nox
from load.stages import LoadStage, StagedWorkloadConfig, StagedLoadSummary, run_staged_task_workload
from load.safety import SafetyLimits, default_safety_limits
from load.thresholds import Threshold

stages: list[LoadStage] = [
    LoadStage(5, 2),    # Ramp up to 5 VUs over 2s
    LoadStage(20, 5),   # Ramp up to 20 VUs over 5s (plateau)
    LoadStage(0, 2)     # Ramp down to 0 VUs over 2s
]
cfg: StagedWorkloadConfig = StagedWorkloadConfig("api_ramp_profile", stages, 50, 1000)
thresholds: list[Threshold] = [
    Threshold("error_rate", "<=", 0.01),
    Threshold("p95_latency", "<=", 150.0)
]
safety: SafetyLimits = default_safety_limits()

def task(idx: int) -> bool:
    # Execute request / handler
    return True

summary: StagedLoadSummary = run_staged_task_workload(cfg, task, thresholds, safety)
assert summary.all_stages_passed
```

---

## 12. Property Generator Combinators & Weighted Distributions

Synthesize rich domain test data using one_of choices, weighted probability distributions, and invariant-preserving filtered generators.

```nox
from core.prng import Prng
from property.generator import IntGen
from property.combinators import (
    OneOfIntGen, OneOfStringGen,
    WeightedStringOption, WeightedChoiceStringGen,
    FilteredIntGen, MappedIntGen
)

prng: Prng = Prng(42)

# 1. Weighted choice generator (e.g., 80% GET, 15% POST, 5% DELETE)
methods: list[WeightedStringOption] = [
    WeightedStringOption(80, "GET"),
    WeightedStringOption(15, "POST"),
    WeightedStringOption(5, "DELETE")
]
gen_method: WeightedChoiceStringGen = WeightedChoiceStringGen(methods)
req_method: str = gen_method.generate(prng)

# 2. Filtered generator with discard budget
def is_even(n: int) -> bool:
    return (n % 2 == 0)

even_gen: FilteredIntGen = FilteredIntGen(IntGen(1, 100, False), is_even)
val: int = even_gen.generate(prng)
assert (val % 2 == 0)
```

