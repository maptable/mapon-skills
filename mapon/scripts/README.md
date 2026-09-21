# MapOn CLI


A zero-dependency Python 3 script that drives the MapOn MCP endpoint
(`/mcp/`, stateless JSON-RPC) to build maps from data — the default way to
drive MapOn. File upload keeps its dedicated multipart endpoint and
login keeps its REST endpoint.
(An MCP client can call the same tools too; `python3 mapview_cli.py endpoint`
prints its URL.) `python3 mapview_cli.py tools` prints the live tool catalog — the
authoritative list, including tools newer than this script.

## Setup

1. **Connect (guided)** — run `login` with no arguments: it prints a
   login link, you finish it in a browser (login, signup, and workspace
   selection work exactly as on the website), and the token is stored
   automatically.
   ```bash
   python3 mapview_cli.py login
   ```
   Prefer a manual token? Generate one in the MapOn UI
   (**Settings → Access Token**, starts with `mapview_pat_`) and store it:
   ```bash
   python3 mapview_cli.py login --token "mapview_pat_xxx"
   ```

2. **Token storage** (one time, per user — every command reads it
   automatically; run `python3 mapview_cli.py status` first to check whether one is
   already stored): `login` writes the platform's per-user config dir —
   `~/.config/mapview/tokens.json` (Linux and macOS;
   `$XDG_CONFIG_HOME/mapview/tokens.json` when the var is set),
   `%APPDATA%\mapview\tokens.json` (Windows). Multiple hosts and workspaces
   coexist there — `switch` replaces only the locally active token; the
   previous workspace's token stays valid server-side.

3. **Run** (Python 3.9+, standard library only — no `pip install`):
   ```bash
   python3 mapview_cli.py <command> [options]
   ```

## Commands

### login
Connect this machine to MapOn. With no arguments it runs the guided
browser login: it opens a login request, prints the URL, polls until you
finish in the browser, and stores the delivered token. With `--token
<value>` it just stores that token; `--token -` reads it from stdin
(explicit — an implicit stdin read would hang under agents with an open
pipe). Replacing an existing token is reported.
```bash
python3 mapview_cli.py login                # guided browser login
python3 mapview_cli.py login --token "mapview_pat_xxx"
```

### switch
Connect to a different workspace: re-runs the guided browser login and stores
the new token. The login link carries a pick flag, so the browser asks you to
choose a workspace instead of auto-entering your current one. The previous
workspace's token stays valid server-side — PATs coexist per org; only the
locally active token is replaced.
```bash
python3 mapview_cli.py switch
```

### status
Show whether an access token is configured and where it comes from, plus the
effective host and every stored workspace slot on it. Local check only —
token validity surfaces on the first API call. Exits non-zero when no token
is set.
```bash
python3 mapview_cli.py status
```

### tools
List the server's **live tool catalog** — the authoritative list of every tool,
including ones newer than this CLI. With no arguments prints one line per tool
(name + short description); with a tool name prints that tool's full
description and input schema. Use it whenever a typed subcommand is missing or
a call fails with an unknown tool/parameter error.
```bash
python3 mapview_cli.py tools
python3 mapview_cli.py tools update_slicers
```

### call
Call **any** tool with a JSON payload — the generic escape hatch that makes
every server-side tool usable even without a typed subcommand. The payload is
inline JSON (`--json`) or a file/stdin (`--input`). Parameter names and types
come from the tool's schema (`tools <name>`).
```bash
python3 mapview_cli.py call get_map --json '{"mapId": 123}'
python3 mapview_cli.py call some_new_tool --input payload.json
```

### create_map
Create a new map. Returns `{mapId, shareUrl}`.
```bash
python3 mapview_cli.py create_map --name "Sales by Province" --desc "Q3 2026"
```

### add_layer
Add a data layer to a map. The payload (mapId, layerType, columns, rows,
fieldMappings, styleValues) is read from a JSON file or stdin.
```bash
python3 mapview_cli.py add_layer --input layer.json
# or via stdin:
cat layer.json | python3 mapview_cli.py add_layer
```
See `layer.json` example shape below.

### upload
Upload image files to a map for **attachment columns** (info-window cover
images). Prints one line per image — `fileID  fileName` — ready to paste into
attachment cells. Limits: ≤20 files per call, ≤5MB each, ≤100MB total; image
types only (jpg, jpeg, png, gif, webp — no bmp/svg). Upload each unique image
**once** and reuse its fileID across rows — every upload stores a new object.
`--name` renames the stored file (single file only).
(Agent clients without the CLI can POST the same endpoint as JSON:
`{mapId, data: <standard base64>, name}`.)
```bash
python3 mapview_cli.py upload 123 photo1.jpg photo2.png
python3 mapview_cli.py upload 123 tmp8f3c.png --name "Store front.jpg"
```

### upload_html
Host a **self-contained HTML file** on a map and get its public viewer URL —
anyone with the link opens it in a browser, no login, independent of the
map's sharing switch. The file must be fully self-contained (inline CSS/JS,
data: URI images; ≤5MB). `--name` records a document name; `--type` selects
the frontend page the URL opens (only `report` today). Prints
`htmlId  url` — relay the url as-is. No in-place update: re-hosting yields a
new URL; `delete_html` the old one when replacing. `get_map` lists the map's
hosted pages under `hostedHtml`.
```bash
python3 mapview_cli.py upload_html 123 report.html --name "Q3 sales report"
```

### delete_html
Delete a hosted HTML page by passing back its public URL — the exact url
`upload_html` printed or `get_map` listed. The link stops working
immediately. Editor permission.
```bash
python3 mapview_cli.py delete_html "https://…/project/123/report?reportID=…"
```

### list_maps
List all active maps in the workspace. Returns `[{mapId, name, desc, layerCount, updateTime}, ...]`.
```bash
python3 mapview_cli.py list_maps
```

### get_map
Get one map's full state + its layer list. Returns `{mapId, name, desc, shareUrl, baseMap, center, zoomLevel, layers: [{layerId, name, layerType, rowCount}, ...]}`.
```bash
python3 mapview_cli.py get_map --map-id 123
```

### update_layer
Update a layer in place — data (`--mode replace|append|upsert`), visualization, name, comparison, or order. Only what you pass changes. Data updates only work on layers created via add_layer.
```bash
# restyle: new palette + classification on an existing layer
python3 mapview_cli.py update_layer --map-id 123 --layer-id "abc123" \
  --input style.json          # {"styleValues": {"colors": [...], "fillMethod": "quantile"}}

# append new rows (incremental columns optional)
python3 mapview_cli.py update_layer --map-id 123 --layer-id "abc123" --mode append --input rows.json

# fix rows by id: provided fields overwrite, unmentioned fields survive
python3 mapview_cli.py update_layer --map-id 123 --layer-id "abc123" --mode upsert --input fixes.json

# rename only
python3 mapview_cli.py update_layer --map-id 123 --layer-id "abc123" --name "Q4 Sales"

# move within the layer order (top renders above the others; bottom is the
# default-shown position for comparison groups) / toggle comparison membership
python3 mapview_cli.py update_layer --map-id 123 --layer-id "abc123" --order top
python3 mapview_cli.py update_layer --map-id 123 --layer-id "abc123" --comparison true
```

### get_layer_data
Read back a layer's stored columns + rows (paged, projectable, filterable by row id). Use it to verify writes or recover auto-generated row ids after append. `--formatted` adds each row's `displayText` — display-ready strings (`2024/03/01`, `¥1,234.56`, `15%`) for reporting.
```bash
python3 mapview_cli.py get_layer_data --map-id 123 --layer-id "abc123" --limit 100
python3 mapview_cli.py get_layer_data --map-id 123 --layer-id "abc123" --columns province sales
python3 mapview_cli.py get_layer_data --map-id 123 --layer-id "abc123" --ids row-1 row-2
python3 mapview_cli.py get_layer_data --map-id 123 --layer-id "abc123" --formatted
```

### update_map
Update map-level settings: name, description, basemap, viewport, share link, or share password. Only what you pass changes.
```bash
python3 mapview_cli.py update_map --map-id 123 --name "Q3 Sales" --base-map dark --center "116.4,39.9" --zoom 4

# share link: publish an unshared map (e.g. one built in the web editor —
# maps from create_map are shared by default) or un-publish; the response
# reports shared + shareUrl
python3 mapview_cli.py update_map --map-id 123 --share on

# share password: server-generated random, returned in cleartext
# (--password on also enables sharing when --share is not given; --password off drops the password only and never turns sharing on)
python3 mapview_cli.py update_map --map-id 123 --password on
python3 mapview_cli.py update_map --map-id 123 --reset-password
```

### delete_layer
Delete a layer from a map entirely (data files included). Prefer update_layer for corrections; delete only when the layer should not exist.
```bash
python3 mapview_cli.py delete_layer --map-id 123 --layer-id "abc123"
```

### get_quota
Read-only quota status: geocoding usage, map count, and per-map layer/row limits.
```bash
python3 mapview_cli.py get_quota
```

### list_slicers
List the map's slicers — the interactive filter controls shown on the map and share page. Returns each slicer's id, name, kind, `(layerId, columnId)` sources, and default filter.
```bash
python3 mapview_cli.py list_slicers --map-id 123
```

### update_slicers
Add, update (replace wholesale by id), remove, or reorder the map's slicers. Complex payloads are read from `--input <file>` or stdin.
```bash
# add a plain slicer filtering one column, defaulting to equals "Guangdong"
cat <<'EOF' | python3 mapview_cli.py update_slicers --map-id 123 --mode add
{"name": "Province", "sources": [{"layerId": "abc", "columnId": "col_prov"}],
 "operator": "equals", "value": "Guangdong"}
EOF

# add a rolling-window datetime default (last month)
cat <<'EOF' | python3 mapview_cli.py update_slicers --map-id 123 --mode add
{"name": "Updated", "sources": [{"layerId": "abc", "columnId": "col_date"}],
 "datetimeRange": {"type": "last", "duration": "P1M"}}
EOF

# add a cascading slicer (province → city drill-down)
cat <<'EOF' | python3 mapview_cli.py update_slicers --map-id 123 --mode add
{"name": "Region", "kind": "cascading", "levels": [
  {"name": "Province", "sources": [{"layerId": "abc", "columnId": "col_prov"}]},
  {"name": "City", "sources": [{"layerId": "abc", "columnId": "col_city"}]}]}
EOF

python3 mapview_cli.py update_slicers --map-id 123 --mode remove --slicer-id <slicerId>

# replace rebuilds the full list; array order = display order
cat <<'EOF' | python3 mapview_cli.py update_slicers --map-id 123 --mode replace
[{"id": "<keep-first>", "name": "Province", "sources": [{"layerId": "abc", "columnId": "col_prov"}]},
 {"id": "<keep-second>", "name": "Category", "sources": [{"layerId": "abc", "columnId": "col_cat"}]}]
EOF
```

### geocode
Batch-geocode addresses to lng/lat (≤100 per call). Each call consumes the workspace's geocoding quota, misses included — tell the user before the first call of a session; `get_quota` shows the balance.
```bash
python3 mapview_cli.py geocode --addresses "北京市朝阳区" "上海市浦东新区" --country CN
```

### region_match
Match admin levels / coordinates to region UIDs (≤100 per call). Put each region name in the adminN field of its level (`admin1` province, `admin2` city, `admin0` country) — free-text `address` is not accepted.
```bash
python3 mapview_cli.py region_match --input items.json
```

## Local payload builders (fmt-*)

Pure-local commands that turn agent-side data into the canonical columns/rows JSON the write API requires — no request, no token. Output goes to stdout; feed it to `add_layer`/`update_layer` here, or paste it as the MCP tool arguments. Column ids **always come from your input** (datasource field id, an index, or a unique column name) and pass through verbatim — keeping the datasource↔column matching is your job; the tools never generate or map ids.

Validation runs locally before anything is submitted: structural errors (missing/duplicate ids, unknown row keys, ragged CSV lines) and vocabulary checks against the script's built-in vocabulary snapshot (no request; when the server rejects a value anyway, its error lists the accepted options). Types left undeclared are inferred conservatively (`multiLineText` is the safe fallback; report on stderr).

### fmt-columns
Loose column definitions → canonical columns JSON.
```bash
echo '[{"id": "fldA1", "name": "销售额", "type": "number"},
       {"id": "0", "name": "门店", "type": "singleChoice",
        "typeOptions": {"choices": [{"name": "高", "color": "redLight"}, {"name": "中"}]}}]' \
  | python3 mapview_cli.py fmt-columns > columns.json
```
- `id` is required and passed through verbatim; `name` defaults to `id`.
- `type` may be omitted → inferred from `--rows records.json` (sample), else `multiLineText`; `--strict` rejects undeclared types instead.
- Choice colors are palette names (`redLight`, `blueDark`, …) — hex is rejected.

### fmt-rows
Records → canonical rows JSON (cells keyed by column id).
```bash
echo '[{"id": "row-7", "fldA1": 6573, "0": "朝阳店"}, {"fldA1": 99}]' \
  | python3 mapview_cli.py fmt-rows --columns columns.json > rows.json
```
- Record keys must be column ids (unknown keys fail with the declared id list); row `id` passes through, id-less rows stay id-less (append assigns one server-side). If a declared column is literally named `id`, a flat record's `id` value goes to that column — pass a row id via the nested form `{"id": ..., "cells": {...}}`.
- `--columns` accepts a columns array, a `fmt-csv` output, or a whole `get_layer_data` response — the last one formats new rows against an existing layer for append/upsert.

### fmt-csv
One-shot CSV file → `{columns, rows}`.
```bash
python3 mapview_cli.py fmt-csv --file data.csv --id index
python3 mapview_cli.py fmt-csv --file data.csv --id name --typed-header
```
- `--id index` (default): column position (`"0"`, `"1"`, …) is the id — stable while the column order holds.
- `--id name`: the header name is the id and must be unique. **Boundary**: renaming a column later (in the web editor) changes the name you see in `get_layer_data`, but stored ids never change — for datasources you will keep updating, prefer the datasource's native ids or `--id index`; use `--id name` for one-shot imports.
- `--typed-header`: header cells carry inline types (`销售额:number,开业日期:datetime`).
- Unnamed header cells (a trailing comma, an unnamed export column) fail with the column position — name the column or drop it.
- All-empty columns (a declared column blank in every data row) fail locally with the column named — the server rejects the same write; drop the column or fix the export that lost its values.
- Values pass through verbatim (dates/amounts as strings are fine — the server converts by declared type); empty cells become `null`; ragged lines fail with the line number.

## Coordinate conversion (convert-coords)

The map stores and renders **WGS84 only**. Geolocation fields of Chinese table apps — 飞书多维表格 (Feishu Bitable), 钉钉 (DingTalk), 企业微信 (WeCom), 腾讯文档/腾讯表格 (Tencent Docs/Sheets) — and any Amap/高德/Tencent-sourced coordinates are **GCJ-02**; Baidu's are **BD-09**. Submitted raw, points land 100–700 m off in China with no error — the numbers look valid. Convert before `add_layer`/`update_layer` (and before `update_map --center`). Pure local math (same algorithm as the server-side datasource connectors): no request, no token.

```bash
# one point
python3 mapview_cli.py convert-coords --from gcj02 --lng 116.410244 --lat 39.916405

# rewrite the two columns of a records/rows file in place (--out may equal --input)
python3 mapview_cli.py convert-coords --from gcj02 --input records.json --lng-col lng --lat-col lat --out records.json

# coordinates in one "lng,lat" string column (string shape is preserved)
python3 mapview_cli.py convert-coords --from bd09 --input records.json --coord-col location --out records.json
```

- `--input` accepts a flat records array, canonical rows (`[{"cells": {...}}]`, e.g. `fmt-rows` output), or an object with a `rows` array (`get_layer_data` response).
- Cells may be numbers or numeric strings; output is rounded to 6 decimals (~0.1 m).
- Rows with missing/non-numeric coordinates are counted and left untouched (summary on stderr); points outside mainland China are left unchanged — the GCJ-02 offset does not apply there.
- The GCJ-02 shift formula is a state secret; this uses the public Krassovsky-ellipsoid approximation (same family as eviltransform / wandergis-coordtransform, identical to the server's `pkg/ewkt/coordtransform`), accurate to ~1–2 m — far below marker-level semantics. The official map APIs (Amap/Baidu convert) only convert **to** GCJ-02/BD-09, not from it.
- **Scope**: this rule is about **geolocation fields** and known GCJ-02/BD-09 sources, where the system is certain. Plain lng/lat number columns of unknown origin could be either system — ask the user where they came from, or submit as WGS84 and state that assumption; never convert blindly.
- `geocode` output is **already WGS84** — never convert it again.

## layer.json example (add_layer payload)

```json
{
  "mapId": 123,
  "layerType": "region",
  "columns": [
    {"id": "region", "name": "City", "type": "multiLineText"},
    {"id": "sales",  "name": "Sales", "type": "number"}
  ],
  "rows": [
    {"cells": {"region": "rg:cn.guangdong.shenzhen", "sales": 6573}},
    {"cells": {"region": "rg:cn.guangdong.guangzhou", "sales": 1274}},
    {"cells": {"region": "rg:cn.guangdong.dongguan", "sales": 211}}
  ],
  "fieldMappings": {"regionColumn": "region", "colorColumn": "sales"},
  "styleValues": {
    "aggregateMethod": "sum",
    "fillMethod": "quantile",
    "colors": ["#eff3ff","#c6dbef","#9ecae1","#6baed6","#4292c6","#2171b5","#084594"],
    "regionLevel": "admin2",
    "regionId": "rg:cn.guangdong"
  }
}
```

Notes on styleValues (see [layer-types.md](../references/layer-types.md) for the full list):
- `regionLevel` sets the overview level — admin2 here for a city map. It may be coarser than the UIDs: the same city-matched rows at admin1 render a province map that drills into cities on click.
- `regionId` focuses the layer on one province/city (e.g. `"rg:cn.guangdong"` for Guangdong) — the viewport auto-fits to that region. Omit for a whole-country view.
- `icons` (array aligned with `colors`) gives each color band its own marker on `icon` layers.

## items.json example (region_match payload)

```json
{
  "items": [
    {"admin1": "广东省", "country": "CN"},
    {"admin0": "China", "admin1": "Beijing"},
    {"longitude": 116.4, "latitude": 39.9}
  ]
}
```

## Output

Each command prints the API response JSON to **stdout** (for piping / parsing).
Errors go to **stderr** with a non-zero exit code.

## Share link

`create_map`, `add_layer`, `update_layer`, and `update_map` responses carry
`shareUrl` — the complete viewer link, built by the server from the
deployment's frontend origin. Present it as-is; never compose or hardcode
hostnames. The only credential needed is the access token.
