# Where the dataset comes from

The dashboard's dataset is a **point-in-time snapshot** of every `cq:Page` that is
actually live on a Southern Company AEM publisher, joined with author metadata
(titles, dates, replication, page type) to compute freshness bands and flags.

The encrypted dataset contains two separate snapshots:

| Tab | Snapshot | Sites | Published pages | Live article CFs |
|---|---|---:|---:|---:|
| Production | 2026-09-22 | 14 | 3,675 | 3,756 |
| Stage | 2026-10-02 | 14 | 6,686 | 3,515 |

Source workbooks are `build/report/Published-Pages-Report-Prod-2026-09-22.xlsx`
and `build/report/Published-Pages-Report-Stage-2026-10-02.xlsx`. Production is
preserved from its existing snapshot. Stage enumerates each reachable publisher
independently and combines their paths; empty leftover page shells are excluded.

Stage website and sitemap requests are anonymous. Protected website responses
(HTTP 401/403) are **unverified**, not broken. Unavailable sitemaps yield unknown
coverage, not missing entries. The dashboard shows these limitations and only
uses verified checks in URL-health and sitemap-coverage figures.

The workbook is produced by `build/report/index.js` (Node, axios + exceljs). That
directory is part of `build/`, so it exists only on the **private** mirror — the
public repo never gets the scraper, the Excel, or the author credentials. The
script talks to AEM author + publishers; pass `AEM_USER` / `AEM_PASS` (see the
header of `index.js`). Do not paste those into this README, issues, or the
public tree.

## How the report is built (join)

1. **Publisher is the source of truth for “published.”** Every renderable `cq:Page`
   (a `cq:Page` node that still has `jcr:content`) under each site root is
   enumerated on reachable publishers. Empty leftover shells from a deactivate
   that removed content are skipped — they 404 on the live site.
2. **Author is metadata enrichment.** The same paths are looked up on author for
   `jcr:title`, `cq:lastModified` (falling back to `jcr:created`), `cq:lastReplicated`,
   `cq:lastReplicationAction`, and `sling:resourceType`.
3. Freshness and flags are computed on the join, then written to one sheet per site
   plus Overview / Articles Overview / Sections rollups.
4. Each site’s public sitemap is fetched; pages and article CFs are flagged
   **In Sitemap**.
5. Live URLs are probed (redirects + soft-404) → **URL OK / URL Status**.
6. Virtual articles under `/content/dam/global/articles` are enumerated on the
   publisher (live CFs only), attributed by brand folder, and appended to each site
   tab under an `ARTICLES` banner.

`build/convert_xlsx.py` reads the **per-site sheets only**. Overview, Articles
Overview, and Sections are skipped on purpose — the app recomputes those aggregates
so they cannot drift.

Article CFs (`/content/dam/global/articles/...`, or rows after the `ARTICLES`
banner) are split onto `site.articles[]` with their own `totals.articles`. Page
totals stay pages-only so the ~3,700 CFs never inflate section counts. The
dashboard counts **live (publisher)** article CFs only — matching pages — not the
Articles Overview “Total Article CFs” figure that also includes unpublished
author-only fragments.

## Freshness bands

Effective date is `cq:lastModified`, falling back to `jcr:created`. Missing or
unparseable dates count as Very Stale.

| Band | Default threshold |
|---|---|
| Fresh | newer than 1 year |
| Aging | 1–2 years |
| Stale | 2–3 years |
| Very Stale | 3+ years, or no usable date |

Thresholds are stored on the JSON as `thresholdsYears` (`fresh` / `aging` / `stale`).

## Flags

- **Live Out-of-Date** — `cq:lastModified` is newer than `cq:lastReplicated`. Author
  has unpublished changes, so the live copy is behind. Both dates must be present.
- **Not On Author** — the path exists on a publisher but not on author (orphan or
  deactivated on author).
- **Missing cq:lastModified** — no last-modified date; age may be from `jcr:created`
  or unknown.
- **Not in sitemap** — path is live but absent from that site’s public sitemap.
- **URL fail** — live probe did not return a healthy page (HTTP error or soft-404).
  Soft-404 detection means URL-fail counts are higher than raw HTTP 4xx.

## Sites in this snapshot

| Site | Content path | Live domain |
|---|---|---|
| Southern Telecom | `/content/southerntelecom` | https://www.southern-telecom.com |
| Southern Power | `/content/southernpower` | https://www.southernpowercompany.com |
| Alabama Power | `/content/alabama-power` | https://www.alabamapower.com |
| Georgia Power | `/content/georgia-power` | https://www.georgiapower.com |
| Mississippi Power | `/content/mississippi-power` | https://www.mississippipower.com |
| Southern Nuclear | `/content/southern-nuclear` | https://www.southernnuclear.com |
| Southern Company | `/content/southerncompany` | https://www.southerncompany.com |
| Florida Natural Gas | `/content/florida-natural-gas` | https://www.onlyfng.com |
| AGL | `/content/southern-co-gas/agl` | https://www.atlantagaslight.com |
| Nicor Gas | `/content/southern-co-gas/nicor-gas` | https://www.nicorgas.com |
| Virginia Natural Gas | `/content/southern-co-gas/virginianaturalgas` | https://www.virginianaturalgas.com |
| Chattanooga Gas | `/content/southern-co-gas/chattanooga-gas` | https://www.chattanoogagas.com |
| Southern Company Gas | `/content/southern-co-gas/southerncompanygas` | https://www.southerncompanygas.com |
| 1 Minute Kitchen | `/content/southern-co-gas/1minute-kitchen` | https://www.1minute.kitchen |

Author editor links use `https://author.southerncompany.com/editor.html` for
Production and `https://author-stage.southerncompany.com/editor.html` for Stage.
Stage live links use the configured `stage.` website domains.

## Schema

Top-level object (schema version 2):

```json
{
  "schemaVersion": 2,
  "defaultEnvironment": "prod",
  "environments": {
    "prod": {
      "generated": "2026-09-22",
      "environment": "prod",
      "thresholdsYears": { "fresh": 1, "aging": 2, "stale": 3 },
      "sites": []
    },
    "stage": {
      "generated": "2026-10-02",
      "environment": "stage",
      "thresholdsYears": { "fresh": 1, "aging": 2, "stale": 3 },
      "sites": []
    }
  }
}
```

Each snapshot has its own site list, dates, links, and totals. The app can also
read legacy single-environment exports. `refresh.py` merges the selected snapshot
into the existing bundle; updating Stage never replaces Production, and vice versa.


Each site: `{ name, root, liveBase, authorBase, totals, pages, articles }`.

`totals` is recomputed by the converter from the **page** list (`published`, `fresh`,
`aging`, `stale`, `veryStale`, `oldestYears`, `missingLastModified`, `liveOutOfDate`,
`notOnAuthor`, `inSitemap`, `notInSitemap`, `urlOk`, `urlFail`). The same shape is
nested at `totals.articles` for live article CFs. Token-estimate integers are stored
on each row for a later export; they are not shown in the dashboard.

Each page:

| Field | Meaning |
|---|---|
| `path` | Full AEM path |
| `title` | `jcr:title` (empty when not on author) |
| `band` | `fresh` / `aging` / `stale` / `veryStale` |
| `lastModified` | Effective date (`YYYY-MM-DD`), from last-modified or created |
| `lastReplicated` | `cq:lastReplicated` (`YYYY-MM-DD`) |
| `yearsSinceUpdate` | Fractional years to 1 decimal, or `null` |
| `daysSinceModified` | Whole days, or `null` |
| `daysSincePublished` | Whole days since last replicate, or `null` |
| `dateSource` | `cq:lastModified`, `jcr:created`, or `(unknown)` |
| `missingLastModified` | `true` when `cq:lastModified` was absent |
| `liveOutOfDate` | Author newer than last publish |
| `onAuthor` | `false` for publisher-only orphans |
| `resourceType` | `sling:resourceType` |
| `replicationAction` | e.g. `Activate` |
| `liveUrl` | Public URL (`.html` mapping) |
| `authorUrl` | Author editor URL |
| `inSitemap` | `true` / `false` / `null` (blank in Excel) |
| `urlOk` | `true` / `false` / `null` after the live probe |
| `urlStatus` | Probe result (status or soft-404 note) |
| `articlePublishDate` | Article CF publish date (blank on most pages) |
| `primaryTag` | Primary tag when the scrape recorded one |
| `liveHtmlTokens` / `liveTextTokens` / `cfTokens` / `combinedTokens` | Token estimates (JSON only) |

`articles[]` uses the same row shape. Article paths live under
`/content/dam/global/articles`.

## How to refresh

All of this is run from `build/` on a machine that has the private working copy
(or the public clone, where `build/` is local and gitignored).

**New Excel you already have**

```bash
cd build
python3 refresh.py --xlsx report/Published-Pages-Report-Prod-YYYY-MM-DD.xlsx
```

That converts the workbook → `raw/live-content-data.json` → encrypted
`../live-content-data.json`. `index.html` is untouched (`--data-only`). Pass
`--full` as well when `template.html` changed, so the login shell is rebuilt
with the same content key.

**End to end (scrape AEM, then convert + encrypt)**

```bash
cd build
# first time in report/: npm install  (refresh.py does this)
AEM_USER=... AEM_PASS=... python3 refresh.py --fetch --env prod
# or update the Stage tab, preserving Production:
AEM_USER=... AEM_PASS=... python3 refresh.py --fetch --env stage
```

`report/index.js` writes `Published-Pages-Report-{Prod|Stage}-YYYY-MM-DD.xlsx` next
to itself. `refresh.py` converts the newest workbook matching the selected
environment; `--env both` refreshes both. No matching workbook stops the refresh.
The initial migration rebuilds the app automatically; later refreshes are data-only
unless `--full` is supplied. Credentials belong in environment variables, never
in the report source.

Commit `live-content-data.json` on the public repo; run `publish.py` to update
the private mirror (xlsx + scraper included).

The content key is reused, including when the app is rebuilt.

## Excel download

The dashboard generates a fresh `.xlsx` from the selected environment’s loaded
snapshot. It includes an Overview per site plus all Pages and Articles, regardless
of active search, site selection, or filters. Filenames include the environment
and snapshot date. Dates and numbers retain their Excel types, links are clickable,
and unavailable sitemap/URL checks are labeled `Unverified`. Malformed dates
already present in the source remain unchanged text.

ExcelJS 4.4.0 and the export worker are embedded inside the encrypted app during
build. Downloads run entirely in the browser; no data is sent to an export service
and no plaintext workbook is added to the public repository. The version, hash,
and MIT license are retained in private `build/vendor/`.
