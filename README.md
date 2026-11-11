# traverse-maps

Map data for the Traverse iPhone app: a catalog of **feature cells** — the lifts,
pistes, ski areas and roads of a 0.25° × 0.25° box, as GeoJSON — that the app fetches when someone downloads an offline map of an area, so
the downloaded map also knows which parts of a track were a lift ride, a run or a drive; and a list of **resorts** (below) the app offers
for download, one resort per download.

This repository holds **release assets only**; there is no source here. The cells are built by `ios/Scripts/mappack/build-cells.py` in the
app's repository and uploaded by hand.

## What is where

| Release tag | Asset | What it is |
|---|---|---|
| `catalog` | `catalog.json` | The list of cells that exist and the list of resorts. Replaced on every build. The app reads this one URL (`TRAVERSE_MAP_CATALOG_URL`). |
| `cells-<yyyymmdd>` | `<id>.geojson` | One cell per file, named by its south-west corner: `c<lat×4>_<lon×4>` (so `c167_179` is the box whose south-west corner is 41.75° N, 44.75° E; negative values take a leading `m`, e.g. `cm1_m280`). |

`catalog.json` is `{"format": "traverse-cells/1", "cellDegrees": 0.25, "cells": [...], "resorts": [...]}`; each cell row carries the cell's `id`, `bbox`
(`[west, south, east, north]`), asset `url`, `bytes`, `sha256`, the two source snapshot dates, the OSM `highwayClasses` the cell keeps and its
`attribution`. A cell is at most 5,000,000 bytes. A cell file is a GeoJSON `FeatureCollection` of curated features (`properties.layer` is
`lift`, `run`, `skiArea` or `highway`); a lift, run or highway with at least one vertex inside a cell's box is kept whole in that cell, including the part past its edge, so a feature crossing an edge sits in every cell that holds one of its vertices — a crossing line with no vertex inside a cell is not in that cell.

### Resorts

`resorts` is a top-level array added to the same `traverse-cells/1` format: a reader that does not know it (every app before 0.14.0) ignores
it, and a catalog without it is valid. A resort is a ski area — or several, see below — taken from OpenSkiMap's data for the countries the
build lists; the build prints its resorts, so each one can be checked against the data. A row:

```json
{"id": "osa:<smallest member ski-area id>", "name": "…", "country": "GE", "region": "…" , "footprint": [west, south, east, north],
 "bbox": [west, south, east, north], "lifts": 24, "runs": 62, "skiAreaIds": ["…"]}
```

- `id` — `osa:` plus the smallest of the member ski-area ids (compared as strings). It follows the data: if a resort's membership changes
  (OpenSkiMap adds, removes or re-splits a ski area) the smallest member, and so the id, can change.
- `name` — the members' OpenSkiMap names (the English part of a "ka, en, ru" string), most lifts and runs first, joined with ` · `; a resort
  with no named member is `Ski area · <region>` (or the country when there is no region).
- `country` — ISO 3166-1 alpha-2 code of the first member's first place; `region` — its region, or `null`.
- `footprint` — the box of the member ski areas' linked lifts and runs; `bbox` — the footprint grown by 2 km on each side: the area a resort
  download covers. Coordinates have 5 decimals.
- `lifts`, `runs` — how many distinct lifts and runs are linked to the members (at least 1 each: a ski area with no lift or no run, or one
  marked abandoned or disused, is not listed). `skiAreaIds` — the member ids, sorted.

Ski areas whose footprints lie within 3 km of each other are one resort (and one download): OpenSkiMap often splits one resort into several
named zones. The catalog is **merged** on every build: a cell that was not rebuilt keeps its row and its URL, so the catalog keeps naming
older `cells-<yyyymmdd>` releases — never delete a release while a published row's `url` points into it.

## Licence and attribution

The data is **derived from OpenStreetMap** and is therefore made available under the
[Open Database Licence (ODbL) 1.0](https://opendatacommons.org/licenses/odbl/1-0/) — this repository's cells are a derivative database, and the
share-alike terms of the ODbL apply to them: if you use them, you must keep this notice and attribute the sources, and a database you derive
from them is also under the ODbL.

- **© OpenStreetMap contributors** — the lifts, pistes, ski areas and roads originate from OpenStreetMap
  (<https://www.openstreetmap.org/copyright>). Roads are read from the Overpass API.
- **OpenSkiMap.org** — lift, piste and ski-area records come from OpenSkiMap's open exports
  (<https://openskimap.org>), which are themselves derived from OpenStreetMap.

Each catalog row repeats these credits in its `attribution` field, and the app prints them on every map that uses a cell.

## Using the cells

Everything needed is in the catalog: fetch `catalog.json`, pick the cells whose `bbox` meets your area, fetch each `url`, and check its
`sha256`. Several cells may carry the same feature; the identity of a feature is (`properties.layer`, `properties.id` — or `osmId` for a
highway). There is no API and no guarantee of availability: the assets are regenerated when the app's maps are refreshed.
