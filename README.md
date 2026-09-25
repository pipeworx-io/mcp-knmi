# @pipeworx/knmi

Dutch national weather service open data — station observations, national radar
precipitation, HARMONIE-AROME forecast runs and the gridded daily climate series
— from the KNMI Data Platform Open Data API.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1679+ live data sources.

## Tools

- `knmi_datasets(search?, verify?, limit?)` — the datasets worth calling, what
  each contains, how often it updates, and (checked live) its newest file.
- `knmi_dataset_files(dataset, version, max_keys?, order_by?, sorting?, begin?, page_token?)` —
  files in a dataset, newest first by default.
- `knmi_file_url(dataset, version, filename)` — a temporary signed download URL
  for one file.

## Auth

Keyless in practice, keyed by design. Every endpoint needs an `Authorization`
header, but KNMI **publishes a shared anonymous key** on its docs page and this
pack ships it as the default, so callers need nothing. BYO override via
`_apiKey` for a registered key.

The anonymous key is shared worldwide (50 requests/minute, 3000/hour) and KNMI
rotates it on an expiry date printed on the docs page:

- A **429** is that shared bucket, not an outage. A registered key gets a
  dedicated 200 req/s.
- A **401/403** means the published key has rotated — read the "Anonymous key"
  section of <https://developer.dataplatform.knmi.nl/open-data-api> and update
  `ANON_KEY`. The tools refuse with wording that says so.

Register free at <https://developer.dataplatform.knmi.nl/>. A Pipeworx-held key
would be `PLATFORM_KNMI_KEY`; declare `platformKeyEnv` in the pack manifest in
the same change that lands the value, never before.

Whole-dataset downloads need a separate bulk key by email to opendata@knmi.nl.
KNMI's fair-use policy treats aggressive polling as abuse; use their
Notification Service for new-file push instead of tight loops.

## Data sources

- `GET /open-data/v1/datasets/{dataset}/versions/{version}/files` — file listing.
- `GET /open-data/v1/datasets/{dataset}/versions/{version}/files/{filename}/url` —
  signed download URL.
- Docs: <https://developer.dataplatform.knmi.nl/open-data-api>

Things worth knowing:

- **There is no `/datasets` listing endpoint** — it answers 404. `knmi_datasets`
  carries a curated registry instead and verifies each entry live, so a retired
  dataset reports as retired rather than vanishing.
- Auth is checked **before** routing, so an unknown path also answers 401. You
  cannot probe the route table without a key.
- Dataset names are opaque and versioned as strings (`"1.0"`, `"2"`, `"5"`).
- `Actuele10mindataKNMIstations/2` still answers but stopped receiving files;
  `10-minute-in-situ-meteorological-observations/1.0` replaced it. A stale
  newest-file timestamp is the only signal, hence `verify`.
- Files are scientific binaries (NetCDF, HDF5, GRIB in tar), not JSON. HARMONIE
  runs are 1–3 GB each.
- Responses may carry an `X-KNMI-Deprecation` header ahead of a dataset retiring.

## Verified

2026-09-17 — 11 datasets verified live; radar composite listing and a signed
download URL both returned real payloads on the published anonymous key.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "knmi": {
      "url": "https://gateway.pipeworx.io/knmi/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/knmi/mcp` returns the tools in the table
above **plus the shared Pipeworx meta-tools** — `ask_pipeworx`,
`discover_tools`, `search_within`, `remember`/`recall` and the rest of the
gateway-wide set. So the tool count you see is larger than this table: a
single-pack endpoint currently lists roughly 30 shared tools alongside the
pack's own. The connection's `initialize` response states its exact scope, and
is the authoritative answer for a given day.

This is deliberate, not multiplexing by accident. The meta-tools are what let a
scoped connection answer a question this pack does not cover — via
`ask_pipeworx`, which routes across the whole catalog — without you adding a
second MCP server. There is currently no way to mount a pack endpoint without
them; if the extra schemas cost you more context than the routing is worth,
connect to the full gateway once rather than to several pack endpoints.

Or connect to the full Pipeworx gateway to get every pack's tools listed
directly, instead of just this one's:

```json
{
  "mcpServers": {
    "pipeworx": {
      "url": "https://gateway.pipeworx.io/mcp"
    }
  }
}
```

Both URLs reach the same gateway and the same 1679+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/knmi_datasets \
  -H 'Content-Type: application/json' \
  -d '{"search":"radar"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/knmi_datasets`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "knmi": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-knmi"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-knmi
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Knmi data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
