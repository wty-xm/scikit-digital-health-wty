# Repository Guidelines

## Project Structure & Module Organization
- `src/skdh/` contains the core Python package.
- `tests/` holds pytest suites organized by domain (e.g., `tests/activity/`, `tests/sleep/`) with files named `test_*.py`.
- `docs/` contains Sphinx documentation; built HTML lands in `docs/_build/html/`.
- `examples/` provides notebooks and sample data.
- `build_scripts/` includes packaging helpers (e.g., `build_scripts/dist.sh`).
- `docs-wty/` contains fork-specific notes (example: `docs-wty/skdh_sleep_overview.md`).

## Build, Test, and Development Commands
- `pip install scikit-digital-health[dev]` installs dev/test dependencies (pytest, coverage, etc.).
- `pytest tests/` runs the unit test suite.
- `coverage run -m pytest && coverage html` generates HTML coverage in `coverage/`.
- `flake8 src/` checks PEP 8 style, complexity, and line length.
- `python -m build` builds wheel and sdist (uses Meson; requires a C/Fortran compiler).
- `cd docs && make html` builds Sphinx docs to `docs/_build/html/`.

## Coding Style & Naming Conventions
- Follow PEP 8: no tabs, remove trailing whitespace.
- Line length is 80 characters (see `.flake8`).
- Use descriptive branch names (example: `add-new-gait-metrics`).
- Keep tests in the relevant domain folder under `tests/` and name them `test_*.py`.

## Testing Guidelines
- All code changes should include or update tests.
- Prefer tests that fail before the fix and pass after it.
- Aim for 100% statement coverage where practical; use the coverage command above.
- Run tests locally before pushing to avoid CI churn.

## Commit & Pull Request Guidelines
- Commit often with clear, descriptive messages (no specific format is documented).
- PR titles and descriptions should be clear, concise, and self-explanatory.
- Reference issues in PR comments with `xref gh-1234`; close issues with `closes gh-1234`.
- For significant changes, add a `changes/<branch-name>.md` summary file.
