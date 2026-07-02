# reconcile-web v1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Read-only FastHTML viewer over the `reconcile-archive` checkout: login, month index with counts, month view rendering `status.md`, authed file downloads.

**Architecture:** Two nbdev notebooks. `nbs/00_archive.ipynb` → `reconcile_web/archive.py` (pure, UI-free data access). `nbs/01_app.ipynb` → `reconcile_web/app.py` (FastHTML app factory `create_app()` with Beforeware auth; routes call `archive` functions). No database, no writes, no LLM at request time.

**Tech Stack:** nbdev (notebook-first, inline `assert` tests), FastHTML (`python-fasthtml`), Python-Markdown (`markdown` + `tables` extension), uvicorn (comes with fasthtml), starlette `TestClient`.

**Spec:** `docs/superpowers/specs/2026-07-02-reconcile-web-design.md` — the current revision in the repo. Where this plan and the spec disagree, the spec wins.

## Global Constraints

- Source of truth is `nbs/*.ipynb`. **Never edit `reconcile_web/*.py` by hand** — they are nbdev outputs.
- Always work inside the repo venv: `source /home/doyu/reconcile-web/.venv/bin/activate` (Python 3.12).
- nbdev CLI commands are **hyphenated**: `nbdev-export`, `nbdev-test`, `nbdev-prepare`. Never the underscore forms.
- Only two new runtime deps, exact-pinned in `pyproject.toml`: `python-fasthtml`, `markdown`. No other new packages.
- Status tokens in `status.md` are `collected` and `not-needed` (**hyphen**). The `missing` count comes **only** from the footer line `**missing (…): N**`.
- Env config: `APP_PASSWORD`, `SESSION_SECRET`, `ARCHIVE_DIR`. Missing env → `RuntimeError` at startup.
- nbdev docments style: types in signatures, one-line docstrings, inline `#` parameter comments.
- Commits are local; **do not push or open a PR without the user's explicit approval**. No direct push to `main`.
- Commit policy (user-approved 2026-07-02): approving an execution mode for this plan **is** the standing approval for the per-task local commits below — no per-commit diff confirmation needed. Push/PR stay gated at Tasks 5 and 9.
- Before any commit that touches notebooks, normalize them:
  `find nbs -name '*.ipynb' -print0 | xargs -0 python /home/doyu/.claude/skills/nbdev-tdd/scripts/normalize_notebooks.py`

## File Structure

- `nbs/00_archive.ipynb` (create; replaces scaffold `nbs/00_core.ipynb`) → exports `reconcile_web/archive.py`: `list_months`, `month_counts`, `status_html`, `safe_file`. Tests live as plain (non-export) cells in the same notebook.
- `nbs/01_app.ipynb` (create) → exports `reconcile_web/app.py`: `create_app`, `serve`. Tests via `starlette.testclient.TestClient` in plain cells.
- `nbs/index.ipynb` (modify): fix the `reconcile_web.core` import; document env vars + run command (becomes README).
- `pyproject.toml` (modify): `dependencies` (currently `[]` at line 18) gets the two pins.
- Delete: `nbs/00_core.ipynb`, `reconcile_web/core.py`.

Both notebooks duplicate a ~20-line test fixture (`make_archive`). Deliberate: test fixtures aren't worth a shared module (YAGNI), and each notebook must be runnable standalone by `nbdev-test`.

## Test Fixture (used by Tasks 2–8)

This exact cell is added to **both** notebooks (plain code cell, no `#| export`):

```python
# test fixture: a throwaway archive matching the status.md contract in the spec
import tempfile
from pathlib import Path

STATUS_MD = """# {m} — receipt status

| date | title | amount | status | receipt_file | note |
|---|---|---|---|---|---|
| {m}-29 | OPENAI *CHATGPT SUBSCR | -110.46 | collected | [{m}-29_openai.pdf](receipts/{m}-29_openai.pdf) | src=x.pdf |
| {m}-06 | NORDEA | -20.46 | not-needed |  |  |

**missing (retrievable from Gmail/portal/Nordea): 1**
"""

def make_archive(months=('2025-07', '2025-08')):
    root = Path(tempfile.mkdtemp())
    (root/'README.md').write_text('readme')
    (root/'notamonth').mkdir()
    for m in months:
        d = root/m
        (d/'receipts').mkdir(parents=True)
        (d/'statement.pdf').write_bytes(b'%PDF-1.4 fake')
        (d/'statement.csv').write_text('date;amount\n')
        (d/'status.md').write_text(STATUS_MD.format(m=m))
        (d/'receipts'/f'{m}-29_openai.pdf').write_bytes(b'%PDF-1.4 fake receipt')
    return root
```

Expected counts for any fixture month: `{'collected': 1, 'not-needed': 1, 'missing': 1}`.

---

### Task 1: Deps + notebook scaffold swap

**Files:**
- Modify: `pyproject.toml` (line 18, `dependencies = []`)
- Modify: `requirements.txt` (regenerate freeze)
- Create: `nbs/00_archive.ipynb`
- Modify: `nbs/index.ipynb` (first cell)
- Delete: `nbs/00_core.ipynb`, `reconcile_web/core.py`

**Interfaces:**
- Consumes: nothing.
- Produces: importable empty `reconcile_web.archive` module; installed `fasthtml` + `markdown`; branch `archive-layer`.

- [ ] **Step 1: Branch**

```bash
cd /home/doyu/reconcile-web && git checkout -b archive-layer
```

- [ ] **Step 2: Install and pin deps**

```bash
source .venv/bin/activate
pip install python-fasthtml markdown
pip freeze | grep -iE '^(python-fasthtml|markdown)=='
```

Take the two lines printed by the grep (e.g. `python-fasthtml==0.12.x`, `Markdown==3.x`) and put them, verbatim but lowercased in the name, into `pyproject.toml`:

```toml
dependencies = ["python-fasthtml==<version from grep>", "markdown==<version from grep>"]
```

Then refresh the dev freeze: `pip freeze > requirements.txt`

- [ ] **Step 3: Create `nbs/00_archive.ipynb`**

Cells, in order:

1. markdown: `# archive` / `> Read-only access to the reconcile-archive checkout`
2. code: `#| default_exp archive`
3. code: `#| hide` / `from nbdev.showdoc import *`
4. code:
   ```python
   #| export
   import re, html
   from pathlib import Path
   import markdown as md
   ```
5. the **Test Fixture** cell from the top of this plan
6. code: `#| hide` / `import nbdev; nbdev.nbdev_export()`

- [ ] **Step 4: Remove the scaffold module**

```bash
rm nbs/00_core.ipynb reconcile_web/core.py
```

Edit `nbs/index.ipynb` first cell: `from reconcile_web.core import *` → `from reconcile_web.archive import *`.

- [ ] **Step 5: Export and verify**

```bash
nbdev-export && python -c "import reconcile_web.archive, fasthtml, markdown; print('ok')"
nbdev-test --path nbs/00_archive.ipynb
```

Expected: `ok`, tests pass (only the fixture cell runs). `reconcile_web/archive.py` exists; `core` gone from `reconcile_web/_modidx.py`.

- [ ] **Step 6: Commit**

```bash
find nbs -name '*.ipynb' -print0 | xargs -0 python /home/doyu/.claude/skills/nbdev-tdd/scripts/normalize_notebooks.py
git add -A && git commit -m "feat: swap scaffold core -> archive module, add fasthtml+markdown deps"
```

---

### Task 2: `list_months`

**Files:**
- Modify: `nbs/00_archive.ipynb` (new cells before the closing `#| hide` export cell — same for all archive tasks)

**Interfaces:**
- Consumes: `make_archive` fixture.
- Produces: `list_months(archive_dir: str|Path) -> list[str]` — sorted `YYYY-MM` dir names containing `status.md`.

- [ ] **Step 1: Stub + failing test**

Add an export cell:

```python
#| export
MONTH_RE = re.compile(r'^\d{4}-\d{2}$')

def list_months(
    archive_dir: str|Path,  # path to the reconcile-archive checkout
) -> list[str]:             # sorted month names like '2025-07'
    "Month directories (YYYY-MM containing a status.md) in sorted order"
    raise NotImplementedError
```

And a plain test cell below it:

```python
root = make_archive()
assert list_months(root) == ['2025-07', '2025-08']
(root/'2030-01').mkdir()  # month-shaped dir without status.md -> excluded
assert list_months(root) == ['2025-07', '2025-08']
```

- [ ] **Step 2: Verify it fails**

Run: `nbdev-test --path nbs/00_archive.ipynb`
Expected: FAIL with `NotImplementedError`.

- [ ] **Step 3: Implement**

Replace the stub body:

```python
#| export
MONTH_RE = re.compile(r'^\d{4}-\d{2}$')

def list_months(
    archive_dir: str|Path,  # path to the reconcile-archive checkout
) -> list[str]:             # sorted month names like '2025-07'
    "Month directories (YYYY-MM containing a status.md) in sorted order"
    return sorted(p.name for p in Path(archive_dir).iterdir()
                  if p.is_dir() and MONTH_RE.match(p.name) and (p/'status.md').exists())
```

- [ ] **Step 4: Verify it passes**

Run: `nbdev-test --path nbs/00_archive.ipynb`
Expected: PASS.

- [ ] **Step 5: Export, normalize, commit**

```bash
nbdev-export
find nbs -name '*.ipynb' -print0 | xargs -0 python /home/doyu/.claude/skills/nbdev-tdd/scripts/normalize_notebooks.py
git add nbs/00_archive.ipynb reconcile_web/archive.py reconcile_web/_modidx.py
git commit -m "feat: list_months"
```

---

### Task 3: `month_counts`

**Files:**
- Modify: `nbs/00_archive.ipynb`

**Interfaces:**
- Consumes: `make_archive`.
- Produces: `month_counts(archive_dir, month) -> dict` — raw status tokens as keys plus `'missing'` from the footer line; raises `ValueError` if the footer line is absent (fail closed). Task 6 renders `c.get('collected', 0)` / `c.get('not-needed', 0)` / `c.get('missing', 0)`.

- [ ] **Step 1: Stub + failing test**

Export cell:

```python
#| export
MISSING_RE = re.compile(r'\*\*missing[^:]*:\s*(\d+)\*\*')

def month_counts(
    archive_dir: str|Path,  # archive root
    month: str,             # month name like '2025-07'
) -> dict:                  # raw status tokens -> count, plus 'missing' from the footer line
    "Tally the status column of status.md; read missing from the footer"
    raise NotImplementedError
```

Test cell:

```python
root = make_archive()
assert month_counts(root, '2025-07') == {'collected': 1, 'not-needed': 1, 'missing': 1}
# unknown tokens surface under their own key, never silently dropped
sm = root/'2025-08'/'status.md'
sm.write_text(sm.read_text().replace('not-needed', 'wtf-token'))
c = month_counts(root, '2025-08')
assert c['wtf-token'] == 1 and 'not-needed' not in c
# malformed status.md (no footer line) fails closed, never a silent missing=0
sm.write_text('\n'.join(l for l in sm.read_text().splitlines() if not l.startswith('**missing')))
try: month_counts(root, '2025-08'); assert False, 'should have raised'
except ValueError: pass
```

- [ ] **Step 2: Verify it fails**

Run: `nbdev-test --path nbs/00_archive.ipynb`
Expected: FAIL with `NotImplementedError`.

- [ ] **Step 3: Implement**

```python
#| export
MISSING_RE = re.compile(r'\*\*missing[^:]*:\s*(\d+)\*\*')

def month_counts(
    archive_dir: str|Path,  # archive root
    month: str,             # month name like '2025-07'
) -> dict:                  # raw status tokens -> count, plus 'missing' from the footer line
    "Tally the status column of status.md; read missing from the footer"
    text = (Path(archive_dir)/month/'status.md').read_text()
    counts = {}
    for line in text.splitlines():
        line = line.strip()
        if not line.startswith('|'): continue
        cells = [c.strip() for c in line.strip('|').split('|')]
        if len(cells) < 4: continue
        tok = cells[3]
        if not tok or tok.lower() == 'status' or set(tok) <= {'-'}: continue  # header/separator rows
        counts[tok] = counts.get(tok, 0) + 1
    m = MISSING_RE.search(text)
    if m is None: raise ValueError(f"{month}/status.md: missing-count footer line not found")
    counts['missing'] = int(m.group(1))
    return counts
```

- [ ] **Step 4: Verify it passes**

Run: `nbdev-test --path nbs/00_archive.ipynb`
Expected: PASS.

- [ ] **Step 5: Export, normalize, commit**

```bash
nbdev-export
find nbs -name '*.ipynb' -print0 | xargs -0 python /home/doyu/.claude/skills/nbdev-tdd/scripts/normalize_notebooks.py
git add nbs/00_archive.ipynb reconcile_web/archive.py reconcile_web/_modidx.py
git commit -m "feat: month_counts"
```

---

### Task 4: `status_html`

**Files:**
- Modify: `nbs/00_archive.ipynb`

**Interfaces:**
- Consumes: `make_archive`.
- Produces: `status_html(archive_dir, month) -> str` — HTML fragment; receipt links rewritten to `/m/<month>/receipt/<name>.pdf`; raw HTML escaped; any href not starting with `/m/` stripped.

- [ ] **Step 1: Stub + failing test**

Export cell:

```python
#| export
RECEIPT_LINK_RE = re.compile(r'\]\(receipts/([A-Za-z0-9._-]+\.pdf)\)')
HREF_RE = re.compile(r'href="(?!/m/)[^"]*"')  # any rendered href not pointing at an authed route

def status_html(
    archive_dir: str|Path,  # archive root
    month: str,             # month name like '2025-07'
) -> str:                   # HTML fragment of the rendered status.md
    "Render status.md to HTML: rewrite receipt links, escape raw HTML, allowlist hrefs"
    raise NotImplementedError
```

Test cell:

```python
root = make_archive()
h = status_html(root, '2025-07')
assert '/m/2025-07/receipt/2025-07-29_openai.pdf' in h
assert '](receipts/' not in h and 'href="receipts/' not in h
assert '<table>' in h                       # tables extension active
assert 'not-needed' in h                    # rest of the Markdown intact
# raw HTML must be escaped, not passed through
sm = root/'2025-07'/'status.md'
sm.write_text(sm.read_text() + '\n<script>alert(1)</script>\n')
h = status_html(root, '2025-07')
assert '<script>' not in h and '&lt;script&gt;' in h
# Markdown link syntax survives HTML escaping — the href allowlist must strip it
sm.write_text(sm.read_text() + '\n[evil](javascript:alert(1))\n')
h = status_html(root, '2025-07')
assert 'javascript:' not in h
assert 'href="/m/2025-07/receipt/2025-07-29_openai.pdf"' in h  # receipt hrefs survive the allowlist
```

- [ ] **Step 2: Verify it fails**

Run: `nbdev-test --path nbs/00_archive.ipynb`
Expected: FAIL with `NotImplementedError`.

- [ ] **Step 3: Implement**

```python
#| export
RECEIPT_LINK_RE = re.compile(r'\]\(receipts/([A-Za-z0-9._-]+\.pdf)\)')
HREF_RE = re.compile(r'href="(?!/m/)[^"]*"')  # any rendered href not pointing at an authed route

def status_html(
    archive_dir: str|Path,  # archive root
    month: str,             # month name like '2025-07'
) -> str:                   # HTML fragment of the rendered status.md
    "Render status.md to HTML: rewrite receipt links, escape raw HTML, allowlist hrefs"
    text = (Path(archive_dir)/month/'status.md').read_text()
    text = RECEIPT_LINK_RE.sub(rf'](/m/{month}/receipt/\1)', text)
    out = md.markdown(html.escape(text, quote=False), extensions=['tables'])
    return HREF_RE.sub('', out)  # Markdown [x](javascript:…) survives escaping; only /m/… hrefs pass
```

- [ ] **Step 4: Verify it passes**

Run: `nbdev-test --path nbs/00_archive.ipynb`
Expected: PASS.

- [ ] **Step 5: Export, normalize, commit**

```bash
nbdev-export
find nbs -name '*.ipynb' -print0 | xargs -0 python /home/doyu/.claude/skills/nbdev-tdd/scripts/normalize_notebooks.py
git add nbs/00_archive.ipynb reconcile_web/archive.py reconcile_web/_modidx.py
git commit -m "feat: status_html"
```

---

### Task 5: `safe_file` (+ PR #1)

**Files:**
- Modify: `nbs/00_archive.ipynb`

**Interfaces:**
- Consumes: `make_archive`, `MONTH_RE` (Task 2).
- Produces: `safe_file(archive_dir, month, kind, name=None) -> Path`; `kind ∈ {'statement_pdf','statement_csv','receipt'}`; receipt `name` must be a `.pdf` basename; raises `FileNotFoundError` for traversal/non-pdf/missing — the app maps it to 404.

- [ ] **Step 1: Stub + failing test**

Export cell:

```python
#| export
def safe_file(
    archive_dir: str|Path,  # archive root
    month: str,             # month name like '2025-07'
    kind: str,              # 'statement_pdf' | 'statement_csv' | 'receipt'
    name: str|None = None,  # receipt filename (receipt kind only)
) -> Path:                  # resolved file inside the month dir
    "Resolve a servable file; raise FileNotFoundError on traversal or anything missing"
    raise NotImplementedError
```

Test cell:

```python
root = make_archive()

def rejects(*a):
    try: safe_file(*a)
    except FileNotFoundError: return True
    return False

assert safe_file(root, '2025-07', 'statement_pdf').name == 'statement.pdf'
assert safe_file(root, '2025-07', 'statement_csv').is_file()
assert safe_file(root, '2025-07', 'receipt', '2025-07-29_openai.pdf').is_file()
assert rejects(root, '2025-07', 'receipt', '../statement.pdf')      # traversal
assert rejects(root, '2025-07', 'receipt', '/etc/hostname')         # absolute
assert rejects(root, '2025-07', 'receipt', 'nope.pdf')              # missing
(root/'2025-07'/'receipts'/'secret.txt').write_text('x')
assert rejects(root, '2025-07', 'receipt', 'secret.txt')            # exists but not a .pdf
assert rejects(root, '2025-07', 'receipt', None)                    # no name
assert rejects(root, '..', 'statement_pdf')                         # bad month
assert rejects(root, '2025-07', 'bogus_kind')                       # bad kind
(root/'2025-07'/'receipts'/'evil.pdf').symlink_to('/etc/hostname')  # symlink escape
assert rejects(root, '2025-07', 'receipt', 'evil.pdf')
```

- [ ] **Step 2: Verify it fails**

Run: `nbdev-test --path nbs/00_archive.ipynb`
Expected: FAIL with `NotImplementedError`.

- [ ] **Step 3: Implement**

```python
#| export
def safe_file(
    archive_dir: str|Path,  # archive root
    month: str,             # month name like '2025-07'
    kind: str,              # 'statement_pdf' | 'statement_csv' | 'receipt'
    name: str|None = None,  # receipt filename (receipt kind only)
) -> Path:                  # resolved file inside the month dir
    "Resolve a servable file; raise FileNotFoundError on traversal or anything missing"
    if not MONTH_RE.match(month): raise FileNotFoundError(month)
    mdir = (Path(archive_dir)/month).resolve()
    if kind == 'statement_pdf':   p = mdir/'statement.pdf'
    elif kind == 'statement_csv': p = mdir/'statement.csv'
    elif kind == 'receipt':
        if not name or name != Path(name).name or not name.endswith('.pdf'):
            raise FileNotFoundError(name)
        p = mdir/'receipts'/name
    else: raise FileNotFoundError(kind)
    rp = p.resolve()  # follows symlinks; an escaping symlink lands outside mdir
    if not (rp.is_file() and rp.is_relative_to(mdir)): raise FileNotFoundError(p)
    return rp
```

- [ ] **Step 4: Verify it passes**

Run: `nbdev-test --path nbs/00_archive.ipynb`
Expected: PASS.

- [ ] **Step 5: Full check, commit**

```bash
find nbs -name '*.ipynb' -print0 | xargs -0 python /home/doyu/.claude/skills/nbdev-tdd/scripts/normalize_notebooks.py
nbdev-prepare
git add -A && git commit -m "feat: safe_file"
```

Expected: `nbdev-prepare` green (export + test + clean, all notebooks).

- [ ] **Step 6: PR #1 — STOP, ask the user**

Show `git log --oneline main..` and the diffstat. **Ask the user to approve pushing branch `archive-layer` and opening the PR** ("feat: archive data layer"). Do not push without approval.

---

### Task 6: App factory, auth, month index

**Files:**
- Create: `nbs/01_app.ipynb`
- (Branch: after PR #1 is approved/merged per the user's instruction, continue on `archive-layer` or a new `app-layer` branch from it — ask the user at the Task 5 checkpoint.)

**Interfaces:**
- Consumes: `list_months`, `month_counts` from `reconcile_web.archive` (Tasks 2–3).
- Produces: `create_app(archive_dir=None, password=None, session_secret=None)` → FastHTML app. Missing config → `RuntimeError`. Beforeware skip list is `['/login']` only. Routes so far: `GET/POST /login`, `GET /logout`, `GET /`.

- [ ] **Step 1: Create `nbs/01_app.ipynb` with stub + failing tests**

Cells, in order:

1. markdown: `# app` / `> FastHTML viewer: auth gate + month index + month view + authed downloads`
2. code: `#| default_exp app`
3. code: `#| hide` / `from nbdev.showdoc import *`
4. code:
   ```python
   #| export
   import os, secrets
   from pathlib import Path
   from fasthtml.common import fast_app, Beforeware, Titled, Form, Input, Button, P, Ul, Li, A, NotStr
   from starlette.responses import RedirectResponse, FileResponse
   from starlette.exceptions import HTTPException
   from reconcile_web.archive import list_months, month_counts, status_html, safe_file
   ```
5. code (stub):
   ```python
   #| export
   def create_app(
       archive_dir: str|Path|None = None,  # archive root (default: env ARCHIVE_DIR)
       password: str|None = None,          # shared password (default: env APP_PASSWORD)
       session_secret: str|None = None,    # session signing key (default: env SESSION_SECRET)
   ):
       "Build the FastHTML viewer app; raise RuntimeError on missing/invalid config"
       raise NotImplementedError
   ```
6. the **Test Fixture** cell from the top of this plan (verbatim duplicate)
7. test cell:
   ```python
   from starlette.testclient import TestClient

   root = make_archive()
   app = create_app(archive_dir=root, password='pw', session_secret='s3cret')
   client = TestClient(app, base_url='https://testserver')  # https: Secure cookie must round-trip

   # unauthenticated -> redirected to /login
   r = client.get('/', follow_redirects=False)
   assert r.status_code == 303 and r.headers['location'] == '/login'
   # static-looking paths are NOT auth-exempt (FastHTML's static route can serve pdf/csv)
   assert client.get('/static/x.pdf', follow_redirects=False).status_code == 303
   # login page reachable without auth
   assert client.get('/login').status_code == 200
   # wrong password -> error notice, still no access
   r = client.post('/login', data={'password': 'wrong'})
   assert 'Wrong password' in r.text
   assert client.get('/', follow_redirects=False).status_code == 303
   # correct password -> index with months and counts
   r = client.post('/login', data={'password': 'pw'}, follow_redirects=False)
   assert r.status_code == 303
   r = client.get('/')
   assert r.status_code == 200 and '2025-07' in r.text and '2025-08' in r.text
   assert 'collected 1' in r.text and 'not-needed 1' in r.text and 'missing 1' in r.text
   # logout ends the session
   client.get('/logout', follow_redirects=False)
   assert client.get('/', follow_redirects=False).status_code == 303
   # missing config fails fast
   try:
       create_app(archive_dir=root, password='pw'); assert False, 'should have raised'
   except RuntimeError as e:
       assert 'SESSION_SECRET' in str(e)
   ```
8. code: `#| hide` / `import nbdev; nbdev.nbdev_export()`

- [ ] **Step 2: Verify it fails**

```bash
nbdev-export && nbdev-test --path nbs/01_app.ipynb
```
Expected: FAIL with `NotImplementedError`. (Export first: the notebook imports the generated `reconcile_web.archive`.)

- [ ] **Step 3: Implement `create_app`**

Replace the stub cell:

```python
#| export
def create_app(
    archive_dir: str|Path|None = None,  # archive root (default: env ARCHIVE_DIR)
    password: str|None = None,          # shared password (default: env APP_PASSWORD)
    session_secret: str|None = None,    # session signing key (default: env SESSION_SECRET)
):
    "Build the FastHTML viewer app; raise RuntimeError on missing/invalid config"
    archive_dir = archive_dir or os.environ.get('ARCHIVE_DIR')
    app_password = password or os.environ.get('APP_PASSWORD')
    session_secret = session_secret or os.environ.get('SESSION_SECRET')
    missing = [n for n, v in (('ARCHIVE_DIR', archive_dir), ('APP_PASSWORD', app_password),
                              ('SESSION_SECRET', session_secret)) if not v]
    if missing: raise RuntimeError(f"missing config: {', '.join(missing)}")
    if not Path(archive_dir).is_dir(): raise RuntimeError(f"ARCHIVE_DIR is not a directory: {archive_dir}")

    def auth_before(req, session):
        if not session.get('auth'): return RedirectResponse('/login', status_code=303)

    # skip only /login: the app serves no local static files (Pico CSS comes from the CDN), and
    # FastHTML's default static route can serve pdf/csv from cwd — an exempted path would bypass auth
    app, rt = fast_app(before=Beforeware(auth_before, skip=[r'/login']),
                       secret_key=session_secret, sess_https_only=True)

    def login_page(error=False):
        body = [Form(Input(name='password', type='password', required=True),
                     Button('Login'), method='post', action='/login')]
        if error: body.insert(0, P('Wrong password'))
        return Titled('Login', *body)

    @rt('/login', methods=['GET'])
    def login_form(): return login_page()

    @rt('/login', methods=['POST'])
    def login_submit(password: str, session):
        if password and secrets.compare_digest(password, app_password):
            session['auth'] = True
            return RedirectResponse('/', status_code=303)
        return login_page(error=True)

    @rt('/logout', methods=['GET'])
    def logout(session):
        session.pop('auth', None)
        return RedirectResponse('/login', status_code=303)

    @rt('/', methods=['GET'])
    def index():
        items = [Li(A(m, href=f'/m/{m}'),
                    f" — collected {c.get('collected', 0)}, not-needed {c.get('not-needed', 0)},"
                    f" missing {c.get('missing', 0)}")
                 for m in list_months(archive_dir) for c in [month_counts(archive_dir, m)]]
        return Titled('reconcile-archive', Ul(*items), P(A('Logout', href='/logout')))

    return app
```

- [ ] **Step 4: Verify it passes**

Run: `nbdev-test --path nbs/01_app.ipynb`
Expected: PASS. If the Secure-cookie round-trip fails (session never sticks), confirm the `TestClient` `base_url` is `https://testserver`.

- [ ] **Step 5: Export, normalize, commit**

```bash
nbdev-export
find nbs -name '*.ipynb' -print0 | xargs -0 python /home/doyu/.claude/skills/nbdev-tdd/scripts/normalize_notebooks.py
git add nbs/01_app.ipynb reconcile_web/app.py reconcile_web/_modidx.py
git commit -m "feat: app factory with auth gate and month index"
```

---

### Task 7: Month view route

**Files:**
- Modify: `nbs/01_app.ipynb` (the `create_app` cell + a new test cell before the closing export cell)

**Interfaces:**
- Consumes: `status_html` (Task 4), `create_app` internals (Task 6).
- Produces: `GET /m/{month}` — 200 with rendered table for known months, 404 for unknown.

- [ ] **Step 1: Failing test**

New plain test cell:

```python
root = make_archive()
app = create_app(archive_dir=root, password='pw', session_secret='s3cret')
client = TestClient(app, base_url='https://testserver')
client.post('/login', data={'password': 'pw'})

r = client.get('/m/2025-07')
assert r.status_code == 200
assert '/m/2025-07/receipt/2025-07-29_openai.pdf' in r.text   # rewritten receipt link
assert '/m/2025-07/statement.pdf' in r.text and '/m/2025-07/statement.csv' in r.text
assert client.get('/m/1999-01').status_code == 404            # unknown month
```

- [ ] **Step 2: Verify it fails**

Run: `nbdev-test --path nbs/01_app.ipynb`
Expected: FAIL — `/m/2025-07` returns 404 (route not defined), first status assert trips.

- [ ] **Step 3: Implement**

Add inside `create_app`, after the `index` route, before `return app`:

```python
    @rt('/m/{month}', methods=['GET'])
    def month_view(month: str):
        if month not in list_months(archive_dir): raise HTTPException(404)
        return Titled(month,
            P(A('statement.pdf', href=f'/m/{month}/statement.pdf'), ' · ',
              A('statement.csv', href=f'/m/{month}/statement.csv')),
            NotStr(status_html(archive_dir, month)),
            P(A('← months', href='/')))
```

- [ ] **Step 4: Verify it passes**

Run: `nbdev-test --path nbs/01_app.ipynb`
Expected: PASS.

- [ ] **Step 5: Export, normalize, commit**

```bash
nbdev-export
find nbs -name '*.ipynb' -print0 | xargs -0 python /home/doyu/.claude/skills/nbdev-tdd/scripts/normalize_notebooks.py
git add nbs/01_app.ipynb reconcile_web/app.py reconcile_web/_modidx.py
git commit -m "feat: month view route"
```

---

### Task 8: Authed file routes

**Files:**
- Modify: `nbs/01_app.ipynb` (the `create_app` cell + a new test cell)

**Interfaces:**
- Consumes: `safe_file` (Task 5).
- Produces: `GET /m/{month}/statement.pdf`, `GET /m/{month}/statement.csv`, `GET /m/{month}/receipt/{name}` — `FileResponse` or 404; all behind auth.

- [ ] **Step 1: Failing test**

New plain test cell:

```python
root = make_archive()
app = create_app(archive_dir=root, password='pw', session_secret='s3cret')
client = TestClient(app, base_url='https://testserver')
client.post('/login', data={'password': 'pw'})

assert client.get('/m/2025-07/statement.pdf').content.startswith(b'%PDF')
assert client.get('/m/2025-07/statement.csv').status_code == 200
assert client.get('/m/2025-07/receipt/2025-07-29_openai.pdf').content.startswith(b'%PDF')
assert client.get('/m/2025-07/receipt/nope.pdf').status_code == 404
# downloads never bypass auth
anon = TestClient(app, base_url='https://testserver')
assert anon.get('/m/2025-07/receipt/2025-07-29_openai.pdf', follow_redirects=False).status_code == 303
```

- [ ] **Step 2: Verify it fails**

Run: `nbdev-test --path nbs/01_app.ipynb`
Expected: FAIL — statement.pdf request 404s (no route), `startswith(b'%PDF')` assert trips.

- [ ] **Step 3: Implement**

Add inside `create_app`, after `month_view`, before `return app`:

```python
    def _file(month, kind, name=None):
        try: return FileResponse(safe_file(archive_dir, month, kind, name))
        except FileNotFoundError: raise HTTPException(404)

    @rt('/m/{month}/statement.pdf', methods=['GET'])
    def statement_pdf(month: str): return _file(month, 'statement_pdf')

    @rt('/m/{month}/statement.csv', methods=['GET'])
    def statement_csv(month: str): return _file(month, 'statement_csv')

    @rt('/m/{month}/receipt/{name}', methods=['GET'])
    def receipt(month: str, name: str): return _file(month, 'receipt', name)
```

- [ ] **Step 4: Verify it passes**

Run: `nbdev-test --path nbs/01_app.ipynb`
Expected: PASS.

- [ ] **Step 5: Export, normalize, commit**

```bash
nbdev-export
find nbs -name '*.ipynb' -print0 | xargs -0 python /home/doyu/.claude/skills/nbdev-tdd/scripts/normalize_notebooks.py
git add nbs/01_app.ipynb reconcile_web/app.py reconcile_web/_modidx.py
git commit -m "feat: authed file download routes"
```

---

### Task 9: `serve` entrypoint + docs + finish (PR #2)

**Files:**
- Modify: `nbs/01_app.ipynb` (one export cell before the closing cell)
- Modify: `nbs/index.ipynb` ("How to use" section)

**Interfaces:**
- Consumes: `create_app` (Task 6).
- Produces: `serve(host='0.0.0.0', port=8000)`; documented run command.

- [ ] **Step 1: Add `serve`**

Export cell (no test — it's a thin uvicorn wrapper with nothing to assert without binding a port):

```python
#| export
def serve(
    host: str = '0.0.0.0',  # bind address
    port: int = 8000,       # port
):
    "Run the viewer under uvicorn; config comes from env"
    import uvicorn
    uvicorn.run(create_app(), host=host, port=port)
```

- [ ] **Step 2: Document usage in `nbs/index.ipynb`**

Replace the "How to use / Fill me in please!" markdown cell and the `1+1` code cell with one markdown cell:

````markdown
## How to use

A read-only viewer over a `reconcile-archive` checkout. Configuration is env-only:

| env | meaning |
|---|---|
| `ARCHIVE_DIR` | path to the reconcile-archive checkout |
| `APP_PASSWORD` | the single shared login password |
| `SESSION_SECRET` | session-cookie signing key (generate once per deployment) |

Run:

```sh
ARCHIVE_DIR=~/reconcile-archive APP_PASSWORD=... SESSION_SECRET=... \
  uvicorn --factory reconcile_web.app:create_app
```

or `python -c "from reconcile_web.app import serve; serve()"`. HTTPS termination is the
reverse proxy's job; the app must never be exposed directly.
````

- [ ] **Step 3: Full validation**

```bash
find nbs -name '*.ipynb' -print0 | xargs -0 python /home/doyu/.claude/skills/nbdev-tdd/scripts/normalize_notebooks.py
nbdev-prepare
nbdev-readme
```

Expected: all green; `README.md` regenerated with the usage section.

- [ ] **Step 4: Manual smoke test against the real archive**

```bash
ARCHIVE_DIR=$HOME/reconcile-archive APP_PASSWORD=testpw SESSION_SECRET=testsecret \
  python -c "from reconcile_web.app import serve; serve(host='127.0.0.1', port=8899)" &
curl -s --retry 10 --retry-connrefused --retry-delay 1 \
  -o /dev/null -w '%{http_code} %{redirect_url}\n' http://127.0.0.1:8899/   # expect 303 -> /login
curl -s -c /tmp/cj -d 'password=testpw' -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8899/login  # expect 303
kill %1
```

Note: over plain HTTP the Secure cookie won't round-trip in a browser — that's by design (HTTPS-only); the curl checks above only verify routing and login handling.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat: serve entrypoint and usage docs"
```

- [ ] **Step 6: PR #2 — STOP, ask the user**

Show `git log --oneline` since the branch point and the diffstat. **Ask the user to approve pushing and opening the PR** ("feat: FastHTML viewer app"). Do not push without approval.

---

## Self-Review Notes

- Spec coverage: login/logout (T6), month index + counts (T6), month view + statement links (T7), authed downloads (T8), `status.md` contract incl. unknown tokens + footer fail-closed `ValueError` (T3), link rewrite + raw-HTML escape + `/m/` href allowlist (T4), traversal/symlink/non-`.pdf` rejection (T5), no static auth exemption (T6), env fail-fast (T6 test), Secure/HttpOnly/Lax cookie (`sess_https_only=True`, Starlette defaults), pinned deps + run command (T1, T9). Non-goals untouched.
- Rate-limiting intentionally absent (spec: YAGNI).
- Spec's "unreadable month at request time → 500 with a short message, logged" is satisfied by
  Starlette's default exception handling (plain 500 response + logged traceback) — no custom
  handler is added, per YAGNI.
- `_quarto.yml`/docs pipeline untouched; `nbdev-prepare` keeps it consistent.
