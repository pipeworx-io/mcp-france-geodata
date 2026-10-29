# @pipeworx/france-geodata

Point and radius questions on French public geodata: communes and population, buildings around a point, land cover at a point, and flood-risk records for a place. Every call is a live per-request proxy; nothing is copied or stored.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1856+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `fr_commune(name | insee_code | lat+lon, limit?)` — commune register entry: INSEE code, postcodes, population, area, centre, department, region. "Population of Lyon" answers 69123, 519,127.
- `fr_buildings_near(lat, lon, radius_m?, limit?, include_geometry?)` — buildings within up to 500 m of a point, nearest first, each with GeoJSON polygon, height, floors, dwellings, use and nature. Reports `total_matched` and `truncated`; up to 200 candidates are ranked per call.
- `fr_land_cover(lat, lon)` — the CORINE Land Cover 2018 class (code and label) of the polygon containing the point. `found:false` outside coverage.
- `fr_flood_zones(lat, lon, radius_m?)` — flood hazards recorded for the commune at the point, flood-zone atlases and significant-flood-risk territories within the radius.

## Auth

Keyless.

## Data sources

- Etalab / INSEE, commune register — names, codes, population.
- IGN, BD TOPO — building footprints and attributes.
- Copernicus / IGN, CORINE Land Cover 2018 — land-cover classes (25 ha mapping unit, so it describes the surroundings of a point, not the spot itself).
- Géorisques (French Ministry of Ecological Transition), GASPAR — flood-risk records.

Related packs: `ban-fr` (addresses, reverse geocoding), `france-dvf` (property sales).

## Notes for the next person

- `fr_flood_zones` is a register listing, not a parcel-level flood polygon. There is no national flood-polygon layer on the IGN platform (only isolated local plans), and the hazard register's plan-of-prevention endpoint ignores location filters, so it is not used. Absence of a record is not proof of no flood risk, and the tool says so.
- The flood source resets connections from some residential and fleet-office networks but answers from Cloudflare's edge. Verify it from a Worker, not from a laptop.
- The building filter takes latitude first (OGC axis order for WGS84); the land-cover filter needs an explicit SRID with longitude first. Mixing them returns an empty set with HTTP 200, which looks like "no buildings here".
- The building source answers an OGC exception as XML with HTTP 200; that is surfaced as an error, not an empty result.
- Buildings: when more than 200 match, the first 200 returned by the source are ranked, so a closer building may be missed; narrow `radius_m`. The tool flags this.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "france-geodata": {
      "url": "https://gateway.pipeworx.io/france-geodata/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/france-geodata/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1856+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/fr_commune \
  -H 'Content-Type: application/json' \
  -d '{"name":"Lyon","limit":3}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/fr_commune`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-france-geodata` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/france-geodata/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "france-geodata": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-france-geodata"]
    }
  }
}
```

Without a key it calls anonymously, capped at 50 calls a day. A free account
raises that to 200 a day: get a key at https://pipeworx.io/signup and add
`"env": { "PIPEWORX_API_KEY": "<your key>" }` to the server entry.
The key is sent as a Bearer token, the same way a hosted connection sends it,
so calls are metered and billed exactly as they are there.

It lists **only** this pack's tools by default. Set `PIPEWORX_META_TOOLS=1` to
also list `ask_pipeworx` and the other gateway-wide tools.

Behind an HTTPS proxy: Node's built-in `fetch` does not read `HTTPS_PROXY` by
default. Newer Node releases honour it when `NODE_USE_ENV_PROXY=1` is set; check
your Node version's documentation.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about France Geodata data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
