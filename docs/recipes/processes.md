# Process Pools with `Run(flavor="processes")`

## Context

CPU-bound workloads often bounce between `ProcessPoolExecutor` and ad-hoc
thread fallbacks when fork/spawn fails. `Run(flavor="processes")` keeps the
stdlib API but adds deterministic downgrades, and compat imports spread the same
behaviour through the codebase.

## Before (stdlib)

```python
from concurrent.futures import ProcessPoolExecutor, ThreadPoolExecutor

def crunch(payloads):
    try:
        with ProcessPoolExecutor(max_workers=4) as pool:
            return list(pool.map(expensive_fn, payloads))
    except OSError:
        with ThreadPoolExecutor(max_workers=4) as fallback:
            return list(fallback.map(expensive_fn, payloads))
```

## After (unirun)

```python
from unirun import Run, RuntimeConfig, map, reset, to_process
from unirun.compat.concurrent import futures as compat_futures

def crunch(payloads):
    config = RuntimeConfig(cpu_bound=True, force_process=True, max_workers=4)
    with Run(flavor="processes", config=config, trace=True) as executor:
        return list(map(executor, expensive_fn, payloads))

def crunch_via_compat(payloads):
    with compat_futures.ProcessPoolExecutor(max_workers=4) as pool:
        return list(pool.map(expensive_fn, payloads))

async def crunch_async(payloads):
    return await asyncio.gather(*(to_process(expensive_fn, payload) for payload in payloads))

def teardown():
    reset()
```

```diff
-with ProcessPoolExecutor(max_workers=4) as pool:
-    return list(pool.map(expensive_fn, payloads))
+with Run(flavor="processes", config=config) as executor:
+    return list(map(executor, expensive_fn, payloads))
```

### Env toggles

- `UNIRUN_FORCE_PROCESS=1` — globally prefer process pools.
- `UNIRUN_COMPAT_MODE=managed|passthrough` — decide whether compat constructors
  route through the scheduler.
- `UNIRUN_CPU_BOUND=1` — hint that heavy payloads deserve processes when
  flavor="auto".

Managed compat emits `RuntimeWarning` when downgrading to threads so you can
trace decisions.

## Validate

```bash
UNIRUN_FORCE_PROCESS=1 uv run pytest tests/test_process_executor.py -k map
UNIRUN_COMPAT_MODE=passthrough uv run pytest tests/test_process_executor.py -k compat
```

Remember to call `unirun.reset()` between tests so cached pools do not bleed
across scenarios.
