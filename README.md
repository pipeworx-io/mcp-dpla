# @pipeworx/dpla

The Digital Public Library of America — ~50 million digitised items aggregated
from the Library of Congress, the National Archives, the Smithsonian,
HathiTrust, Internet Archive and hundreds of state libraries, universities and
historical societies, searchable in one query.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1683+ live data sources.

## Tools

- `dpla_search_items(...)` — search by free text, title, creator, contributing hub, subject, type, named collection or date range.
- `dpla_item(id)` — one record in full, including the raw source metadata as the contributing institution wrote it.
- `dpla_collections(q | title)` — which named collections inside contributing institutions hold matching material, with an item count each.

## Auth

Platform-backed since 2026-09-18: `PLATFORM_DPLA_KEY` lives in the encrypted
`platform_keys` table (no Cloudflare binding) and `platformKeyEnv` is declared
in `workers/gateway/src/pack-manifest.json`, so callers need nothing. Pass your
own key as `_apiKey` to use it instead.

The key is **free, instant and self-service** — no approval, no plan, no
payment:

```
curl -XPOST https://api.dp.la/v2/api_key/<your-email>
```

DPLA emails the key within a minute or two.

Verify the platform key is really set with
`curl -s https://gateway.pipeworx.io/manifest.json` → `dpla.platform_key_set`,
never from a file on a laptop.

## Data sources

- <https://api.dp.la/v2/items> — item search; `dpla_collections` is the
  `sourceResource.collection.title` facet of this same endpoint.
- <https://api.dp.la/v2/items/{id}> — one record.

Notes the next person would otherwise rediscover:

- **`/v2/collections` no longer exists** (fleet #2249, 2026-09-18). DPLA's API
  serves exactly three routes — `GET /v2/items`, `GET /v2/items/{id}`,
  `POST /v2/api_key/{email}` (their source: `github.com/dpla/dpla-api`,
  `src/index.ts`). `api.dp.la/v2/collections` answers
  `404 The requested resource could not be found` to a valid key, the same
  body it gives no key at all. An earlier version of this pack read that 404
  as "bad key" because it had only ever been run without a working one. Named
  collections now surface only as the `sourceResource.collection.title` facet
  on the items search (`facets=sourceResource.collection.title`, `facet_size`
  up to 2,000), which is what `dpla_collections` is built on.
- **A 404 from DPLA is never a key problem.** A missing key and a wrong key
  give the **identical 403** and the identical body
  (`{"error":"invalid_api_key","message":"Invalid or inactive API key."}`),
  checked **before** any other parameter is validated. So a 403 says nothing
  about whether the rest of the query was well formed, and a 404 says the
  path is gone, not that the key is. Verify a key by calling `/v2/items` with
  it, not by reading an error.
- `page_size=0` does not suppress docs — it falls through to DPLA's default
  page — so `dpla_collections` asks for one doc and ignores it.
- Filters are **dotted paths into the record**, not flat names:
  `sourceResource.title`, `sourceResource.creator`, `provider.name`,
  `sourceResource.subject.name`, `sourceResource.collection.title`,
  `sourceResource.date.after`.
- **Fields are a string in one contributor's records and an array in another's**
  — DPLA aggregates metadata written by hundreds of separate institutions and
  does not normalise cardinality. `list()`/`one()` in the pack handle both;
  anything reading `d.sourceResource.creator[0]` directly will break on some
  contributors. `sourceResource.collection` is an object with `title` (not
  `name`), so it has its own `collectionTitles()` reader.
- `isShownAt` is the object on the holding institution's own site; DPLA holds
  the description, the institution holds the object.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "dpla": {
      "url": "https://gateway.pipeworx.io/dpla/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/dpla/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1683+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/dpla_search_items \
  -H 'Content-Type: application/json' \
  -d '{"q":"Dust Bowl","limit":10}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/dpla_search_items`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "dpla": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-dpla"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-dpla
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Dpla data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
