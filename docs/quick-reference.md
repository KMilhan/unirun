# Quick Reference

Use this cheat sheet to map unirun surfaces to their standard-library
counterparts, highlight the most common overrides, and understand how compat
modules behave in managed versus passthrough modes.

## Run Flavors

| `Run(flavor=...)` | Returns stdlib executor | When to prefer it | Key overrides / env toggles |
| --- | --- | --- | --- |
| `"auto"` (default) | Scheduler-chosen executor (threads, processes, or interpreters) | Let capabilities pick a backend based on workload hints | `RuntimeConfig` or env toggles (`UNIRUN_FORCE_THREADS`, `UNIRUN_FORCE_PROCESS`, `UNIRUN_THREAD_MODE`, `UNIRUN_MAX_WORKERS`) steer decisions |
| `"threads"` | `concurrent.futures.ThreadPoolExecutor` | IO-bound or mixed workloads where threads suffice | `UNIRUN_THREAD_MODE=auto|gil|nogil`, `max_workers`, `name` (thread prefix) |
| `"processes"` | `concurrent.futures.ProcessPoolExecutor` | CPU-bound work under the GIL | `UNIRUN_FORCE_PROCESS=1`, `max_workers` |
| `"interpreters"` | `unirun.executors.subinterpreter.SubInterpreterExecutor` → falls back to threads if unsupported | Free-threaded builds with PEP 554 interpreters | `prefers_subinterpreters` hint, fallback emits warnings |
| `"none"` | Shared thread pool (singleton) | Opt out of automation; reuse the global thread executor | Only `max_workers`/`name` apply |

Notes:
- Set `RuntimeConfig(thread_mode="nogil")` or `UNIRUN_THREAD_MODE=nogil` to pin nogil-aware thread pools when running on free-threaded interpreters.
- `Run` reuses parent executors when flavors and overrides align, minimizing pool churn.
- `trace=True` (or a callback) surfaces decision metadata for observability.

## Compat – `unirun.compat.concurrent.futures`

Managed compat re-exports stdlib names but routes constructors through the scheduler. Passthrough mode (`UNIRUN_COMPAT_MODE=passthrough`) forwards directly to the stdlib.

| Compat export | Stdlib analogue | Managed-mode notes |
| --- | --- | --- |
| `ThreadPoolExecutor` | `concurrent.futures.ThreadPoolExecutor` | Picks scheduler-backed pools; `UNIRUN_THREAD_MODE` controls worker naming and nogil behaviour |
| `ProcessPoolExecutor` | `concurrent.futures.ProcessPoolExecutor` | Respects scheduler sizing / fallbacks; downgrades emit `RuntimeWarning` |
| `Executor`, `Future`, `TimeoutError`, `CancelledError`, `ALL_COMPLETED`, `FIRST_*`, `wait`, `as_completed` | Identical stdlib symbols | Behaviour unchanged; helpers operate on managed executors |

Downgrade warnings call out when managed compat must fall back (e.g., interpreters unavailable). Set `UNIRUN_COMPAT_MODE=managed` (default) for scheduler integration.

## Compat – `unirun.compat.asyncio`

| Compat helper | Stdlib source | Managed-mode behaviour |
| --- | --- | --- |
| `to_thread` | `asyncio.to_thread` | Uses `unirun.to_thread` with scheduler sizing; honours thread-mode toggles |
| `run_in_executor` | `asyncio.AbstractEventLoop.run_in_executor` | When `executor is None`, opens `Run(flavor="auto")` to back tasks |
| `gather`, `TaskGroup`, `sleep`, `run`, `get_event_loop`, `get_running_loop`, `new_event_loop`, `Future`, `Task`, `CancelledError` | Direct stdlib re-export | Semantics unchanged; passthrough just defers to stdlib |

Passthrough vs managed is controlled by `UNIRUN_COMPAT_MODE`. Managed mode uses the scheduler for implicit executors while leaving explicit executors untouched.

## Environment Toggles

| Variable | Effect |
| --- | --- |
| `UNIRUN_THREAD_MODE=auto|gil|nogil` | Pins how thread pools behave across GIL vs nogil interpreters |
| `UNIRUN_FORCE_THREADS=1` / `UNIRUN_FORCE_PROCESS=1` | Force scheduler decisions toward threads or processes |
| `UNIRUN_COMPAT_MODE=managed|passthrough` | Controls whether compat modules delegate to the scheduler |
| `UNIRUN_MAX_WORKERS`, `UNIRUN_PREFERS_SUBINTERPRETERS` | Influence sizing and interpreter preference |

See `RuntimeConfig.from_env()` for the full list of supported overrides.
