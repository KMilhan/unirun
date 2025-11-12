# Unirun Recipes

These guides show practical, stdlib-fluent migrations to unirun v1.0. Each
recipe pairs a "before" snippet written against the raw standard library with an
"after" version that layers in `Run`, compat modules, and the relevant env
toggles. Use them as scaffolding when you need more than the high-level mapping
in the [Quick Reference](../quick-reference.md).

## Included guides

- [Thread pools](threads.md) — move `ThreadPoolExecutor` blocks to
  `Run(flavor="threads")`, `RuntimeConfig`, and helpers like `submit`, `map`,
  `to_thread`.
- [Process pools](processes.md) — convert CPU-bound calls to
  `Run(flavor="processes")`, keep compat downgrades, and surface
  `UNIRUN_FORCE_PROCESS` hints.
- [Asyncio](asyncio.md) — replace `asyncio.to_thread` / `run_in_executor` with
  `unirun.compat.asyncio`, then consolidate coroutine work inside a `Run`
  context.

## How to work with the recipes

1. Read the "Before" block to anchor yourself in the stdlib baseline.
2. Apply the "After" diff, adjusting `RuntimeConfig` or env toggles as needed.
3. Run the suggested validation command (usually `uv run pytest …`) to confirm
   behaviour locally; add `unirun.reset()` fixtures so tests start clean.

All commands assume `uv sync --extra benchmark` has been run and the virtual
environment activated.
