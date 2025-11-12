# Repository Guidelines

## Golden Rule
Every interface, document, and helper must describe concurrency in the language
Python developers already know from the standard library (`concurrent.futures`,
`asyncio`, `multiprocessing`). We are not inventing a new concurrency model—just
enhancing stdlib ergonomics. Avoid new nouns or surface syntax when threads,
processes, sub-interpreters, or free-threaded runtimes can be expressed with
existing terminology.

## Project Structure & Module Organization
Core logic lives in `src/unirun/`: `run.py` and `scheduler.py` coordinate executor selection, `capabilities.py` captures interpreter traits, and `workloads.py` supplies deterministic helpers. The optional CLI wrapper is in `src/unirun_bench/` (`__main__.py`, `cli.py`) so it can ship independently. Tests reside in `tests/` with discovery driven by pytest; reuse the existing `test_unirun.py` layout when extending coverage. Packaging metadata and Hatch build targets stay in `pyproject.toml`.

## Build, Test, and Development Commands
- `uv venv --python 3.14 && source .venv/bin/activate`: create the local environment using uv (adjust Python version as needed).
- `uv sync --extra benchmark`: install editable dependencies plus the benchmark extra.
- `make lint-check`: run `ruff` and `ty` via uv to enforce lint/type gates (mirrors the pre-commit hook).
- `uv run pytest`: execute the full regression suite (pytest auto-discovers unittest cases too).
- `uv run scripts/update_compat_parity.py`: regenerate the compat API baseline when stdlib exports change.
- `uv run python -m unirun_bench --profile all --samples 5 --json`: optional benchmark sweep for manual verification.
- `make contract-versions`: run the CPython `test_asyncio`/`test_concurrent_futures` suites against the compat layer for Python 3.11–3.14 (downloads sources if needed).
- **Always rerun `make lint-check` and `uv run pytest` before committing or pushing.** Hooks execute the lint gate automatically, but running these locally catches regressions faster and keeps CI green.

## Coding Style & Naming Conventions
Follow PEP 8 defaults: 4-space indentation, soft wrap near 100 characters, module-level constants in UPPER_SNAKE_CASE. Public APIs must expose explicit type hints (Literal unions, Protocols) that reuse standard-library names. Keep docstrings concise and situational, prefer `snake_case` for functions, and reserve `PascalCase` for dataclasses or capability records. When extending executors, keep thread names aligned with the `unirun-*` prefix so logs remain searchable. Anchor surface names and docstrings in standard-library vocabulary (`Executor`, `Future`, `to_thread`); call out when behavior is a drop-in replacement with optional enhancements.
As an agent, you keep the ratio of comment to code, 30 to 70. Follow Google Python Style Guide for docstrings and comments, keep EN-us spelling.

## Testing Guidelines
Prefer pytest with function-only tests (`test_<feature>`) under `tests/`. Use fixtures for setup instead of classes; when touching executor state, call `unirun.reset()` in a fixture or `finally` block. Reuse workloads from `unirun.workloads` to maintain deterministic timing and cover both auto and forced executor modes. Existing unittest-based suites continue to run via pytest—adapt or replace them incrementally.

Organize the suite so that each test module covers exactly one concurrency or parallelism feature. Companion parity checks with the CPython stdlib belong in files that share the feature name and end with `_double.py` (for example, `test_thread_executor.py` and `test_thread_executor_double.py`).

## Security & Configuration Tips
- Runtime behaviour is controlled through environment variables; keep them in `.env` files or CI secrets rather than committing overrides. Common toggles are `UNIRUN_THREAD_MODE`, `UNIRUN_FORCE_THREADS`, `UNIRUN_FORCE_PROCESS`, `UNIRUN_COMPAT_MODE`, and `UNIRUN_COMPAT_SITE`.
- Use passthrough mode (`UNIRUN_COMPAT_MODE=passthrough`) when you need stdlib-complete fallback behaviour; document why you enabled it and reset to `managed` once the investigation completes.
- Benchmarks and compat contract downloads write into `dist/` or `badges/`. Do not point those scripts at shared locations or commit generated tarballs.
- The benchmark CLI and compat tooling assume no external services; avoid adding new runtime dependencies without guarding imports so the core package remains dependency-free.
- Never hard-code interpreter paths, access tokens, or private URLs in source or tests. If a command needs credentials (for example, release scripts), require them via environment variables and document the expectation in `docs/release-checklist.md`.

## Commit & Pull Request Guidelines
- Keep commits atomic: each commit should capture a cohesive, reviewable change. Break larger features into logical commits (code, tests, docs, tooling) rather than batching unrelated edits. This matches the workflow request in this session.
- When a commit resolves a GitHub issue, mention the issue number in the commit message body (for example, `closes #171`).

- Pull request titles can be as descriptive as needed; no enforced character limit.
- Commit subjects must begin with a gitmoji shortcode (e.g., `:sparkles:`) and may not use raw Unicode emojis or shorthand such as `:feat`. Follow the gitmoji with a single space and an imperative summary (e.g., `:sparkles: add interpreter executor docs`).
- Reference related issues in the body, including reproduction or benchmark notes when concurrency paths change.
- Pull requests should summarize user-facing impact, list test commands executed (include `pytest` runs), and attach before/after numbers for performance tweaks. Add screenshots or JSON excerpts only when the CLI output changes to preserve review context.
## Git Hooks
- Run `git config core.hooksPath githooks` once so the repository-managed hooks take effect.
- The `githooks/pre-commit` script shells out to `make lint-check`, ensuring `uv run ruff check .` and `uv run ty check .` succeed before commits land.
## Benchmark & Performance Notes
`unirun_bench` is optional but ideal for stress-testing new heuristics. Run the command above after altering executor defaults and share key JSON metrics in PR discussions. Avoid adding runtime dependencies; if a benchmark needs extras, guard imports so the core package stays dependency-free.

## Milestones & Labels Policy
- Milestones track roadmap phases defined in `docs/milestones.md`; during the first
  phase, map qualifying issues to **Executor Surface Alignment**. Advance through
  the remaining phases in order (Scheduler Enhancements → Compat Layer Delivery →
  Instrumentation & Observability → Verification Upgrades → Release Readiness) so
  burndown charts match the published timeline.
- Only attach a milestone when the issue directly contributes to that phase’s
  acceptance criteria. Cross-cutting chores stay unmilestoned unless they unblock
  the active phase.
- Labels follow a `group:value` format so queries stay predictable:
  - `area:*` — surfaced module or subsystem (`area:scheduler`, `area:compat`,
    `area:docs`).
  - `type:*` — work category (`type:bug`, `type:feature`, `type:docs`,
    `type:infra`).
  - `status:*` — workflow hints (`status:needs-triage`, `status:blocker`,
    `status:ready`).
  - `priority:p{1-3}` — urgency indicator aligned with roadmap risk.
- Every newly created issue should get one `area:*` and one `type:*` label during
  triage; add `status:*` and `priority:*` as soon as ownership is clear. Update
  labels whenever scope shifts so dashboards stay trustworthy.

## Issue-Driven Workflow (Loop)
- Default GitHub CLI target is `KMilhan/unirun`. Run `gh repo set-default KMilhan/unirun` once so `gh issue` and `gh pr` commands apply to this repository.
- Maintain the status labels `status:ready`, `status:in-progress`, `status:blocked`, and `status:review`. Create any missing label via `gh label create "status:<name>" -R KMilhan/unirun --color 0366d6`.
- Pull the next task from open, non-epic issues that already have `status:ready` plus `area:*` and `type:*`. Prefer higher `priority:p1 → p3` and older update timestamps when multiple issues qualify. Example query: `gh issue list -R KMilhan/unirun -l "status:ready" -l "type:feature" -s open --limit 5`.
- Starting work: `gh issue edit <n> -R KMilhan/unirun --add-label "status:in-progress" --remove-label "status:ready"` and leave a short comment outlining intended steps. Reconfirm milestone alignment with `docs/milestones.md` while triaging.
- Implementation loop: keep commits atomic, use gitmoji shortcodes (e.g., `:sparkles:`) with imperative subjects, mention related issues plus benchmark/test commands in the commit body, and push frequently (`git push origin HEAD`).
- After verifying lint + tests, close the issue with context: `gh issue close <n> -R KMilhan/unirun -c "Completed in <sha/url>"`. Reapply `status:ready` only if additional follow-up remains.

## One-Phrase Command: "start work"
- Saying **“start work”** (optionally `start work #<n>`) triggers the automation below. Skip manual prep unless you need to override a specific issue number.
  1. Ensure `gh` points at this repo and all status labels exist (`status:ready|in-progress|blocked|review`).
  2. Select the highest-priority open issue following the workflow rules above (prefer `status:ready`, then `priority:p1`, then `type:feature`, else oldest open). `start work #<n>` pins to a particular issue.
  3. Transition the issue to in-progress (`gh issue edit …`) and comment with the immediate plan (tests to run, files to touch, benchmark expectations).
  4. Implement and push: make the minimal change, commit with gitmoji shortcode + imperative subject, include rationale/tests/issue references in the body, and push to `main`.
  5. Close the issue via `gh issue close <n> -R KMilhan/unirun -c "Done in <sha/url>"` once the change lands.
  6. Stop after one issue unless you explicitly request “start work continuously,” which loops back to step 2.
- **preview start** reports the candidate issue and intended steps without editing GitHub state.
- **pause work** reassigns `status:ready`, removes `status:in-progress`, and comments why work stopped.
- **next** closes the current issue (if complete) and immediately selects the next ready issue.
