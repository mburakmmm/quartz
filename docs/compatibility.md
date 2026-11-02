# Quartz Cross-Platform & Compiler Compatibility Matrix

> **Framework:** Quartz for NOX  
> **Status:** Production Reference  
> **Pinned Toolchain:** `nox-lang v2.0.108+`  
> **Target Package:** `quartz`

Quartz is tested and verified across modern operating systems, hardware architectures, and NOX compiler backends.

---

## 1. Operating System & Architecture Matrix

| Platform | Architecture | CI Status | Compiler Backend | Notes |
|---|---|---|---|---|
| **Windows** | `x86_64` | `VERIFIED / GREEN` | `QBE` & `LLVM` | Primary Windows testbed. Automatic fallback to QBE when Clang is not present on PATH. |
| **macOS** | `Apple Silicon (ARM64)` | `VERIFIED / GREEN` | `LLVM` & `QBE` | Developed by NOX author on M4 Mac. M:N work-stealing threading fully active. |
| **Linux** | `x86_64 (Ubuntu)` | `VERIFIED / GREEN` | `QBE` & `LLVM` | Standard cloud CI runner environment. |
| **Linux** | `ARM64 (Ubuntu)` | `VERIFIED / GREEN` | `QBE` & `LLVM` | Verified via GitHub Actions multi-arch runners. |

---

## 2. Compiler Backends: QBE vs. LLVM

The NOX compiler supports both QBE and LLVM backends. Quartz is carefully architected to run identically on both backends without behavioral drift:

| Feature / Subsystem | QBE Backend | LLVM Backend | Quartz Strategy |
|---|---|---|---|
| **Concurrency Scheduling** | Cooperative single-threaded fibers | M:N work-stealing multithreading | Quartz load generators avoid blocking `sleep_ms` in critical worker paths and use non-blocking pacing to prevent deadlocks. |
| **Channel Payloads** | Primitives (`int`, `str`) | Primitives & Structs | Channels use primitive payloads (or serialized message strings) to guarantee clean process exits and prevent destructor memory issues. |
| **Numeric Arithmetic** | IEEE-754 64-bit float | IEEE-754 64-bit float | Safe integer boundary guard ($2^{53}$) enforced consistently across both backends. |
| **Monotonic Clocks** | `instant_now()` nanoseconds | `instant_now()` nanoseconds | High-resolution duration measurement verified on both backends. |

---

## 3. Standard Library Module Requirements

Quartz relies strictly on standard library modules bundled with `noxc`:

- `nox.test`: Underlying test assertions and test runner integration.
- `nox.time`: High-precision monotonic timestamps (`instant_now()`).
- `nox.fs`: Atomic file writes, directory creation, artifact persistence.
- `nox.json`: JSON serialization and deserialization for reports and contracts.
- `nox.http`: In-process request/response structures and opt-in streaming network requests.
- `nox.process`: Subprocess execution and watchdog timeout orchestration.

---

## 4. Known Compiler Constraints & Design Patterns

1. **Class Generics & Inheritance:** In NOX 2.0.108, generic classes cannot inherit from base classes (`class Box(Base)[T]` is rejected). Quartz uses concrete typed generators (`IntGen`, `StringGen`, etc.) rather than a generic `Gen[T]` base class.
2. **Generic Methods:** Class methods cannot introduce their own generic type parameters (`def method[T]`). Quartz uses module-level generic functions or type-erased helper functions.
3. **Module-Level Imports:** `from ... import` statements must strictly reside at the module top level; nested imports within function bodies raise compilation errors.
4. **JSON Large Integers:** In `nox.json`, numbers are stored as 64-bit floating point numbers. Numbers exceeding $2^{53}$ ($9,007,199,254,740,992$) lose lower-bit precision. Contract schemas validate integer bounds using explicit guards.
5. **Path Separators:** Quartz uses forward slashes (`/`) for all internal path handling to ensure cross-platform compatibility between Windows and POSIX systems.
