# @pipeworx/bodacc

French official business-events bulletin (BODACC — Bulletin officiel des
annonces civiles et commerciales): dated notices per company — creations,
insolvency proceedings (redressement/liquidation judiciaire), business
sales/transfers, deregistrations, modifications, and annual account filings.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1683+ live data sources.

## Tools

- `bodacc_company_events(siren | name, event_type?)` — full BODACC event
  history for one company, newest first. `siren` is the same 9-digit id used
  by the `entreprises-fr` and `insee` packs — chain a SIREN looked up there
  straight into this tool (or the reverse).
- `bodacc_search(q?, name?, event_type?, departement?, since?, until?, siren?)`
  — open-ended search across all BODACC notices. `q` is whole-notice full
  text (name, activity, judgment/act text, address, ...); `name` is the
  separate, field-scoped form for matching the company name only. Also
  filters by event family, département, date range, and/or SIREN.
- `bodacc_insolvency_feed(since, until?, departement?)` — insolvency-only
  (`familleavis=collective`) notices published since a date, newest first.
  Built for polling/monitoring with an advancing `since`.
- `bodacc_get_notice(id)` — a single notice by its BODACC id.

## Auth

Keyless. The upstream (DILA's Opendatasoft portal) requires no registration
or API key for read access to this dataset.

## Data sources

- <https://bodacc-datadila.opendatasoft.com/api/explore/v2.1/catalog/datasets/annonces-commerciales/records>
  — the live BODACC records API (Opendatasoft Explore v2.1). ~50M+ rows,
  updated as new bulletins publish (daily, business days).

Traps:

- **`offset + limit` is capped at 10,000** for this dataset (confirmed live:
  `offset=9999&limit=2` → 400 `InvalidRESTParameterError`). `total_count` in
  the API response is NOT capped — only how far pagination can reach into it
  is. Every tool here reports `total_count`, `count` (rows actually
  returned), and a `truncated` flag (`offset + limit >= 10000` while
  `total_count` exceeds it) so a caller never reads the cap as the true
  count. Narrow with `event_type` / `departement` / a date range to stay
  under the cap on a broad query.
- **`registre` is an array**, not a scalar — `["805243102", "805 243 102"]`
  (digits-only and space-grouped forms of the same SIREN). ODSQL
  `registre="805243102"` matches against the array directly; this pack always
  strips spaces from an input SIREN before building that filter, and reads
  `registre[0]` (digits-only) back out when shaping a record.
- **Most upstream fields are per-family, not universal.** `jugement` is
  populated for insolvency/conciliation notices, `acte` for
  creations/sales, `radiationaurcs` for deregistrations,
  `modificationsgenerales` for modifications, `depot` for account filings —
  the others are `null` on a given row. Each is itself a JSON-encoded
  *string*, not a nested object; this pack parses whichever of them are
  non-null into a single `detail` object per notice rather than exposing raw
  upstream field names.
- **`familleavis` codes** (pass to `event_type`): `creation`,
  `immatriculation`, `collective` (insolvency), `conciliation`,
  `retablissement_professionnel`, `modification`, `radiation`, `dpc`
  (account filings), `vente` (sale/transfer), `divers`. Source of truth:
  the `ac_mapping_famille_avis` dataset on the same Opendatasoft domain.
  `event_type` also accepts English aliases (`sale`/`transfer`→`vente`,
  `insolvency`/`bankruptcy`→`collective`, `deregistration`/`deletion`→
  `radiation`, `accounts`→`dpc`, `registration`→`immatriculation`) — an
  unrecognized value is a thrown error naming the valid codes, never a
  silent `total_count: 0` (fixed 2026-09-23: `event_type: "sale"` used to
  build `familleavis="sale"`, an upstream field value that matches nothing).
- **ODSQL's `search()` is field-scoped unless you omit the field.**
  `search(commercant, "...")` matches the `commercant` field only (company
  name as filed); the unscoped `search("...")` matches every indexed field
  (activity description, judgment/act text, address, etc). `bodacc_search`'s
  `q` uses the unscoped form on purpose — the field-scoped form silently
  returned `total_count: 0` for `q: "fonds de commerce"` (a term that never
  appears in a company name) while the unscoped form returns 3,188 hits for
  the same query, confirmed live 2026-09-23. Use `name` instead of `q` to get
  the field-scoped, name-only match deliberately.
- Every call here hits the upstream endpoint directly, per request: the API is
  keyless, has no meaningful rate limit for normal query volumes, and a
  `since`-bounded monitoring query naturally stays well under the 10k
  pagination cap.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "bodacc": {
      "url": "https://gateway.pipeworx.io/bodacc/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/bodacc/mcp` returns the tools in the table
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
curl -X POST https://gateway.pipeworx.io/v1/tools/bodacc_company_events \
  -H 'Content-Type: application/json' \
  -d '{"siren":"805243102"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/bodacc_company_events`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "bodacc": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-bodacc"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-bodacc
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Bodacc data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
