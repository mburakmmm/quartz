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
        shrunk: ShrinkResultInt = shrink_int(res.seed, prop_commutative, 50)
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
