# reconcile-web — Design (v1)

**Status:** design, approved for spec review; revised after design review (2026-07-02).

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
- **Serve the Markdown directly.** Render `status.md` → HTML with **Python-Markdown** (`markdown`
  package, `tables` extension), wrap it in the authed layout. Receipt links are rewritten **on the
  raw Markdown text before rendering** (no HTML parsing): `](receipts/<name>.pdf)` →
  `](/m/<month>/receipt/<name>.pdf)`, basename-only and `.pdf`-only. Raw HTML in `status.md` is
  escaped, never passed through to the page, and a post-render pass strips any `href` that does
  not start with `/m/` — Markdown link syntax like `[x](javascript:…)` survives HTML escaping,
  and the allowlist closes that hole. This removes any custom table parser/renderer.
  Minimum code, fully deterministic.

## The `status.md` contract

The interface between the collection skill and this viewer. A valid `status.md`:

```markdown
# 2026-05 — receipt status

| date | title | amount | status | receipt_file | note |
|---|---|---|---|---|---|
| 2026-05-29 | OPENAI *CHATGPT SUBSCR | -110.46 | collected | [2026-05-29_openai.pdf](receipts/2026-05-29_openai.pdf) | src=… |
| 2026-05-06 | NORDEA | -20.46 | not-needed |  |  |

**missing (retrievable from Gmail/portal/Nordea): 0**
```

- Status tokens in the table are `collected` and `not-needed` (**hyphen**, as materialized).
- The `missing` count lives **only in the footer line** — no table row carries it.
- `month_counts` tallies the status-column tokens as-is (keys are the raw tokens) and reads
  `missing` from the footer line. Unknown tokens appear in the tally under their own key —
  never silently dropped. A `status.md` without the footer line is malformed: `month_counts`
  raises `ValueError` (fail closed — never a silent `missing: 0`).
- (Alternative considered: materialize a `counts.json` in the collection skill so the app parses
  nothing. Deferred — the tally is a few lines here, vs. a new cross-repo artifact plus backfill
  of 12 existing months. Revisit if the table format ever changes.)

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
2. **Month index** — list of all month folders, each linking to its month view (show
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
  - `list_months() -> list[str]` — sorted directories in `ARCHIVE_DIR` matching `^\d{4}-\d{2}$`
    **and containing `status.md`** (ignores `README.md`, `vendors.md`, `.git`, strays).
  - `month_counts(month) -> dict` — tally per the `status.md` contract above: raw status tokens
    as keys (`collected`, `not-needed`, …) plus `missing` from the footer line. (This is the only
    bit of `status.md` we parse; the table itself is rendered as Markdown, not parsed.)
  - `status_html(month) -> str` — read `status.md`, rewrite `](receipts/<name>.pdf)` links to
    `](/m/<month>/receipt/<name>.pdf)` on the raw text, then Markdown→HTML with raw HTML escaped.
  - `safe_file(month, kind, name=None) -> Path` — resolve a file to serve; `kind ∈ {statement_pdf,
    statement_csv, receipt}` (`name` used only for receipts and must be a `.pdf` basename);
    **raises `FileNotFoundError`** for
    traversal (`..`, absolute paths, symlinks escaping the month dir) and for files that don't
    exist inside that month's expected location — one error convention, the app maps it to 404.
- **`nbs/01_app.ipynb` → `reconcile_web/app.py`** — the FastHTML app: `Beforeware` auth, routes,
  FT-component layout, file responses. Imports `archive`.

(The scaffold's `00_core.ipynb`/`reconcile_web/core.py` is renamed/repurposed to `00_archive`.)

## Routes

| Route | Purpose |
|---|---|
| `GET /login`, `POST /login`, `GET /logout` | single-password form; set/clear `session['auth']` |
| `GET /` | month index (links + counts) |
| `GET /m/{month}` | month view: `status_html(month)` inside layout + statement.pdf/csv links |
| `GET /m/{month}/statement.pdf` | authed `FileResponse` via `safe_file(month, 'statement_pdf')` |
| `GET /m/{month}/statement.csv` | authed `FileResponse` via `safe_file(month, 'statement_csv')` |
| `GET /m/{month}/receipt/{name}` | authed `FileResponse` via `safe_file(month, 'receipt', name)` |

(No generic `/file/{kind}/{name}` route: the statement filenames are fixed, so a `name` segment
would be dead weight there, and per-kind routes keep the URL ↔ `kind` mapping trivial.)

`{month}` is validated against `list_months()`; unknown → 404.

## Auth

`Beforeware`: redirect to `/login` when `session.get('auth')` is unset, with an explicit skip list
containing **only `/login`**. The app serves no local static assets (Pico CSS comes from the CDN),
and FastHTML's default static route can serve `pdf`/`csv` from the working directory — an exempted
static path would be an auth bypass, so nothing else is skipped. `POST /login` compares the submitted password to env `APP_PASSWORD` with
`secrets.compare_digest` and sets `session['auth'] = True`. Single shared credential — no user
table.

Session cookie: signed with env `SESSION_SECRET` (not FastHTML's auto-generated `.sesskey`, which
would silently rotate on redeploy and log the accountant out); flags `HttpOnly`, `SameSite=Lax`
(Starlette defaults) and `Secure` (`sess_https_only=True` — HTTPS is guaranteed by the proxy).

## Security

- Every route authed except `/login`; PDFs/CSVs served **only** through the authed file routes
  (not `/static`), so downloads can't bypass auth.
- `safe_file` prevents traversal: whitelist `kind`, `name` must match a receipt/statement file that
  actually exists within `ARCHIVE_DIR/<month>/…`; reject `..`, absolute paths, symlinks out.
- Rendered `status.md` can't inject markup: raw HTML is escaped by the renderer, link rewriting
  only touches `](receipts/<basename>.pdf)` patterns, and a post-render allowlist strips any
  `href` not starting with `/m/` — `javascript:` or external hrefs never reach the page.
  (No sanitizer library: content is self-generated; escape + allowlist is the proportionate
  defense.)
- `APP_PASSWORD` and `SESSION_SECRET` from env, never committed. Repo private. HTTPS enforced at
  the proxy.
- No rate-limiting / lockout in v1 (YAGNI; a single long password + HTTPS is the accepted bar).

## Data flow

`request → Beforeware(auth) → route → archive.py reads ARCHIVE_DIR files → FT render / FileResponse`.
No database, no writes, no external calls.

## Error handling

- Missing env (`APP_PASSWORD`, `SESSION_SECRET`, `ARCHIVE_DIR`) or unreadable `ARCHIVE_DIR` →
  **fail fast at startup** with a clear message, not a broken app.
- Unreadable or malformed month at request time (e.g. footer line missing → `ValueError`) →
  clear error page (500 with a short message), logged.
- Unknown month or file (`safe_file` raises `FileNotFoundError`) → 404.
- Wrong password → re-render `/login` with an error notice (no detail leakage).

## Testing (nbdev inline asserts)

- `00_archive`: `list_months` returns sorted months for a fixture archive **and skips non-month
  entries** (`README.md`, a stray dir); `month_counts` matches a known `status.md` (hyphenated
  `not-needed`, footer-line `missing`, an unknown token surfacing under its own key) and
  **raises `ValueError` when the footer line is removed**; `status_html` rewrites
  `receipts/x.pdf` → the authed route, leaves other Markdown intact, **escapes a `<script>`
  planted in the fixture**, and **strips the href of a planted `[x](javascript:alert(1))`
  link**; `safe_file` returns the right path for valid input and **raises `FileNotFoundError`**
  for `..`, unknown files, a **non-`.pdf` name** (`secret.txt`), and a **symlink pointing
  outside the month dir** (e.g. `receipts/evil.pdf → /etc/hostname`).
- `01_app`: smoke via FastHTML's test client — `/` and `/m/{month}` redirect to `/login` when
  unauthenticated; after login they return 200; the receipt route serves an existing receipt and
  404s a bogus name.
- Fixtures: a tiny throwaway archive dir (2 months, a couple of receipts) created in the test cell.

## Deployment (out of scope here)

Owned by the user's automated CI/CD (VM spawn + secure config + deploy, learned from the Solveit2
course). This app only commits to being automation-friendly: config via env (`APP_PASSWORD`,
`SESSION_SECRET`, `ARCHIVE_DIR`), pinned deps, and a documented run command
(`uvicorn reconcile_web.app:app` or a `serve()` entrypoint). No manual deploy steps are designed
or maintained here.

Runtime deps (to be declared in `pyproject.toml` `dependencies` — currently empty — at
implementation time): `python-fasthtml` (brings uvicorn/starlette) and `markdown`.

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
