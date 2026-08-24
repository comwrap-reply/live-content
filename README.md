# Live Content

An interactive, credential-gated dashboard of every **live published page** across
the Southern Company AEM portfolio — freshness bands, unpublished author changes,
and publisher-only orphans — so a team can see what is actually on the live sites
and what needs attention.

The site is a small static app plus one encrypted data file. The app itself and the
dataset are **AES-256-GCM encrypted**; a single content key is wrapped per user with a
key derived from their `username:password` (PBKDF2-SHA256, 310k iterations), so the
files are safe to host on a public GitHub Pages URL. Credentials are distributed
out-of-band — never commit them to this repo, this README, or any issue/PR.

This is a **read-only inventory**. Refreshing the data replaces the JSON; there is no
workshop state file.

## What's in the tool

- **Overview** — portfolio-wide freshness bar, KPI cards (% fresh, live out-of-date,
  broken live URLs, % in sitemap, oldest page), cross-site search, and a site list
  sortable by needs-attention, size, or name. Site rows show out-of-date / broken
  counts when non-zero, plus live article CF counts.
- **Site detail** — Pages | Articles toggle, per-kind bar and KPIs, a section
  breakdown on pages (first path segment after the site root), band toggles, flag
  filters (live out-of-date, not on author, missing modified date, not in sitemap,
  URL fail), an optional type filter, search, sort, and pagination.
- **Rows** — title, path, last modified, last published, days since published,
  Author and Live links, plus badges for the flags the Excel already computed.

## Repo layout

| Path | Committed? | What it is |
|---|---|---|
| `index.html` | yes — push | Login shell plus the app, gzipped then encrypted. Only changes when `template.html` or the set of users changes. |
| `live-content-data.json` | yes — push | The dataset (sites + pages), gzipped then encrypted (`EDSENC2` envelope). Replace this to refresh the data. |
| `README.md` / `SOURCES.md` / `.gitignore` | yes — push | This file, where the dataset comes from, and the rule that keeps build inputs out of the public repo. |
| `build/` | no — keep out of the public repo | Everything needed to rebuild. **Never push to the public repo** — `users.txt` holds the passwords, `raw/live-content-data.json` is the plaintext dataset, and `template.html` is the unencrypted app. Shared through the private mirror instead (see `build/publish.py`). |

## Deploying

1. Push `index.html`, `live-content-data.json` (plus this README / SOURCES.md / `.gitignore`).
2. Repo → Settings → Pages → deploy from branch → `main`, root (`/`).
3. The site serves at `https://<user>.github.io/<repo>/`.

> GitHub Pages sites are publicly reachable at their URL even when the repo is private.
> That is fine here — the content is ciphertext without a login. Keeping the repo private
> additionally protects the encrypted files at the source and is preferred.

## Refreshing the data

The scrape script, the Excel snapshot, and the converter all live in `build/`
(private mirror only — never on this public repo). One command does convert +
re-encrypt:

```bash
cd build
pip install -r requirements.txt      # once

# From an existing / new workbook
python3 refresh.py --xlsx report/Published-Pages-Report-Prod-YYYY-MM-DD.xlsx

# Or scrape AEM, write a new xlsx, convert, and encrypt (needs Node + AEM_PASS)
python3 refresh.py --fetch --env prod
```

`--data-only` is the default, so `index.html` and the login stay as they are.
Commit the regenerated `live-content-data.json`, then `python3 publish.py`.
See [`SOURCES.md`](SOURCES.md) for the join, schema, and `build/report/` layout.

Drop `--data-only` when you have also changed `template.html` and want a new `index.html`.
That reuses the existing content key as long as a listed user still unlocks the current
`index.html`. The build prints `content key reused from index.html` when this happens.

## Credentials

Logins live in `build/users.txt`, one `username:password` per line. The content key is
**reused by default** — unwrapped from the existing `index.html` using any listed user
whose password still verifies — so refreshing data or editing the app never invalidates
deployed data.

- **Add a user:** add a line to `users.txt` and run a full build. The same content key is
  wrapped for them; everyone else is untouched.
- **A user already in `index.html` but not listed** is **carried forward** untouched.
- **Rotate the content key:** `python3 build.py --rotate-key`. This mints a brand-new key.
  It refuses to run if a carried-forward user would be locked out, so list everyone
  (with passwords) first.

## Running it locally

The dataset is fetched at runtime, and browsers block `fetch` between `file://` pages, so
serve the folder instead:

```bash
cd <repo root> && python3 -m http.server 8000   # then open http://localhost:8000/
```

Opening `index.html` straight off disk still works: after you log in, the app notices the
fetch failed and shows a banner with a file picker for `live-content-data.json`.

## Security model (honest version)

- `index.html` and `live-content-data.json` are real ciphertext — there is nothing
  readable in the repo or on the wire without a login.
- A single content key encrypts both; each user's `username:password` unwraps a copy of
  it. Anyone with the URL **and** a login has everything; treat the password like the
  content itself.
- Because the ciphertext is public, offline brute-force against a weak password is
  possible in principle. Rotate to a longer passphrase (`--rotate-key`) if the audience
  for this repo widens.
- Login persists per browser tab (sessionStorage); closing the tab locks the tool.
