# Quartz Cross-Platform & Compiler Compatibility Matrix

> **Framework:** Quartz for NOX  
> **Status:** Production Reference  
> **Pinned Toolchain:** `nox-lang v2.1.186+`  
> **Target Package:** `quartz`

Quartz is tested and verified across modern operating systems, hardware architectures, and NOX compiler backends.

---

## 1. Operating System & Architecture Matrix

| Platform | Architecture | CI Status | Compiler Backend | Notes |
|---|---|---|---|---|
| **Windows** | `x86_64` | `VERIFIED / GREEN` | `QBE` (Primary, CI) / `LLVM` (Optional) | Primary Windows development testbed. Uses QBE by default (no Clang required). CI runs QBE for Windows; Linux and macOS run both QBE and LLVM. |
| **macOS** | `Apple Silicon (ARM64)` | `VERIFIED / GREEN` | `LLVM` & `QBE` | Developed by NOX author on M4 Mac. M:N work-stealing threading fully active. |
| **Linux** | `x86_64 (Ubuntu)` | `VERIFIED / GREEN` | `QBE` & `LLVM` | Standard cloud CI runner environment. |
| **Linux** | `ARM64 (Ubuntu)` | `VERIFIED / GREEN` | `QBE` & `LLVM` | Verified via GitHub Actions multi-arch runners. |

---

## 2. Compiler Backends: QBE vs. LLVM

The NOX compiler supports both QBE and LLVM backends. Quartz is carefully architected to run identically on both backends without behavioral drift:

| Feature / Subsystem | QBE Backend | LLVM Backend | Quartz Strategy |
|---|---|---|---|
| **Concurrency Scheduling** | Cooperative single-threaded fibers + isolated OS threads (`nox.thread`) | Shared work-stealing M:N scheduler across `spawn`, `TaskGroup`, `nox.thread` (`--release`) | Quartz load generators support both coroutine waves, OS worker waves, and cooperative `TaskGroupRunner` with `nox.tasks.is_cancelled()`. |
| **Channel Payloads** | Primitives (`int`, `str`) | Primitives & Structs | Channels and shared config cells use primitives or `nox.atomic.AtomicRef` byte snapshots to guarantee portability across both backends. |
| **Numeric Arithmetic** | IEEE-754 64-bit float + exact `i64` JSON | IEEE-754 64-bit float + exact `i64` JSON | Safe $2^{53}$ guard preserved in `schema_integer()`, while `schema_int64()` leverages `nox.json.as_int()` for exact 64-bit integers. |
| **Monotonic Clocks** | `instant_now()` & `monotonic_ns()` | `instant_now()` & `monotonic_ns()` | Sub-millisecond (`us` / `ns`) and millisecond latency measurement verified identically on both backends. |

---

## 3. Standard Library Module Requirements

Quartz relies strictly on standard library modules bundled with `noxc 2.1.186+`:

- **Core Portability (`quartz.nox`):** `nox.test`, `nox.time` (`instant_now`, `monotonic_ns`), `nox.fs` (`write_string_atomic`, `create_dir_all`, `remove_dir_all`), `nox.path` (`safe_join`), `nox.tempfile`, `nox.glob`, `nox.fnmatch`, `nox.mem` (`stats`), `nox.json` (`as_int`, tape parser), `nox.regex`, `nox.crypto` (`blake3`, `sha256`), `nox.http` (`set_http2`, `set_http3`, `client_pool`, `HttpCapabilityError`), `nox.grpc`, `nox.grpc_server`, `nox.protobuf`, `nox.tasks` (`TaskGroup`, `checkpoint`), `nox.atomic` (`AtomicInt`, `AtomicRef`), `nox.property`, `nox.process`.
- **Opt-In Desktop Accelerators (`accel/`):** `nox.yyjson`, `nox.pcre2`, and `nox.xxhash` (backed by `lib/noxclibs.a`). Kept deliberately separate from `quartz.nox` so core builds remain free of `noxclibs` links on WebAssembly, mobile, or `--deny-capability dynamic_code` targets.

---

## 4. Known Compiler Constraints & Design Patterns

1. **Class Generics & Inheritance:** Generic classes cannot inherit from base classes (`class Box(Base)[T]` is rejected). Quartz uses concrete typed generators (`IntGen`, `StringGen`, etc.) rather than a generic `Gen[T]` base class.
2. **Generic Methods:** Class methods cannot introduce their own generic type parameters (`def method[T]`). Quartz uses module-level generic functions or type-erased helper functions.
3. **Module-Level Imports:** Single-line `from ... import ...` statements must strictly reside at the module top level; nested imports and parenthesized multi-line `from` imports raise syntax/compilation errors.
4. **JSON Large Integers:** In `nox.json`, `as_int()` and `number_text()` preserve full exact 64-bit integers (`> 2^53`). Quartz's `schema_integer()` retains the strict $2^{53}$ guard for JS-compatible JSON contracts, while `schema_int64()` validates exact 64-bit integers.
5. **Path Separators & Confinement:** Quartz uses forward slashes (`/`) for internal path handling and `nox.path.safe_join()` for snapshots and fuzz corpus directories to prevent path traversal across Windows and POSIX systems.
6. **Package Consumption via `requires[]`:** External NOX packages declare `"alias": "quartz"` in `nox.json`. In `noxc 2.1.186`, leaf modules (`quartz.core.assert`, `quartz.core.types`, `quartz.core.prng`, `quartz.core.fixture`, `quartz.contract.types`, `quartz.accel.pcre2_assert`, `quartz.accel.xxhash_digest`) import cleanly directly via `from quartz.<module> import ...` in consumer projects; multi-module test suites run natively inside the repository or via the global `quartz` CLI (`noxc install github.com/mburakmmm/quartz`).
