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
    assert_eq_float(3.14159, 3.14150, 0.001, "float within tolerance")
```

### Setup and Teardown Fixtures

Use `Fixture` to ensure database connections, temp files, or mock servers are reliably torn down, even when assertions fail.

```nox
from core.fixture import Fixture
from core.assert import assert_true

def setup_db() -> str:
    # Initialize in-memory resource
    return "db_active_connection"

def teardown_db() -> None:
    # Always executed even on test failure
    print("Database cleaned up")

fixture: Fixture = Fixture(setup_db, teardown_db)
fixture.enter()
try:
    assert_true(True, "database query verified")
finally:
    fixture.exit()
```

---

## 2. Property-Based Testing & Shrinking

Property-based testing exercises invariant properties across hundreds of boundary-biased, randomized inputs with deterministic PRNG seeds.

### Testing Invariants with `IntGen`

```nox
from property.generator import IntGen
from property.property import PropertyConfig, run_int_property
from property.shrink import shrink_int

def test_multiplication_commutative() -> None:
    # Generate integers between -1000 and 1000, boundary-biased around 0, 1, -1
    gen: IntGen = IntGen(-1000, 1000)
    config: PropertyConfig = PropertyConfig(100, 42, 50)

    def prop_commutative(x: int) -> bool:
        return (x * 2) == (2 * x)

    res = run_int_property("int_commutative", gen, prop_commutative, config)
    if not res.passed:
        # Automatically shrinks to minimal counterexample
        minimal: int = shrink_int(res.failing_input, prop_commutative, 50)
        print("Failure shrunk to: " + str(minimal))
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
from fuzz.fuzzer import Fuzzer, FuzzOutcome, OUTCOME_NORMAL, OUTCOME_CRASH
from fuzz.harness import run_in_process

def parse_csv(data: str) -> None:
    if "TRIGGER_BUG" in data:
        raise Exception("Deliberate parser crash")

fuzzer: Fuzzer = Fuzzer("csv_target")
fuzzer.add_seed("name,age\nalice,30\n")

# Run 500 bounded iterations
fuzzer.run_bounded(parse_csv, 500)
```

Crashes are deduplicated by error signature and written to `corpus/<target>/crashes/<hash>.json`.

---

## 4. HTTP Contract Testing & OpenAPI 3 Schemas

Contract testing verifies that HTTP responses conform to declared OpenAPI schemas without requiring external network dependencies.

### In-Process Handler Contract

```nox
from contract.http import build_request, empty_headers, HttpResponse
from contract.schema import schema_object, schema_string, schema_integer, SchemaField
from contract.compare import compare_response, ContractCheckResult

def my_api_handler(req) -> HttpResponse:
    headers: dict[str, str] = {}
    headers["Content-Type"] = "application/json"
    return HttpResponse(200, "{\"id\": 101, \"username\": \"admin\"}", headers)

def test_api_contract() -> None:
    # Declare expected response schema
    fields: list[SchemaField] = [
        SchemaField("id", schema_integer(1, 999999), True),
        SchemaField("username", schema_string(3, 50), True)
    ]
    expected_schema = schema_object(fields, True)

    req = build_request("GET", "/users/101", "", empty_headers())
    res = my_api_handler(req)

    result: ContractCheckResult = compare_response(res, [200], expected_schema)
    if not result.is_valid:
        for err in result.errors:
            print("Contract violation: " + err)
```

---

## 5. Bounded Local Load Testing

Quartz Load evaluates local HTTP endpoints under closed (virtual users) or open (arrival rate) models with monotonic latency histograms and pass/fail thresholds.

### Running a Closed Workload

```nox
from load.workload import closed_workload
from load.thresholds import Threshold
from load.safety import default_safety_limits
from load.executor import run_handler_load_test, LoadTestSummary

workload = closed_workload("api_benchmark", 5, 2, 100) # 5 VUs, 2s duration, max 100 reqs
thresholds = [
    Threshold("p95_latency_ms", "<=", 25.0), # 95th percentile under 25ms
    Threshold("error_rate", "<=", 0.0)       # 0% errors allowed
]

summary: LoadTestSummary = run_handler_load_test(
    workload,
    my_api_handler,
    build_request("GET", "/ping", "", empty_headers()),
    thresholds,
    default_safety_limits()
)

print("Passed: " + str(summary.all_passed))
print("P95 Latency: " + str(summary.metrics.percentile(95.0)) + "ms")
```

---

## 6. Framework Adapters (Nyx & Aether)

Quartz seamlessly tests web applications built on NOX ecosystem frameworks like Nyx and Aether.

### Nyx Router Contract Testing

```nox
from adapters.nyx import create_nyx_adapter, NyxAdapter
from contract.http import build_request, empty_headers, HttpResponse

adapter: NyxAdapter = create_nyx_adapter()
adapter.register_route("GET", "/health", lambda req: HttpResponse(200, "{\"status\":\"ok\"}", {}))

res: HttpResponse = adapter.handle(build_request("GET", "/health", "", empty_headers()))
assert res.status == 200
```

### Aether Middleware Pipeline Testing

```nox
from adapters.aether import create_aether_adapter, AetherAdapter
from contract.http import build_request, empty_headers, HttpResponse

adapter: AetherAdapter = create_aether_adapter()

def auth_middleware(req):
    # Short-circuit unauthorized requests
    if "Authorization" not in req.headers:
        return HttpResponse(401, "{\"error\":\"unauthorized\"}", {})
    return None

adapter.use_before(auth_middleware)
adapter.register_route("GET", "/secure", lambda req: HttpResponse(200, "{\"secret\":42}", {}))

# Verify unauthorized response
res = adapter.handle(build_request("GET", "/secure", "", empty_headers()))
assert res.status == 401
```
