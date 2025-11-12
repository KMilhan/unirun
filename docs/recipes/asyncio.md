# Asyncio + Compat

## Context

Coroutines usually lean on `asyncio.to_thread` or `loop.run_in_executor`. The
compat modules mirror these helpers but let the scheduler size pools, and `Run`
can wrap the entire async workload once you are ready for explicit scopes.

## Before (stdlib)

```python
import asyncio

async def hydrate(ids):
    loop = asyncio.get_running_loop()
    models = await asyncio.gather(
        *(loop.run_in_executor(None, load_model, ident) for ident in ids)
    )
    return await asyncio.gather(
        *(asyncio.to_thread(enrich_model, model) for model in models)
    )
```

## After (compat + Run)

```python
from unirun import Run, RuntimeConfig
from unirun.compat import asyncio as compat_asyncio

async def hydrate(ids):
    models = await compat_asyncio.gather(
        *(compat_asyncio.run_in_executor(None, load_model, ident) for ident in ids)
    )
    return await compat_asyncio.gather(
        *(compat_asyncio.to_thread(enrich_model, model) for model in models)
    )

async def hydrate_with_run(ids):
    config = RuntimeConfig(io_bound=True, thread_mode="gil")
    async with Run(flavor="threads", config=config) as executor:
        models = await compat_asyncio.gather(
            *(compat_asyncio.get_running_loop().run_in_executor(executor, load_model, ident)
              for ident in ids)
        )
        return await compat_asyncio.gather(
            *(compat_asyncio.to_thread(enrich_model, model) for model in models)
        )
```

```diff
-loop.run_in_executor(None, load_model, ident)
+compat_asyncio.run_in_executor(None, load_model, ident)
+# ...later
+async with Run(flavor="threads", config=config) as executor:
+    compat_asyncio.get_running_loop().run_in_executor(executor, load_model, ident)
```

### Env toggles

- `UNIRUN_COMPAT_MODE=managed|passthrough` — choose whether compat resolves to
  scheduler-backed pools.
- `UNIRUN_THREAD_MODE=gil|nogil` — control thread behaviour on nogil builds.
- `UNIRUN_FORCE_THREADS=1` — keep coroutine work on threads even when
  `Run(flavor="auto")` might jump to processes.

Expose these through `.env` files to keep parity between local runs and CI.

## Validate

```bash
uv run pytest tests/test_async_bridge.py -k run_in_executor
```

Use pytest fixtures that call `unirun.reset()` so every test sees a clean
executor cache.
