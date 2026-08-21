# Month ZIP download — Design

**Status:** design, awaiting user review (2026-08-21). Implements issue #1.

## Goal

One authed route that hands the accountant a month's **entire folder as one zip** —
statement files, receipts, her own reports, `status.md` — so she can store the month's
materials in a single download. Her request, verbatim (2026-08-13): *"If all the
attachments can be downloaded in one zip file, that will make material storing much
easier."*

## Core decision: the zip mirrors the month folder, recursively

`month_zip()` enumerates **everything under `<archive>/<month>/`** at request time and
zips it as-is, preserving relative paths (`statement.*`, `status.md`, `receipts/…`,
`reports/…`). No per-kind allowlist:

- The only request-controlled input is `month`. Enumerated paths come from the server's
  own filesystem, so `safe_file`-style content rules (basename-only, `.pdf`-only, known
  statement names) defend against nothing here — they would only make the zip disagree
  with the folder.
- The archive now files accountant reports per month (`<month>/reports/`, archive commit
  `126bbd4`), so the zip covers them with zero reports-specific code. Any future file
  kind added to the archive appears in the next download, no viewer change.
- Built on the fly with `io.BytesIO` + `zipfile.ZipFile` (deflated), a few MB per month.
  No disk writes, no cache — the read-only invariant is untouched.

**The one content check that stays: symlink containment — on every entry, not just
files.** For every enumerated entry (regular files, directories, symlinks alike),
`resolve()` must land inside the (resolved) month dir. The symlink-escape check is a
load-bearing security invariant of the per-file routes; the zip route must not become
the one path without it (a planted symlink would otherwise exfiltrate whatever it points
at to any downloader). Checking entries rather than files matters because `rglob` never
descends into directory symlinks (no-follow on all supported Pythons; verified on 3.12):
an outside-pointing directory symlink shows up as a single non-file entry and nothing
under it is enumerated — an `is_file()`-only pass would drop it before any check and its
subtree would **silently vanish from the zip**. A silent partial zip is the worst
failure for someone archiving "the month's materials", so the entry check turns it into
a loud one. **Any violation fails the whole zip** with `FileNotFoundError` (→ 404) —
fail closed, never a partial zip.

## Data layer — `nbs/00_archive.ipynb`

```python
def month_zip(
    archive_dir: str|Path,  # archive root
    month: str,             # month name like '2026-07'
) -> bytes:                 # zip of the whole month dir, paths relative to it
```

1. Validate like `safe_file`: `month` matches `MONTH_RE`; `mdir = (root/month).resolve()`
   is inside the resolved root (`is_relative_to`) and `mdir.is_dir()` — the `is_dir` is
   the only gate against nonexistent months, since `resolve()` (strict=False) raises
   nothing there. Anything else → `FileNotFoundError`.
2. Enumerate `sorted(mdir.rglob('*'))` — **all entries**, deterministic order. `rglob`
   never descends into directory symlinks, so an escaping one appears as exactly one
   entry here; filtering to files before the check would silently drop it (see above).
3. Per entry: `rp = p.resolve()` inside `safe_file`'s try/except (`ValueError,
   RuntimeError, OSError` → `FileNotFoundError`); require `rp.is_relative_to(mdir)` —
   else `FileNotFoundError`. Load-bearing here, not just parity: with all entries
   enumerated, no `is_file()` prefilter shields `resolve()` from loops or broken links.
4. Write only regular files: `zf.write(p, arcname=p.relative_to(mdir))` for entries
   where `p.is_file()`, into a `BytesIO`; return its bytes. (An inside-pointing
   directory symlink passes the check but is neither descended nor written; an
   inside-pointing file symlink is included under its own name — harmless.)

Pure, UI-free, same layering as the rest of `archive.py`.

## App layer — `nbs/01_app.ipynb`

- `GET /m/{month}/all.zip` → `_check_month(month)`, then
  `Response(month_zip(...), media_type='application/zip')` with
  `Content-Disposition: attachment; filename="ninjalabo-<month>.zip"`.
  `FileNotFoundError` → 404, same as `_file()`.
- Link: append `all.zip` to `statement_links()` — it then shows on both the month page
  and the HTMX expand row with no extra wiring. A listed month always has `status.md`,
  so the link is never dead.
- Auth: the existing `Beforeware` gate covers the route (skip list is `/login` only) —
  nothing to add.

## Testing (inline asserts, `make_archive()` tmpdir fixture)

Data layer:
1. `namelist()` equals every file under the month dir as sorted relative paths,
   including `receipts/…` and `reports/…`.
2. A sample member's bytes equal the source file's bytes.
3. A **file** symlink escaping the month dir → `FileNotFoundError` (whole zip refused).
4. A **directory** symlink pointing outside → `FileNotFoundError` — pins the fail-closed
   choice: its subtree must never silently vanish from the zip.
5. Invalid month (`'../x'`, `'2026-7'`, nonexistent) → `FileNotFoundError`.

App layer (TestClient):
6. Unauthenticated → 303 to `/login`.
7. Authenticated → 200, `application/zip`, attachment `Content-Disposition`; the body
   opens as a zip and lists the expected names.
8. Unknown month → 404.
9. The month page contains the `all.zip` link.

## Out of scope (YAGNI)

- Whole-period / multi-month zip — not requested; per-month matches her filing.
- Caching, streaming, zip64 — months are a few MB.
- Browsing UI for `reports/` on the month page — issue #9, decided separately after
  this ships.

## Rollout

Branch `feat/month-zip` → nbdev-tdd workflow (stub → failing asserts → implement) →
`nbdev-prepare`. PR (closes #1), merge, and `reconcile-deploy` are **each gated on
explicit approval** — deploy is never an automatic follow-on of merge. The PR also:

- updates CLAUDE.md's security invariants — `safe_file` is no longer the sole serving
  path; the zip route applies the same resolve-based containment to every entry it
  bundles;
- refreshes the stale allowlist-era design comment on issue #1 to match this spec.
