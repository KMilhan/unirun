# Thread Pools with `Run(flavor="threads")`

## Context

IO-heavy services often wrap work inside
`concurrent.futures.ThreadPoolExecutor`. `Run(flavor="threads")` keeps that
surface while adding scheduler-aware sizing, env overrides, and cooperative
usage from async code (`to_thread`).

## Before (stdlib)

```python
from concurrent.futures import ThreadPoolExecutor

def fetch_batch(tasks):
    with ThreadPoolExecutor(max_workers=8, thread_name_prefix="service-io") as pool:
        futures = [pool.submit(call_api, task) for task in tasks]
        return [future.result() for future in futures]

async def fetch_eventually(task):
    loop = asyncio.get_running_loop()
    return await loop.run_in_executor(None, call_api, task)
```

## After (unirun)

```python
from unirun import Run, RuntimeConfig, submit, map, to_thread

def fetch_batch(tasks):
    config = RuntimeConfig(thread_mode="nogil", max_workers=8)
    with Run(flavor="threads", config=config, name="unirun-io") as executor:
        futures = [submit(executor, call_api, task) for task in tasks]
        return [future.result() for future in futures]

async def fetch_eventually(task):
    return await to_thread(call_api, task)
```

```diff
-with ThreadPoolExecutor(max_workers=8) as pool:
-    future = pool.submit(call_api, task)
-    return future.result()
+with Run(flavor="threads", config=config) as executor:
+    future = submit(executor, call_api, task)
+    return future.result()
```

### Env toggles

- `UNIRUN_THREAD_MODE=auto|gil|nogil` — pin nogil-aware pools without touching
  code.
- `UNIRUN_FORCE_THREADS=1` — ensures `Run(flavor="auto")` never selects
  processes/interpreters.
- `UNIRUN_MAX_WORKERS=<n>` — cap shared pools when defaults are too large.

Combine toggles with `RuntimeConfig.from_env()` to keep CI and prod in sync.

## Validate

```bash
uv run pytest tests/test_thread_executor.py -k thread_pool --maxfail=1
```

Add a module fixture that calls `unirun.reset()` after each test to drop cached
executors.
