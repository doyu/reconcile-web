# Repository Guidelines

## Project Structure & Module Organization

This is an nbdev-based Python package. Edit source notebooks in `nbs/`; the files under `reconcile_web/` are generated outputs and should not be treated as the source of truth. `nbs/00_archive.ipynb` owns archive parsing and safe file access, while `nbs/01_app.ipynb` owns the FastHTML app and auth flow. `README.md` and documentation output are generated from notebooks. Keep implementation notes and design records under `docs/superpowers/`.

## Build, Test, and Development Commands

- `pip install -e .`: install the package in editable mode.
- `source .venv/bin/activate`: use the local environment when available.
- `nbdev-prepare`: export notebooks, clean generated files, and run nbdev checks.
- `nbdev-test --path nbs/01_app.ipynb`: run tests for one notebook during focused work.
- `ARCHIVE_DIR=~/reconcile-archive APP_PASSWORD=... SESSION_SECRET=... uvicorn --factory reconcile_web.app:create_app`: run the app locally against a private archive checkout.

Use hyphenated nbdev commands (`nbdev-test`, `nbdev-export`, `nbdev-prepare`), not legacy underscore command names.

## Coding Style & Naming Conventions

Keep code notebook-first. Export production cells with `#| export`; mark exploratory cells with `#| eval: false`. Prefer small pure helpers, concise one-line docstrings, inline nbdev docments for parameters, and explicit `Path` handling for filesystem logic. Use snake_case for functions and variables. Do not add dependencies unless the task requires them and `pyproject.toml` is updated deliberately.

## Testing Guidelines

Tests live as assert-based notebook cells near the code they cover. Add the smallest failing assertion first, then implement the minimal change. For archive/file-security behavior, include traversal, symlink, missing-file, and malformed-month cases where relevant. Run the narrow notebook test first, then `nbdev-prepare` before handing work off.

## Commit & Pull Request Guidelines

Git history uses Conventional Commit prefixes such as `feat:`, `fix:`, and `docs:`. Keep commits focused on one behavior or documentation change. Pull requests should describe the user-facing change, list validation commands run, link the relevant issue when one exists, and include screenshots for visible UI changes.

## Security & Configuration Tips

This app views private financial archive data. Never commit archive contents, passwords, session secrets, account identifiers, or raw receipt data. Keep configuration in environment variables: `ARCHIVE_DIR`, `APP_PASSWORD`, and `SESSION_SECRET`. The app should run behind HTTPS termination and must not be exposed directly.
