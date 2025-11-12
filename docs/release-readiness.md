# Release Readiness Checklist

Use this list to confirm v1.0 is truly ship-ready. Each section links to the
source-of-truth doc so the checklist stays short and enforceable.

## Documentation parity
- [ ] README quick start + release notes reflect the current `Run`/compat
      surface (see [`README.md`](../README.md)).
- [ ] Architecture commitments match the implemented API
      (see [`ARCHITECTURE.md`](../ARCHITECTURE.md)).
- [ ] Quick Reference, Recipes, and Instrumentation guides are updated together:
  - [`docs/quick-reference.md`](docs/quick-reference.md)
  - [`docs/recipes/README.md`](docs/recipes/README.md)
  - [`docs/instrumentation.md`](docs/instrumentation.md)

## Packaging & automation
- [ ] `pyproject.toml` metadata (classifiers, description, version source) is
      up-to-date.
- [ ] `unirun/__init__.py` exports (`__all__`) include all public helpers.
- [ ] Release workflow [`release.yml`](../.github/workflows/release.yml) verified:
  - `v*-rc*` tags publish to TestPyPI via OIDC.
  - Stable `v*` tags publish to PyPI.
  - `workflow_dispatch` (no tag) runs a build-only dry run; artifact stored for
    review.

## Verification
- [ ] Test matrix executed across CPython 3.11–3.14 (GIL + nogil) with results
      attached to the release notes.
- [ ] CI workflow [`test-matrix.yml`](../.github/workflows/test-matrix.yml)
      exercised (Python 3.11–3.14). Trigger via *Actions → Test Matrix → Run
      workflow* or push a `v*` tag; grab artifacts named
      `pytest-python-<version>` for evidence.
- [ ] Benchmarks captured before/after the final change set (`python 3.14`).
- [ ] Benchmark workflow [`bench.yml`](../.github/workflows/bench.yml)
      triggered via *Actions → Bench CI → Run workflow* (or `v*` tag); JSON
      artifact written to `bench/bench.json` and uploaded as `bench-results`.
- [ ] Pytest asyncio configuration locked to `asyncio_mode=auto` (see
      [`pytest.ini`](../pytest.ini)).

## Release process
- [ ] Candidate tag (e.g., `v1.0.0-rc1`) created; burn-in results recorded.
- [ ] Final release notes summarize doc updates, migration guidance, and
      verification artifacts.
- [ ] Outstanding follow-ups filed as separate issues before closing
      [`#16`](https://github.com/KMilhan/unirun/issues/16).
