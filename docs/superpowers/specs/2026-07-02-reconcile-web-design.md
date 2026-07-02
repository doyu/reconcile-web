# reconcile-web — Design (v1)

**Status:** design, approved for spec review (2026-07-02).

## Goal

A small **read-only web app** that serves the accountant everything she needs to check the books:
for each month, the **bank statement** and the **receipts**, with **each transaction linked to its
receipt** (a downloadable link). Deployed on a cheap VM, behind a single password. Also an
**educational project** for learning FastHTML + nbdev.

The data already exists and is presentation-ready in the private repo **`reconcile-archive`**
(one folder per month: `statement.csv`, `statement.pdf`, `receipts/*.pdf`, `status.md`). This app is
a **thin viewer** over that data — it never mutates the archive.

## Core principle: reuse what's already materialized

The hard, fuzzy work (finding receipts, matching them to transactions, FX judgement) was done by an
LLM/skill offline and **materialized into `status.md`** — a Markdown table with one row per bank
transaction, `status`, and a **relative link to each receipt**. So:

- **No LLM at request time.** The viewer's work is deterministic, hot-path, and reliability-critical
  (financial data + auth). An LLM in the request path would be slower, costlier, non-deterministic —
  strictly worse. LLM stays where it belongs: build-time / the collection skill.
- **Serve the Markdown directly.** Render `status.md` → HTML (a Markdown library), wrap it in the
  authed layout, and rewrite the relative receipt links to authed download routes. This removes any
  custom table parser/renderer. Minimum code, fully deterministic.

## Global constraints

- **nbdev** development style (source in `nbs/*.ipynb`, exported to `reconcile_web/`), TDD via
  inline `assert` test cells (`nbdev-test`), per the `nbdev-tdd` skill.
- **YAGNI, ruthlessly.** Only what's listed under Scope. Everything under Non-goals is excluded.
- Real financial data → the app repo and `reconcile-archive` are **private**; all app routes require
  auth; served over **HTTPS**.
- **Deployment/CI/CD is out of scope for this app** (owned by the user, learned separately). The app
  only guarantees it is easy to automate: env-based config, a clear run entrypoint, pinned deps.

## Scope (v1 — the whole thing)

1. **Login** — one shared password (from env), session cookie.
2. **Month index** — list of the 12 month folders, each linking to its month view (show
   collected / not-needed / missing counts).
3. **Month view** — render `status.md` (Markdown → HTML) as the transaction table; each row's
   receipt link points at an authed download route. Top of page: download links for
   `statement.pdf` and `statement.csv`.
4. **Authed file download** — serve `statement.pdf`, `statement.csv`, and `receipts/*.pdf` for a
   month through an authenticated route (never from `/static`).

## Architecture

Single FastHTML app, one uvicorn process. Reads the `reconcile-archive` checkout as plain files
(path from env `ARCHIVE_DIR`). All routes behind a `Beforeware` auth gate. On the VM, HTTPS is
terminated by a reverse proxy (user's CI/CD concern); the archive is refreshed with
`git pull` (read-only deploy key). The app is stateless apart from the session cookie.

```
browser ──HTTPS──> [reverse proxy, user-owned] ──> uvicorn(FastHTML app)
                                                        │ reads files
                                                        ▼
                                              ARCHIVE_DIR/<month>/{statement.csv,statement.pdf,receipts/,status.md}
```

## Components (nbdev notebooks → module)

Keep IO/parsing separate from routing so the data layer can later be lifted into a skill/MCP.

- **`nbs/00_archive.ipynb` → `reconcile_web/archive.py`** — data access, UI-free, pure/testable:
  - `list_months() -> list[str]` — sorted `["2025-07", …, "2026-06"]` from `ARCHIVE_DIR`.
  - `month_counts(month) -> dict` — `{collected, not_needed, missing}` (cheap scan of `status.md`),
    for the index. (This is the only bit of `status.md` we parse in code; the table itself is
    rendered as Markdown, not parsed.)
  - `status_html(month) -> str` — read `status.md`, Markdown→HTML, **rewrite `receipts/<f>` links**
    to `/m/<month>/file/receipt/<f>`.
  - `safe_file(month, kind, name) -> Path` — resolve a file to serve; `kind ∈ {statement_pdf,
    statement_csv, receipt}`; reject path traversal; only return files that exist inside that
    month's expected location.
- **`nbs/01_app.ipynb` → `reconcile_web/app.py`** — the FastHTML app: `Beforeware` auth, routes,
  FT-component layout, file responses. Imports `archive`.

(The scaffold's `00_core.ipynb`/`reconcile_web/core.py` is renamed/repurposed to `00_archive`.)

## Routes

| Route | Purpose |
|---|---|
| `GET /login`, `POST /login`, `GET /logout` | single-password form; set/clear `session['auth']` |
| `GET /` | month index (links + counts) |
| `GET /m/{month}` | month view: `status_html(month)` inside layout + statement.pdf/csv links |
| `GET /m/{month}/file/{kind}/{name}` | authed `FileResponse` via `safe_file` (statement.pdf / statement.csv / receipt) |

`{month}` is validated against `list_months()`; unknown → 404.

## Auth

`Beforeware`: if `session.get('auth')` is unset and the path isn't `/login` (or the app's own CSS),
redirect to `/login`. `POST /login` compares the submitted password to env `APP_PASSWORD`
(constant-time compare) and sets `session['auth'] = True`. Single shared credential — no user table.

## Security

- Every route authed except `/login`; PDFs/CSVs served **only** through the authed `file` route
  (not `/static`), so downloads can't bypass auth.
- `safe_file` prevents traversal: whitelist `kind`, `name` must match a receipt/statement file that
  actually exists within `ARCHIVE_DIR/<month>/…`; reject `..`, absolute paths, symlinks out.
- `APP_PASSWORD` from env, never committed. Repo private. HTTPS enforced at the proxy.
- No rate-limiting / lockout in v1 (YAGNI; a single long password + HTTPS is the accepted bar).

## Data flow

`request → Beforeware(auth) → route → archive.py reads ARCHIVE_DIR files → FT render / FileResponse`.
No database, no writes, no external calls.

## Error handling

- Missing `ARCHIVE_DIR` or unreadable month → clear error page (500 with a short message), logged.
- Unknown month or file → 404.
- Wrong password → re-render `/login` with an error notice (no detail leakage).

## Testing (nbdev inline asserts)

- `00_archive`: `list_months` returns sorted months for a fixture archive; `month_counts` matches a
  known `status.md`; `status_html` rewrites `receipts/x.pdf` → the authed route and leaves other
  Markdown intact; `safe_file` returns the right path for valid input and **raises/None for `..` and
  unknown files**.
- `01_app`: smoke via FastHTML's test client — `/` and `/m/{month}` redirect to `/login` when
  unauthenticated; after login they return 200; the `file` route serves an existing receipt and 404s
  a bogus name.
- Fixtures: a tiny throwaway archive dir (2 months, a couple of receipts) created in the test cell.

## Deployment (out of scope here)

Owned by the user's automated CI/CD (VM spawn + secure config + deploy, learned from the Solveit2
course). This app only commits to being automation-friendly: config via env (`APP_PASSWORD`,
`ARCHIVE_DIR`), pinned deps, and a documented run command (`uvicorn reconcile_web.app:app` or a
`serve()` entrypoint). No manual deploy steps are designed or maintained here.

## Future (noted, not built — YAGNI)

- **skill-first ad-hoc queries:** natural-language questions over the data ("USD charges > $100 in
  Q4") answered by an LLM instead of hand-coded filters — the right place for a request-time LLM,
  but a later increment.
- **Extract the archive logic** (`archive.py`, and the collection/matching pipeline) into a reusable
  **skill, then MCP or code** as reliability/speed demands. Kept extractable by keeping `archive.py`
  UI-free and pure.

## Non-goals (v1 excludes)

Search, filtering, sorting, inline PDF preview, editing/upload, multiple users/roles, per-month
summaries/analytics, a database, any write path, and any deploy tooling in this repo.
