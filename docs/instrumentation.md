# Instrumentation & Decision Traces

Unirun keeps observability hooks aligned with the stdlib. Every executor
decision produces a `DecisionTrace`, and you can capture it at the scope level,
stream it globally, or temporarily disable tracing the same way you would wrap
`ThreadPoolExecutor` in logging.

> Metrics note: upcoming work in issue #26 will expose `RunContext.metrics`
> alongside the trace API. Until then, traces are the primary insight surface.

## Capturing traces on a `Run` scope

`Run(trace=...)` exposes the scheduler decision used to lease the executor:

```python
from unirun import DecisionTrace, Run

# 1. Boolean capture – store the last decision on the scope.
scope = Run(flavor="auto", trace=True)
with scope:
    pass
print(scope.trace.as_dict())

# 2. Callback – stream each decision as it happens.
with Run(trace=lambda trace: print(trace.resolved_mode)):
    pass

# 3. Preallocated sink – reuse a DecisionTrace object to avoid allocations.
sink = DecisionTrace(mode="", hints={}, resolved_mode="", reason="")
with Run(trace=sink):
    pass
print(sink.reason)
```

`DecisionTrace` lives in `unirun.scheduler` and records the requested mode,
resolved backend, reason string (including fallbacks), thread mode,
`max_workers`, and any hints that influenced the decision.

Environment nudges such as `UNIRUN_FORCE_THREADS`, `UNIRUN_FORCE_PROCESS`, and
`UNIRUN_THREAD_MODE` surface directly in the trace so you can confirm how env
overrides interacted with the scheduler.

## Streaming scheduler decisions globally

### `add_decision_listener` / `remove_decision_listener`

Register a callback to receive every `DecisionTrace`, regardless of whether the
executor came from `Run` or a compat import:

```python
from __future__ import annotations

import json
import logging

from unirun import Run, add_decision_listener, remove_decision_listener

logger = logging.getLogger("unirun.decisions")


def emit(trace):
    payload = trace.as_dict()
    logger.info("scheduler decision", extra={"unirun.trace": payload})
    print(json.dumps(payload))

add_decision_listener(emit)
try:
    with Run(flavor="auto"):
        pass
finally:
    remove_decision_listener(emit)
```

Listeners run synchronously—keep handlers fast or hand the payload off to a
queue. Combine multiple listeners to target both structured events and human
logs.

### `observe_decisions()` shortcut

For one-off debugging sessions, wrap the code with `observe_decisions()` to log
every trace via `logging.getLogger("unirun.scheduler")` (or a custom logger and
level):

```python
from unirun import Run, observe_decisions

with observe_decisions(level=20):  # INFO
    with Run(flavor="interpreters"):
        pass
```

Use `last_decision()` when you just need the most recent trace from the global
scheduler (e.g., inside tests where you assert on fallback reasons).

## Toggling / disabling traces

- Set `trace=None` (default) when you do not need per-scope capture.
- Unregister listeners (`remove_decision_listener`) before shutting down long
  running services so exit handlers stay fast.
- Environment variables such as `UNIRUN_FORCE_THREADS`,
  `UNIRUN_FORCE_PROCESS`, `UNIRUN_THREAD_MODE`, and
  `UNIRUN_PREFERS_SUBINTERPRETERS` influence the scheduler and show up inside
  each `DecisionTrace`.

## Runnable example

```python
"""Log scheduler decisions during a batch job."""

from __future__ import annotations

import logging
from unirun import Run, RuntimeConfig, add_decision_listener, remove_decision_listener
from unirun.workloads import count_primes

logging.basicConfig(level=logging.INFO)


def main() -> None:
    def trace_logger(trace):
        logging.info("resolved=%s reason=%s", trace.resolved_mode, trace.reason)

    add_decision_listener(trace_logger)
    try:
        config = RuntimeConfig(cpu_bound=True, force_process=True)
        with Run(flavor="processes", config=config) as executor:
            future = executor.submit(count_primes, 200_000)
            print(f"primes: {future.result()}")
    finally:
        remove_decision_listener(trace_logger)


if __name__ == "__main__":
    main()
```

Run it with `uv run python docs/examples/observe.py` (or similar) to watch the
decision stream as you experiment with env toggles.
