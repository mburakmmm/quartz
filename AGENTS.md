# AGENTS.md — Quartz Test Framework for the NOX Ecosystem

> Status: IMPLEMENTATION BLUEPRINT / NOT YET IMPLEMENTED  
> Language: NOX 2.x; implementation language: Nox first  
> Scope: unit/fixture testing, property-based testing, shrinking, fuzzing, HTTP contract testing and authorized load testing  
> Principle: Quartz complements `nox.test` and `noxc test`; it does not replace or fork the compiler.

## 0. Agent operating contract — mandatory

You are an autonomous coding agent working on **Quartz**, the shared test framework for the NOX ecosystem. Execute the roadmap incrementally, never claim an unimplemented API or passing test. Read this file, the project's README, `nox.json`, and the actual current NOX standard-library docs before editing source.

**Non-negotiable rules**

1. Quartz is a **standalone package**. Its core must not depend on Nyx, Aether, Atlas, Feather, Aura, or any app framework. Integrations live in optional adapters.
2. Implement runnable `.nox` modules and tests; use NOX syntax and real APIs available in the pinned compiler. Pseudocode in this blueprint is a **proposed interface**, not evidence that a NOX feature exists.
3. The default workflow is `noxc check`, `noxc test`, and separate, explicitly invoked advanced-test entry points. Do not add a fictitious `noxc quartz` command to the compiler. Installable `quartz` CLI is a separate package `bin` entry point when supported.
4. A failing property or fuzz case must report a **replayable seed, case index, generator parameters, and minimal counterexample** (when shrinking is supported). Never suppress failures.
5. The property-generation RNG must be **per-run/per-case state**, independent of global `nox.random.seed` and task scheduling. Identical seed + config + version + backend must reproduce a case.
6. Fuzzers and load generators must enforce bounded memory, iteration count, corpus size, input size, execution time and in-flight work. No unrestricted infinite loops in CI.
7. **Never direct load or fuzz traffic to an external host by default.** Loopback/in-process only unless the operator explicitly authorizes a target and its limits. Never circumvent a remote service's protections.
8. Never store credentials, bearer tokens, cookies, personal data, unredacted HTTP bodies or other secrets in snapshots, crash corpora, logs or reports.
9. Every phase must include positive, negative, edge-case, deterministic replay and regression tests plus docs. Do not advance an incomplete phase.
10. Do not modify `nox-lang` compiler/runtime to make Quartz pass without a separately documented upstream issue and an explicit, reviewed dependency decision. Prefer compatible library-level implementations.
11. No unbounded `spawn`, global mutable RNG for parallel workers, shell-string command construction, or blocking `sleep_ms` in portable scheduling paths.
12. Commit `nox.lock`, pin dependency versions and identify NOX compiler/backend versions in all reproducible reports.

### Required agent workflow for every feature

1. Discover current state and inspect the relevant Nox API docs; record assumptions and capability gaps.
2. Make a small vertical slice and add focused tests first or alongside the change.
3. Run compiler check and tests. If tools are not available, state **NOT RUN** explicitly; never substitute a changelog claim for evidence.
4. Add failure-path and resource-limit tests.
5. Update examples, API reference, and `CHANGELOG.md`.
6. Document commands executed, results, remaining risks, and the next smallest step in the PR/commit description.

## 1. Product definition

**Quartz** is a test framework for applications, libraries, HTTP APIs and services written in NOX.

Product pillars:

- **Quartz Core** — assertions, test suites, fixtures, setup/teardown, isolation, tagging, result capture, JSON/JUnit reports, deterministic runner.
- **Quartz Property** — composable typed generators, property checks, reproducible seeds, shrinking, classifications and coverage observations.
- **Quartz Fuzz** — mutation and structure-aware fuzzing, corpus management, reproducer, minimization, fault classification and crash artifacts.
- **Quartz Contract** — in-process HTTP handler contracts and opt-in real HTTP integration contracts, reusable schema validation and compatibility reports.
- **Quartz Load** — safe bounded workloads, concurrency/rate control, latency histograms, latency percentiles, thresholds, phase profiles and exportable performance reports.
- **Quartz Adapters** — optional integrations for Nyx, Aether, NOX standard HTTP, Atlas, Feather and CI systems; no coupling in Core.

**Not in initial scope:** a distributed commercial load-testing SaaS, general browser E2E automation, exhaustive OpenAPI 3.1 / JSON Schema support, a general-purpose formal verification engine, coverage-guided LLVM libFuzzer replacement, cluster orchestration or OS-level coverage instrumentation. Such features require separate specifications and prerequisites.

### Success definition

A NOX developer must be able to (a) write tests, (b) generate hundreds of deterministic inputs, (c) shrink a failure, (d) fuzz a parser with a bounded corpus, (e) verify HTTP responses against a declared contract and (f) measure an explicitly authorized API's latency and error thresholds—all using Quartz with clear, machine-readable results.

## 2. Validated platform assumptions (verify again at implementation)

Documentation reviewed from `mburakmmm/nox-lang`:

- `noxc test` discovers `*_test.nox`, compiles and runs each file, and returns nonzero for failures. Individual test file invocations are supported.
- `nox.test` provides assertion functions, `TestSuite`, and JUnit XML output.
- `nox.random` provides seedable xoshiro256 with **per-program/per-worker state**; do **not** rely on it for cross-worker independent property generation. Implement a dedicated explicit-state Quartz PRNG.
- `nox.http.HttpRequest` / `HttpResponse` can be constructed to test handlers in-process; `nox.http.request` supports HTTP/HTTPS and `timeout_ms`; HTTP 3xx is **not automatically followed**.
- `nox.json` exposes parsed `JsonValue` with a **floating-point number representation**; large integers above 2^53 require explicit preservation strategies when parsing contract bodies.
- `nox.time.instant_now()` measures monotonic time; `nox.time.now_ms()` is wall clock and not suitable for latency measurement.
- `spawn`, `await`, and typed `Channel[T]` are available. QBE uses cooperative single-threaded scheduling; LLVM uses M:N work stealing. **Do not assume the same level of parallelism across backends**.
- `nox.time.sleep_ms` blocks a worker and, on QBE, blocks the only worker. Rate scheduling must be designed/tested to avoid this pitfall.
- `nox.process.Command` can run binaries without shell interpretation but captures output and blocks; use only where appropriate and with timeouts.
- `nox.json`, `nox.fs`, `nox.os`, `nox.http` depend on the capabilities of the build profile; network tests cannot run under a freestanding profile.
- `noxc build --backend llvm|qbe` and hosted target support exist; do not presume a test command supports arbitrary backend flags. Build a separate verified matrix runner as needed.
- `nox.json` package metadata supports a `bin` entry to publish an installable `quartz` CLI; `noxc publish` publishes package metadata, not source bundles.

See: `https://github.com/mburakmmm/nox-lang/blob/main/docs/tools/testing.md`, `docs/stdlib/test.md`, `docs/stdlib/http.md`, `docs/stdlib/random.md`, `docs/stdlib/time.md`, `docs/language/concurrency.md`, `docs/tools/packages.md`.

### Capability-spike requirement

Before coding the runner, execute tiny compiling probes for: exported classes and generics, function-value callbacks, error catching, `Channel` with workload messages, file I/O and atomic reporting, timer precision, HTTP request timeout semantics, arg parsing, and NOX backend compatibility. Adapt the public API to verified language support; **do not quietly add compiler dependencies**.

## 3. Architecture and boundaries

```text
                            +------------------+
                            | quartz CLI       |
                            | selection/report |
                            +---------+--------+
                                      |
                            +---------v--------+
                            | quartz.core      |
                            | runner, suites,  |
                            | fixtures, result |
                            +----+----+----+---+
                                 |    |    |
                     +-----------+    |    +-------------------+
                     |                |                        |
                +----v------+    +----v------+           +-----v------+
                | property  |    | fuzz      |           | contract   |
                | generator |    | mutation  |           | HTTP/schema|
                | shrinker  |    | corpus    |           +-----+------+
                +-----------+    +-----------+                 |
                                                     +---------v--------+
                                                     | load / transport |
                                                     | bounded scheduler|
                                                     +------------------+

                Optional integrations: quartz.adapters.*
```

### Layer responsibilities

- `core` is pure or nearly pure, requiring no network and no third-party framework.
- `property` depends on `core` and explicit-state randomness; shrink is a separate deterministic algorithm.
- `fuzz` depends on `core` and `property` primitives but never on HTTP by default.
- `contract` depends on `core`, `nox.json`, typed schema evaluation and pluggable transports; handler transport is the default.
- `load` depends on `core`, transport and metrics; **must not import the core test runner back into itself**.
- `cli` coordinates and reports. The library must remain usable without CLI installation.
- `adapters` translate app-specific route/framework setup into the common handler/request abstraction.

**Dependency direction:** CLI → modules → Core. No circular imports. Design around Nox's actual module system and avoid relying on runtime reflection or decorators for automatic discovery unless verified.

## 4. Proposed repository layout

```text
quartz/
  AGENTS.md
  README.md
  CHANGELOG.md
  LICENSE
  nox.json
  nox.lock
  quartz.nox                 # public facade
  core/
    types.nox               # status, case, limits, error, result types
    assert.nox              # assertion wrappers
    suite.nox               # explicit registration and runner
    fixture.nox             # lifecycle and cleanup
    prng.nox                # explicit deterministic RNG state
    report.nox              # JSON/JUnit output, redaction
  property/
    generator.nox           # primitive and composite generators
    property.nox            # case runner and assumption accounting
    shrink.nox              # bounded shrink search
    replay.nox              # seed and counterexample replay
  fuzz/
    harness.nox            # target interface, watchdog orchestration
    mutation.nox           # bounded byte/structured mutations
    corpus.nox             # storage, dedupe and retention
    minimize.nox           # crash minimization
  contract/
    types.nox              # Contract/Operation/Schema descriptors
    schema.nox             # strict supported subset validation
    http.nox               # handler transport and network transport
    compare.nox            # response headers/body/status validation
    openapi.nox            # optional supported OpenAPI subset importer
  load/
    workload.nox           # workload definitions, phases, budgets
    executor.nox           # async task/channel scheduling
    metrics.nox            # bounded histogram and aggregates
    thresholds.nox         # pass/fail performance conditions
    safety.nox             # target validation and hard caps
  adapters/
    nyx.nox                # optional
    aether.nox             # optional
  cli/
    main.nox               # proposed installable 'quartz' entry point
  tests/
    core_test.nox
    prng_test.nox
    property_test.nox
    shrink_test.nox
    fuzz_test.nox
    contract_handler_test.nox
    contract_http_test.nox
    load_scheduler_test.nox
    load_metrics_test.nox
    report_test.nox
    backend_matrix_test.nox
  examples/
    property_example.nox
    fuzz_csv_example.nox
    contract_example.nox
    load_local_example.nox
  fixtures/
    corpus/
    contracts/
  docs/
    architecture.md
    api.md
    recipes.md
    security.md
    compatibility.md
  scripts/
    # optional thin CI wrappers, not implementation core
```

File paths are target layout, not evidence those files already exist. Subdirectory/module import support and package layout must be verified in Phase 0.

## 5. Core public API — design, not yet implemented

### Core types

Define typed data for `TestCase`, `TestResult`, `Failure`, `TestSuite`, `Fixture`, `RunnerConfig`, `RunSummary`, `Artifact`, `ResourceLimits`. `TestResult` must include `PASS`, `FAIL`, `SKIP`, `ERROR`, and `TIMEOUT` statuses. Include run ID, timestamps, duration, compiler/backend info, seed, case index and test name. Avoid throwing away diagnostic context.

### Core assertions and fixtures

- Assertions: equals for int/float/string/bool, approximate float with explicit abs/relative tolerances, raises, not-equals, contains, type expectation.
- Avoid exact float equality as an implicit default for statistical results.
- Fixtures: deterministic `setup()` / `teardown()` with `try/finally`; scopes: **case** and **suite** initially; session scope only when cleanup and isolation are proven.
- Test discovery: explicit registry or `_test.nox` files; **no unsupported annotation magic**.
- Test isolation: separate processes for risky fuzz cases when enabled; ordinary unit tests may run in-process with bounded timeout semantics.
- Reports: streaming JSONL for live events; summary JSON and JUnit XML for CI; atomic artifact writes where Nox I/O supports it.

**Illustrative proposed API (validate against the compiler):**

```nox
from quartz.core import TestSuite

suite: TestSuite = TestSuite("math")
suite.check_eq_int("addition", 2 + 2, 4)
suite.write_junit("artifacts/math.xml")
```

Where possible reuse the existing `nox.test.TestSuite` rather than creating an incompatible duplicate; document any Quartz wrapper and conversion.

## 6. Property-based testing specification

### Generator algebra

Implement a typed `Gen[T]`-like public interface **only if compiler generics and first-class function constraints allow it**. Otherwise expose explicit `IntGen`, `StringGen`, `ListIntGen`, `JsonGen` and a type-erased serializable `ValueGen` strategy. The runnable API takes precedence over conceptual elegance.

Initial generators:

- integer: bounded inclusive, boundary-biased, optional signed distribution;
- float: finite by default, optional NaN, +/-Inf, signed zero, subnormals;
- boolean and enum-choice;
- ASCII / Unicode string, including Turkish letters, CJK and emoji; invalid UTF-8 is a **byte generator**, not a string generator;
- byte arrays / raw buffers with explicit max length;
- list, tuple/record, optional/null, union/one-of, dictionary/JSON tree with depth and element bounds;
- filter/map/combine where callback support is verified; fallback to explicit generator class methods.

### Deterministic randomness

Implement Quartz-owned explicit RNG state with documented algorithm and version identifier (e.g. a safe bounded 31-bit LCG first, upgrade only with golden-vector compatibility tests). Never seed global `nox.random` concurrently. Derive worker/case seeds from `(root_seed, test_id, case_index, rng_version)` through a deterministic noncryptographic mixing routine whose signed overflow behavior is defined. Preserve replay across scheduler changes. Never use random generators for secrets.

### Property runner

- `cases` default 100, max bounded in CI; `max_discard_ratio`, `max_size`, `seed` and per-test timeout configurable.
- Separate `pass`, `fail`, `discard/assumption rejection`, `error`, and `timeout`.
- Store the **first actual failing generated input** before shrinking; shrink only candidates that reproduce the **same failure class/signature**.
- On failure save root seed, test ID, case index, path through composed generators, original input, minimal input and shrink attempts.
- Support `assume(predicate)` with skip/discard metrics; never report success when almost all cases were discarded.
- Add `classify(label)` and `collect(value)` instrumentation as later optional steps; these are metrics, not assertions.

### Shrinking

- Integer: 0, sign, halves, nearby boundaries.
- String: empty, chunks removed, Unicode code points reduced.
- List: delete chunks (delta-debugging), shrink individual elements.
- JSON object: remove keys, shrink values and nesting.
- Structured generators must track enough context to generate **valid** shrink candidates; allow invariant-preserving shrink hooks.
- Deterministic priority order, budget by attempt count/time, visited-set dedupe, candidate size/depth caps.
- Output must be a locally minimal counterexample under configured shrink moves, **not an unsupported claim of global minimality**.

**Illustrative desired behavior:**

```text
PROPERTY FAILED  list_reverse_twice
seed=147291  case=37  rng=v1  backend=llvm
original=[4, 9, -2, 0, 88]
shrunk=[1]
replay: quartz replay artifacts/failures/<id>.json
```

### Property acceptance tests

- Same seed/input config yields byte-identical case stream across separate runs and CI agents.
- Boundary generator includes `INT64_MIN`, `INT64_MAX`, zero and adjacent values when allowed.
- Failed invariant is found and shrunk consistently; true invariant never reports a false positive.
- Invalid shrink candidates are skipped, not accepted as failures.
- Discard exhaustion returns an explicit inconclusive/error state.
- Parallel generation preserves identity of each case and replay independent of completion order.

## 7. Fuzzing specification

### Target model

- **In-process target** for pure parsers/serializers/decoders: fast but cannot reliably recover from native crashes or endless loops.
- **Out-of-process target** for compiler, native-code, unsafe plugin and crash-prone targets: watchdog, process timeout, bounded stdout/stderr and exit-code classification.
- **HTTP target** via the Contract transport in a later opt-in phase, always with network restrictions.

Target API concept: `FuzzTarget.run_case(bytes) -> FuzzOutcome` or an equivalent verified Nox API. Outcomes: normal, assertion violation, handled rejection, timeout, crash, resource-limit, infrastructure error. Invalid inputs being rejected correctly are not bugs.

### Engines

1. **Corpus replay** first: stable byte-for-byte inputs and expected outcome category.
2. **Mutation fuzzing:** bit flip, byte flip, insert, delete, duplicate, splice, boundary number replacement, UTF-8 split/injection, format-specific delimiter/quote mutation.
3. **Structure-aware fuzzing:** JSON, CSV and HTTP request models; synthesize valid and deliberately invalid documents.
4. **Generation-based fuzzing:** feed Quartz Property generators into target harness.
5. **Coverage-guided fuzzing is NOT claimed** until real compiler/runtime instrumentation or a stable external coverage provider exists; initial prioritization uses corpus novelty, response/error signatures and deterministic heuristics, not fake branch coverage.

### Corpus and minimization

- Directory layout: `corpus/<target>/seeds`, `crashes`, `timeouts`, `regressions`.
- Content-derived stable ID (cryptographic digest when available), bounded total bytes and file counts, safe filenames and atomic writes.
- Separate public sanitized corpus from private sensitive local artifacts.
- All crash artifacts include binary input, seed, case number, target version, NOX compiler/backend, timeout, failure signature and reproduction command.
- Minimize with bounded delta debugging while preserving the original failure class. Compare crash signature; do not minimize to a different bug accidentally.
- Fuzzer may stop on first failure in CI or continue and dedupe locally, both configurable.

### Fuzz acceptance tests

- Parser crash fixture is found from a saved seed, classified and replayed.
- Corpus mutation never exceeds configured bytes/depth/input count.
- Known valid CSV/JSON fixtures continue to round-trip when expected.
- Malformed UTF-8 and null bytes are passed without lossy transcoding.
- Timeout and abnormal process exit become `FAIL/ERROR`, not a falsely passing case.
- Seed corpus survives interrupted runs without corruption.

## 8. HTTP contract testing specification

### Scope and transport

Start with **handler-first** contract testing: call a NOX `HttpRequest -> HttpResponse` handler directly, no port or network. Next implement opt-in network integration through `nox.http.request` for loopback test servers and authorized targets. Transports must implement the same contract execution interface so comparisons remain equivalent.

### Contract model

- Operation: method, path pattern, request headers/query/body expectations, allowed status codes, response headers and JSON schema.
- Support schema types: null, boolean, string, finite number, integer (with 2^53 awareness), array, object, enum, required keys, optional properties, min/max lengths/values, simple pattern constraints only if supported.
- Initial importer is a **documented OpenAPI subset**, not full OpenAPI/JSON Schema. Unsupported keywords must produce a clear `UnsupportedContractFeature` error, not be ignored silently.
- Validate HTTP header names case-insensitively. Normalize media types and charset parameters carefully; preserve raw values in diagnostic evidence.
- Validate status, required headers, body presence, JSON shape, content type, optional examples and declared error responses.
- Declare schema strictness: default unknown-property policy must be configurable, not implicit.
- Use robust URL/path/query parsing, percent-encoding rules and duplicate query-key policy; do not invent spec conformance.
- For integer JSON values exceeding 2^53, either use a validated raw-number parsing strategy or fail with an explicit precision/unsupported warning. NEVER silently round contract identity fields.

### Provider and consumer workflows

- Provider tests: run application handler against example requests and generated negative/edge cases; verify declared responses.
- Consumer tests: save expectation fixtures from known consumer requirements; compare provider behavior to the consumer contract.
- Compatibility summary: breaking (removed route, method, required field, narrower type, removed status) vs additive changes under an explicitly versioned policy.
- A schema snapshot is reviewed and version-controlled; updating expected results requires explicit approval, not auto-accepting current production output.
- Authentication: use injected test credentials or fake provider, never hardcode secrets; redact authorization and cookie headers.
- Fault injection: malformed JSON, missing auth, conflicting content types, oversized body, timeouts and server errors.

### HTTP safety

- Default transport = in-process. Network mode requires explicit `--allow-network` and target allowlist.
- Default allowlist loopback; explicit remote hosts permitted only for owned/authorized environments.
- Validate DNS resolution and actual connected address where transport facilities allow; refuse private/public mismatches and suspicious redirect chains. `nox.http` does not automatically follow redirects; Quartz should keep redirects **disabled by default**.
- Set per-request timeout and an independent **overall-case deadline**; HTTP's per-operation idle timeout is not a whole-request deadline for streamed bodies.
- Cap request/response body sizes, headers, retries and log excerpts.
- Never blindly replay a recorded POST/DELETE against production services.

### Contract acceptance tests

- In-process happy path, missing required field, wrong media type, incorrect status and invalid response JSON.
- Header matching ignores casing while preserving values.
- `401/403/404/422/500` fixtures validate declared error contracts.
- Unsupported OpenAPI keyword fails explicitly.
- Network disallowed by default; remote host without authorization is rejected.
- Fixtures with secrets verify redaction in JSON, JUnit, console and repro artifacts.

## 9. Load and performance testing specification

### Threat model and guardrails

Load testing can disrupt services. **Only test systems you control or have explicit written authorization to test.** Default to local development endpoints and in-process handlers. This module must make unsafe use difficult even when invoked incorrectly.

Hard safety gates:

- Without explicit authorization: only `127.0.0.1`, `::1` or in-process handlers.
- Opt-in host allowlist, exact scheme/host/port, resolved-IP validation when available; no wildcard `*`, no remote redirects, no silent proxy bypass.
- Explicit upper bounds for users, request rate, test duration, number of requests, request/response bytes, open connections and in-flight tasks. Configurable caps may be **lowered by default**, not silently removed.
- Default limits for local smoke: `virtual_users <= 5`, `duration <= 10s`, `rate <= 10 req/s`, `max_in_flight <= 10`, `timeout <= 2s`; values are proposed defaults to validate in Phase 0.
- A deliberate `--authorized-target <exact-origin>` plus explicit higher limits is required for larger tests; CI policies may impose stricter ceilings.
- Stop scheduling when a global budget is exhausted, when shutdown is requested, or when a safety circuit breaker triggers.
- Never attempt distributed amplification, credential guessing, bypassing throttling, or DoS-style stress against third-party infrastructure.

### Scheduling models

- **Closed model:** N virtual users issue next request after finishing prior request; measures service under user-style concurrency.
- **Open model:** arrivals scheduled at a target rate; a bounded queue and maximum in-flight count enforce backpressure. Reject/skip arrivals when saturated; do not create unbounded tasks.
- Separate warmup, measurement, ramp-up, steady and cooldown phases.
- Use `nox.time.instant_now()` for latency and phase duration. Avoid relying on wall-clock `now_ms()` for elapsed time.
- `spawn`/`Channel` fan-out/fan-in worker pattern. **No `list[Task]` assumption on NOX 2.0**; use channel-based result collection.
- Since QBE scheduling is cooperative and `sleep_ms` blocks its worker, begin with a verified LLVM-hosted workload driver. QBE runs correctness/smoke tests only until scheduling parity is proven.
- For real HTTP load tests, use bounded `nox.http.request` with per-request timeouts; verify actual concurrency with a local test server and telemetry. A client timeout is distinct from server response error.

### Metrics

Emit at least:

- scheduled, started, completed, successful, failed, timed out, rejected due to bounds;
- requests/sec and achieved arrival rate versus requested rate;
- latency min, max, mean, p50, p90, p95 and p99;
- status code counts, transport errors, timeout count, contract violation count;
- run duration, bytes sent/received (if byte measurement is supported), max observed in-flight;
- threshold pass/fail and environment metadata.

Use a bounded histogram, HDR-style or bucketed scheme with a documented precision/maximum value. Percentiles must not require retaining one sample object per request during long runs. Document interpolation, empty-sample behavior, out-of-range values, and whether failed requests count toward latency distributions. Never invent missing or unsupported metrics such as true server CPU or memory without a separate instrumented exporter.

### Threshold examples

- `p95_latency_ms <= 250`
- `error_rate <= 0.01`
- `throughput_rps >= 50`
- `timeouts == 0`
- `contract_violations == 0`

The CLI exits nonzero when an explicitly configured threshold is violated. A safety rejection and a broken target are distinct failure classes. Zero successful requests never yields a bogus passing p95 threshold.

### Load acceptance tests

- Against local deterministic handler: count totals, percentile monotonicity and threshold outcomes.
- Artificial 50/100/200 ms latency fixtures produce plausibly ordered metrics without assuming exact OS timing.
- Hard upper bound on concurrency and queued work is observed under a deliberately slow server.
- Stop condition triggers on duration, request limit, error threshold and cancellation.
- No external traffic is sent when the target is not authorized.
- Compare identical closed and open workloads by requested vs achieved rate; show saturation/rejected arrivals.
- Validate cleanup: all tasks/streams/reports closed on normal completion, error and cancellation.

## 10. CLI and output contracts

The `quartz` executable is a **planned separate entry point**, not an existing compiler command. Initial developer execution should also work through `noxc run cli/main.nox -- ...` after code exists. Implement only options verified by tests.

**Target CLI UX (design):**

```text
quartz test     [path] [--filter NAME] [--report DIR]
quartz property [entry] --seed 42 --cases 1000 --max-shrinks 500
quartz fuzz     [target] --seed 42 --max-cases 10000 --max-input-bytes 65536
quartz replay   artifacts/failures/case.json
quartz contract [contract-file] --handler-fixture LOCAL
quartz load     [scenario-file] --local-only
quartz report   [run-directory]
quartz version
```

When CLI support is not yet implemented, examples must show real runnable `noxc run ...` commands instead, and never pretend otherwise.

### Report schema (versioned)

Every run exports:

```json
{
  "schema_version": "1",
  "engine": "quartz",
  "run_id": "example-run",
  "kind": "property",
  "status": "FAIL",
  "seed": 42,
  "compiler_version": "record-at-runtime",
  "backend": "record-at-runtime",
  "cases_executed": 37,
  "failure_artifact": "artifacts/failures/example.json",
  "duration_ms": 123
}
```

This JSON is a **contract illustration**, not a preexisting report. Avoid misleading placeholders in actual output: fields that cannot be measured must be omitted or explicitly null. Version the schema and keep backward compatibility. Test report parsing, escaping, redaction and atomic writing.

## 11. Phased implementation roadmap

### Phase 0 — Repository, technical probes and contract freeze

Tasks:

- [ ] Create `nox.json`, `nox.lock`, README, architecture and minimal Quartz facade.
- [ ] Pin a tested NOX compiler version and verify `noxc check`, `noxc test`, LLVM/QBE build workflows.
- [ ] Compile mini-probes for callbacks/generics, `spawn`+channels, exception catching, networking, process timeouts and JSON.
- [ ] Decide and document feasible callback/generator/fixture APIs based on compiler reality.
- [ ] Create capability matrix (`hosted`, `freestanding`, LLVM, QBE, wasm targets); mark unverified targets as **UNVERIFIED**.
- [ ] Specify supported run/replay/report formats and safety limits.

Exit criteria: base project compiles and runs at least one test with expected failing exit code; actual supported APIs are documented.

### Phase 1 — Quartz Core MVP

Tasks:

- [ ] Test result model, runner, explicit suite registration, assertions, fixtures.
- [ ] `try/finally` fixture cleanup on pass/fail/timeout; isolation strategy.
- [ ] Deterministic test IDs, filtering, simple tagging.
- [ ] JSON/JSONL and JUnit outputs; proper exit codes.
- [ ] `noxc test` integration without compiler modifications.

Exit criteria: 20+ relevant positive/negative core assertions, deterministic repeated output, JUnit parseability and cleanup verification.

### Phase 2 — Property engine and deterministic replay

Tasks:

- [ ] Explicit RNG state and golden vectors.
- [ ] Integer/float/string/byte/list/JSON bounded generators.
- [ ] Property run loop with case/discard counts and failure signatures.
- [ ] Replay by seed, test ID and case index; shrinker for int/string/list/JSON.
- [ ] Shrink budget, dedupe, persistent counterexamples and CI output.

Exit criteria: 1,000-case deterministic runs reproduce on two runs; known seeded failures shrink and replay identically; discard exhaustion never passes.

### Phase 3 — Fuzz engine

Tasks:

- [ ] Pure target harness, byte mutations, structured JSON/CSV mutation.
- [ ] Corpus persistence, stable IDs, dedupe and minimizer.
- [ ] Bounded out-of-process harness with timeout where supported.
- [ ] Known crash/timeout fixtures, regression replay and failure classification.
- [ ] Integrate safe fuzz targets from Atlas CSV parser or NOX codec as optional validation projects.

Exit criteria: known bug seed discovered, minimized and replayed; malformed input never escapes resource caps; no false-positive crash for handled invalid input.

### Phase 4 — HTTP contract tests

Tasks:

- [ ] Transport-neutral request/response representation and in-process handler adapter.
- [ ] Strict supported JSON schema subset and manual contracts.
- [ ] Status, headers, body, request validation with good diffs.
- [ ] Authorized loopback network transport and timeouts.
- [ ] Versioned fixture snapshots, secret redaction and breaking-change classifier.
- [ ] Optional Nyx/Aether adapters; avoid core dependencies.

Exit criteria: positive/negative provider tests, JSON mismatch reports, unsupported keywords fail explicitly, no unauthorized network request possible by default.

### Phase 5 — Load testing MVP

Tasks:

- [ ] Safe local-only workload config and policy validator.
- [ ] Closed virtual-user model with bounded channel workers.
- [ ] Monotonic timings and bounded latency histogram.
- [ ] Open arrival-rate model with backpressure/rejected-arrival accounting.
- [ ] Ramp, warmup, steady, cooldown; throughput and thresholds.
- [ ] Controlled local fixture server and cancellation/cleanup tests.

Exit criteria: caps proven under saturation; p95/error thresholds accurate in fixture scenarios; forbidden remote target never contacted; no unbounded task/memory growth.

### Phase 6 — Cross-module hardening and integrations

Tasks:

- [ ] Integrate generated payloads with contract tests; optional bounded HTTP fuzz testing.
- [ ] Run Quartz itself against LLVM and QBE with recorded environment and results; quarantine unsupported semantics rather than claiming compatibility.
- [ ] Add end-to-end sample projects: Atlas property tests, Nyx/Aether handler contracts, local NOX server load smoke tests.
- [ ] Add deterministic CI pipelines and optional nightly extended runs.
- [ ] Verify reports are parseable by standard CI tooling, sanitize corpus artifacts and test all failures.

Exit criteria: multi-module CI jobs produce consistent results, crash replays are stable, documented feature-by-backend matrix is accurate.

### Phase 7 — v1.0 stabilization

Tasks:

- [ ] Freeze public interfaces and document semver compatibility.
- [ ] Fix all confirmed P0/P1 defects; triage P2 with workarounds.
- [ ] Benchmark Quartz runner overhead, peak memory, 100k+ property cases and high-volume bounded load scenarios locally.
- [ ] Complete user guide, examples, reference, security guidance, migration notes and release checklist.
- [ ] Tag release, publish package metadata through supported `noxc publish` workflow once registry access is available.

Exit criteria: all release gates in section 12 pass and all unsupported capabilities are explicitly marked.

## 12. Quality gates and CI policy

**Every PR**

- Syntax/type check on changed Nox modules.
- `noxc test` on relevant `*_test.nox` modules.
- Deterministic seed regression test and one negative test.
- JSON/JUnit schema validation.
- Static checks for new unbounded loops, unsafe network defaults, leaked secrets or unchecked file/process operations.
- Reviewer notes specify backend/compiler versions and `NOT RUN` where appropriate.

**Every release candidate**

- Full core + property + fuzz + contract + load suite in a sandboxed local environment.
- LLVM hosted and QBE hosted test matrix where supported; reports identify absent jobs and failures separately.
- 10+ repeated benchmark iterations with median/min/max, warmup and fixed hardware/build metadata.
- Deterministic replay across two fresh processes.
- Corpus recovery test after intentional interruption.
- Security tests for disallowed external targets, header/token redaction, limits, and unsupported schema keywords.
- At least 1,000 randomized property cases per selected invariant and 10,000 bounded fuzz cases per major parser target for nightly jobs; these are proposed targets, revise based on runtime measurements.
- Public docs cover limitations and semantics (float, null, JSON precision, scheduling, HTTP timeout).

### Definition of done for any task

A task is **DONE** only if the code is merged, compiler-checked, relevant tests actually executed, failure behavior covered, resources bounded, docs updated and an execution log exists. Otherwise mark it `PARTIAL`, `BLOCKED` or `NOT RUN`.

## 13. Test inventory — minimum required scenarios

| Area | Must test |
|---|---|
| RNG | fixed golden sequence, repeatability, per-worker independence, boundary bias |
| Generator | zero/empty/null, Unicode, INT64 edges, bounded recursion and sizes |
| Shrinker | deterministic local minimum, invariant preservation, cycle protection, budget |
| Fixtures | cleanup on all exit paths, isolation, duplicate setup prevention |
| Fuzz | valid/invalid corpus, crash/timeout classification, replay, safe filename, limits |
| HTTP | status/header/body mismatch, case-insensitive headers, invalid JSON, precision limits |
| Security | no unauthorized remote requests, token redaction, size/time/case caps |
| Load | closed/open scheduling, saturation, latency percentiles, cancellation, zero successes |
| Report | schema version, JUnit/JSON validity, paths, escaping, secret masking |
| Backend | LLVM/QBE deterministic results and any unsupported capability explicitly reported |

## 14. Integration priorities across NOX ecosystem

1. **Atlas:** property invariants (`sort` idempotence, CSV round-trip under documented conditions, GroupBy conservation, join relationships), fuzz CSV delimiters/UTF-8/quoted fields, compare against pinned reference fixtures.
2. **Nyx:** request/response contracts, status and JSON schema checks, local API performance baselines.
3. **Aether:** router/handler contracts, middleware tests and bounded local load tests.
4. **Feather:** data generator-backed ORM CRUD/property tests and transaction invariants; use ephemeral databases and deterministic fixture cleanup.
5. **NOX compiler/runtime (optional external consumer):** separate process harness, compiler crash reproductions and differential backend tests; do not couple compiler code into Quartz core.

Each integration must be an optional adapter/demo that preserves standalone Quartz usage. Never require an actual production database or live public endpoint for CI.

## 15. Versioning and scope control

Proposed releases:

| Version | Deliverable |
|---|---|
| v0.1.0 | Core runner, assertions, fixtures, JSON/JUnit reports |
| v0.2.0 | Property generators, seeds, shrinking, replay |
| v0.3.0 | Fuzz corpus, mutation, minimization and watchdog |
| v0.4.0 | HTTP handler contracts and schema validator |
| v0.5.0 | Authorized local HTTP tests, adapters and snapshots |
| v0.6.0 | Bounded load scheduler, metrics and thresholds |
| v0.7.0 | Cross-module workflows, CI matrix, advanced reports |
| v0.8.x | Reliability, performance, hardening, release candidate |
| v1.0.0 | Stable documented API with verified matrix and security gates |

Versions are roadmap targets, not current versions. Never bump the version because a feature was planned; bump only when completed and validated.

## 16. Decisions and open questions to resolve by technical spike

- Can NOX callbacks be stored with a typed `Gen[T]`, or are concrete generator classes necessary?
- Can case-specific process isolation be implemented efficiently with Nox's current `nox.process` API? If not, write a restricted external test runner adapter and disclose limitations.
- Is there a suitable nonblocking deadline/timer primitive for high concurrency? If no, restrict QBE load tests and choose a verified scheduler design for LLVM.
- How will failed-case property functions be identified and invoked across restarts without reflection? Prefer explicit stable registration names.
- Which OpenAPI schema keywords are realistically supported without native regex or schema parser packages?
- How will raw JSON integers >2^53 be handled without silent numeric conversion?
- What memory measurement API is available per platform? If none, distinguish optional OS process measurements from Quartz's own counters.
- What backend/target combinations have been executed in CI versus merely compiled?

Record each decision in `docs/architecture.md` as an ADR with alternatives, constraints, and reproducible evidence.

## 17. Next agent action — start here

Implement **Phase 0 only**. First audit the NOX compiler and stdlib capabilities, then produce a compiling Quartz skeleton, a single `_test.nox` file, and the initial JSON result contract. Add no HTTP load generator and no public-network access in the first PR. Report actual test output and unresolved API constraints. After Phase 0 is green, start Phase 1.
