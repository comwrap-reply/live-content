# Where the dataset comes from

The dashboard's dataset is a **point-in-time snapshot** of every `cq:Page` that is
actually live on a Southern Company AEM publisher, joined with author metadata
(titles, dates, replication, page type) to compute freshness bands and flags.

This snapshot was converted from:

    Published-Pages-Report-Prod-2026-08-24.xlsx

14 sites, **3,801** published pages, production only. Stage is not in this file;
drop a `Published-Pages-Report-Stage-*.xlsx` through `build/convert_xlsx.py` if you
need it later (the JSON has an `environment` field).

The workbook is produced by a separate script that is **not** part of this repo
(it talks to AEM author + publishers and holds author credentials):

    ~/Documents/Work/scripts/published-pages-report/index.js

Do not copy that script, its hosts, or its passwords into either GitHub repo.

## How the report is built (join)

1. **Publisher is the source of truth for “published.”** Every `cq:Page` under each
   site root is enumerated on reachable publishers. If the path is on a publisher,
   it is live.
2. **Author is metadata enrichment.** The same paths are looked up on author for
   `jcr:title`, `cq:lastModified` (falling back to `jcr:created`), `cq:lastReplicated`,
   `cq:lastReplicationAction`, and `sling:resourceType`.
3. Freshness and flags are computed on the join, then written to one sheet per site
   plus Overview / Sections rollups.

`build/convert_xlsx.py` reads the **per-site sheets only**. Overview and Sections are
skipped on purpose — the app recomputes those aggregates from the page list so they
cannot drift.

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

Author editor links use `https://author.southerncompany.com/editor.html`.

## Schema

Top-level object:

```
{
  "generated": "2026-08-24",
  "environment": "prod",
  "thresholdsYears": { "fresh": 1, "aging": 2, "stale": 3 },
  "sites": [ /* one object per site */ ]
}
```

Each site: `{ name, root, liveBase, authorBase, totals, pages }`.

`totals` is recomputed by the converter from the page list (`published`, `fresh`,
`aging`, `stale`, `veryStale`, `oldestYears`, `missingLastModified`, `liveOutOfDate`,
`notOnAuthor`).

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

## How to refresh

1. Re-run the Published Pages Report against AEM (outside this repo).
2. Convert the new workbook:

   ```bash
   python3 build/convert_xlsx.py /path/to/Published-Pages-Report-Prod-YYYY-MM-DD.xlsx
   ```

   That overwrites `build/raw/live-content-data.json`.
3. `cd build && python3 build.py --data-only`
4. Commit `live-content-data.json` on the public repo; run `publish.py` to update the
   private mirror.

`index.html` is untouched. The content key is reused.
