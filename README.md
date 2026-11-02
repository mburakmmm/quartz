# Quartz — Test Framework for the NOX Ecosystem

Quartz is a production-grade test framework designed specifically for applications, libraries, HTTP APIs, and services written in the [NOX programming language](https://github.com/mburakmmm/nox-lang).

## Core Capabilities

- **Quartz Core** — High-precision assertions, test suites, non-raising checks, fixture lifecycle (`try/finally`), and machine-readable reports (JSONL, summary JSON, JUnit XML).
- **Quartz Property** — Composable bounded generators, explicit-state deterministic PRNG, property invariant runner, and deterministic counterexample shrinking.
- **Quartz Fuzz** — Byte and structure-aware mutation fuzzing, corpus persistence, crash deduplication, and watchdog out-of-process harness.
- **Quartz Contract** — Transport-neutral HTTP contracts, in-process handler verification, and schema validation.
- **Quartz Load** — Safe bounded local load testing, concurrency controls, monotonic latency histograms, and threshold gates.

## Project Structure

```text
quartz/
  nox.json             # Package manifest (name, entry, bin)
  nox.lock             # Dependency lockfile
  quartz.nox           # Public library facade
  core/                # Test runner, assertion, fixtures, types
  property/            # Property generators, runner, shrinking
  fuzz/                # Fuzzing mutations, harness, corpus
  contract/            # HTTP contracts and schema evaluation
  load/                # Bounded workload scheduler and metrics
  adapters/            # Ecosystem adapters (Nyx, Aether)
  cli/                 # Installable CLI entry point
  tests/               # Test suites executed via noxc test
  docs/                # Architecture records and specifications
```

## Running Tests

Standard workflow using the NOX toolchain:

```powershell
# Type check the project
noxc check quartz.nox

# Run test suite
noxc test

# Run CLI entry point
noxc run cli/main.nox
```

## License

MIT License. See `LICENSE` for details.
