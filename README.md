# Xome

US residential foreclosure/bank-owned (REO)/short-sale auction calendar and
property facts from [xome.com](https://www.xome.com).

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1679+ live data sources.

## Tools

| Tool | What it returns |
|------|-----------------|
| `xome_search` | Browse listings by state (+ optional county): property facts (beds/baths/sqft/year built/lot size/occupancy), sale type, `auction_status`/`bid_deadline`. Not the bid/sale price — see `xome_property_detail`. |
| `xome_property_detail` | Full detail for one listing: property facts, the exact sale event (date + courthouse venue address), sale terms, and the bid/sale **price** with an explicit status label. |

## The bid-price gap — resolved (outcome (a))

The dollar amount is **not** in the page's schema.org markup (`Offer.price`
is absent even on a confirmed-sold listing — verified: a "Sold to Bank"
listing's `Offer` shows only `availability: "https://schema.org/SoldOut"`,
no price). It lives instead in the same page's Next.js App Router RSC flight
payload (`self.__next_f.push([1, "..."])` chunks) as an embedded
`"auctionInfo":{...}` object carrying `bidAmount`, `bidType` and
`liveEventStatus`. This pack extracts it (regex + brace-matching on the
concatenated, unescaped flight chunks) rather than shipping calendar-only.

Every price is labeled by its actual state — never presented as a bare
number a caller could mistake for something it isn't:

| `liveEventStatus` | Meaning | `bidType` | Is it a real price? |
|---|---|---|---|
| `Marketing` | Auction hasn't started | `Est. Opening Bid` / `Starting Bid` | No — an estimate (often $0/unset) or a minimum, never a bid |
| `Post Auction` | Bidding closed, outcome undecided | `High Bid Received` | No — the top bid, not a completed sale |
| `Sold to Bank` / any `Sold*` | Sale complete | `Sale Price` | **Yes** — the real realized price. "Sold to Bank" means the foreclosing lender took the property back (a common, real outcome when no third-party bid clears the lender's credit bid) — still a completed sale with a real price. |

## Auth

None. Keyless, no login required to browse listings.

## Data sources

- `xome.com/sitemap/auctions/sitemap-auctionrecords0.xml.gz` — bulk discovery
  of individual listing URLs.
- `xome.com/auctions/{state}[/{county}]` — paginated listing pages; the
  `ItemList` `application/ld+json` block carries full property facts for
  every row with no extra request per item.
- `xome.com/auctions/{listing-slug}` — one listing's `RealEstateListing` /
  `SingleFamilyResidence`+`Product` / `Offer` / `SaleEvent` JSON-LD graph,
  plus the embedded `auctionInfo` RSC data for price.

### robots.txt

Disallows account/dashboard paths, `/api/`, `/signalr/` and a few report
endpoints — none of which this pack touches; everything here comes from the
plain public listing HTML already served to any visitor. Per Bruce's
2026-09-01 ruling, public unauthenticated content is buildable regardless of
robots; this pack still deliberately stays off every disallowed path since
the same data is already present on the page it fetches.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "xome": {
      "url": "https://gateway.pipeworx.io/xome/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/xome/mcp` returns the tools in the table
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
curl -X POST https://gateway.pipeworx.io/v1/tools/xome_search \
  -H 'Content-Type: application/json' \
  -d '{"state":"ca","county":"fresno"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/xome_search`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "xome": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-xome"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-xome
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Xome data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
