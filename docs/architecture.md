# Quartz Architecture & Technical Decision Records (ADRs)

> **Toolchain:** `noxc 2.0.108`  
> **Environment:** Windows x86_64 (Tested for ecosystem cross-platform compatibility)  
> **Date:** 2026-11-02  

---

## 1. System Overview

Quartz is designed as a standalone, zero-dependency testing framework for the NOX ecosystem. It provides property testing, structure-aware fuzzing, HTTP contract validation, and bounded load testing without relying on third-party application frameworks.

```text
                  +-----------------------------------+
                  |            quartz CLI             |
                  |          (cli/main.nox)           |
                  +-----------------+-----------------+
                                    |
                  +-----------------v-----------------+
                  |            quartz.nox             |
                  |          (public facade)          |
                  +-----------------+-----------------+
                                    |
          +-------------------------+-------------------------+
          |                         |                         |
+---------v---------+     +---------v---------+     +---------v---------+
|    quartz.core    |     |  quartz.property  |     |    quartz.fuzz    |
| runner, asserts,  |     | generators, PRNG, |     | mutations, corpus,|
| fixtures, reports |     | shrinker, replay  |     | watchdog harness  |
+---------+---------+     +-------------------+     +-------------------+
          |
+---------v---------+
|  quartz.contract  |
| HTTP / in-process |
| schema validator  |
+---------+---------+
          |
+---------v---------+
|    quartz.load    |
| bounded scheduler |
| latency histogram |
+-------------------+
```

---

## 2. Capability Matrix (Validated on Windows x86_64)

| Feature / Subsystem | Status | Verified Behavior & Notes |
|---|---|---|
| **Derleme Backend** | `QBE: VERIFIED`<br>`LLVM: UNVERIFIED (Needs Clang/Zig)` | On Windows without Clang/Zig on PATH, `noxc` automatically and successfully builds executables using the built-in QBE backend. |
| **Class & Inheritance** | `VERIFIED` | Classes, constructors, fields, and method overriding work reliably. |
| **Generics** | `PARTIAL / RESTRICTED` | Generic functions (`def id[T]`) and generic classes (`class Box[T]`) are supported, but generic classes cannot inherit from a base class (`err_class_generic_with_base`), and class methods cannot have generic parameters (`err_generic_method_rejected`). |
| **Callbacks** | `VERIFIED` | Function types `(A, B) -> C` can be passed as arguments and invoked indirectly. |
| **Exceptions** | `VERIFIED` | `try / except Exception as e / finally` executes `finally` on both success and error paths. Note: `except` requires imported class names (`from mod import Err`), dotted names are rejected. |
| **Concurrency** | `VERIFIED` | `spawn`, `await`, and `Channel[T]` operate under cooperative single-threaded fiber scheduling on QBE. Rate limiters must not rely on blocking `sleep_ms`. |
| **Monotonic Timing** | `VERIFIED` | `nox.time.instant_now().elapsed_ms()` measures monotonic elapsed time cleanly. |
| **File I/O** | `VERIFIED` | `nox.fs.write_string`, `write_string_atomic`, `read_to_string`, and directory operations work with standard `/` paths on Windows. |
| **JSON Parser** | `VERIFIED` | `nox.json.parse` parses into `JsonValue`. Numbers are IEEE-754 float64; integers > 2^53 lose lower-bit precision. |
| **Process Execution** | `VERIFIED` | `nox.process.Command` runs subprocesses, captures status/stdout/stderr, and correctly raises `ProcessError` when `set_timeout` is exceeded. |
| **Local Imports** | `VERIFIED` | Project-internal multi-file imports (`import core.types` or `from core.types import ...`) require `nox.json` in the project root. |

---

## 3. Architectural Decision Records (ADRs)

### ADR-001: Concrete Generator Implementations over Universal `Gen[T]` Hierarchy
- **Context:** `AGENTS.md` posed the open question of whether to use a typed `Gen[T]` hierarchy or concrete generators. Our technical probe revealed that in NOX 2.0.108:
  1. Generic classes cannot inherit from base classes (`class Box(Base)[T]` is rejected by the compiler).
  2. Methods cannot have their own generic type parameters (`def method[T]` is rejected).
- **Decision:** Implement explicit concrete generator classes (`IntGen`, `FloatGen`, `BoolGen`, `StringGen`, `ByteGen`, `ListGen`, `JsonGen`) with a common convention and type-erased container where necessary.
- **Consequences:** Type safety is preserved without fighting compiler monomorphization boundaries.

### ADR-002: Dedicated Explicit-State Quartz PRNG
- **Context:** `nox.random` provides per-worker state, but property-based testing and fuzzing require deterministic, reproducible input streams that can be replayed on any worker, machine, or operating system given identical `(root_seed, test_id, case_index, rng_version)`.
- **Decision:** Quartz Core and Property engines will use a dedicated explicit-state 32-bit / 64-bit PRNG (e.g. LCG/XorShift with golden vectors) that never touches the global random state.
- **Consequences:** Tests produce 100% deterministic results across Windows, macOS (M-series), and Linux.

### ADR-003: Manifest Requirement and `from ... import` Pattern
- **Context:** Without `nox.json`, `noxc` only resolves modules from `stdlib/nox/`. With `nox.json`, `noxc` resolves project-internal paths (e.g. `core/types.nox`). Furthermore, top-level constants like `STATUS_PASS` are accessed cleanest via `from core.types import STATUS_PASS` rather than attribute lookups on non-variable namespace identifiers.
- **Decision:** Maintain `nox.json` with valid `BinSpec` structure and use `from module import ...` for cross-file imports.

### ADR-004: Preservation Strategy for JSON Integers > 2^53 in Contract Testing
- **Context:** Probe 7 demonstrated that `9007199254740993` is parsed as `9007199254740992.0` due to IEEE-754 float conversion in `nox.json`.
- **Decision:** When evaluating contract bodies, integer identity fields exceeding 2^53 must be detected or validated via raw string matching or schema assertions, warning/failing rather than silently rounding precision.

### ADR-005: Cross-Platform Windows & macOS Scheduling Awareness
- **Context:** The NOX language author develops on macOS M4 (LLVM / M:N work-stealing / kqueue), while this test bed runs on Windows x86_64 (currently using QBE cooperative single-threaded fiber runtime).
- **Decision:** Quartz scheduling paths must strictly avoid blocking calls (`sleep_ms`) in workers and must structure channels and tasks so they run deterministically and safely on both single-threaded cooperative QBE and multi-threaded LLVM backends.
