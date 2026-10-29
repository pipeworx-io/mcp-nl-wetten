# @pipeworx/nl-wetten

Dutch national legislation, consolidated and point-in-time: search by title,
look up a regulation's structure by its BWB id, and read one article's
original Dutch text as it stood on any date.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1856+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `nl_wetten_search(query, as_of?, limit?)` — title search over Dutch
  national legislation; returns the regulation(s) matching, each resolved to
  the version in force on `as_of` (default today), with its BWB id
  (`BWBR...`). Title search only, not full-text search.
- `nl_wetten_get_law(bwb_id, as_of?, offset?, limit?)` — one regulation's
  citeertitel, authority, legal domain(s), the resolved version's own
  validity period, its chapter structure, and a paged list of article
  numbers (never truncated silently — `has_more`/`next_offset`).
- `nl_wetten_get_article(bwb_id, article, as_of?)` — one article's original
  Dutch text, broken into its numbered "lid" (paragraph) subdivisions, plus
  the date that exact text took effect and the resolved version's validity
  period. Pass an earlier `as_of` to read a past version of the same article.

Every date returned is a separate field: `as_of` (what you asked for),
`toestand_valid_from` / `toestand_valid_until` (how long the resolved
consolidated version has applied), and — on `nl_wetten_get_article` —
`article_in_force_since` (when that article's own text last changed), which
can differ from the toestand's own start date when the article itself hasn't
been amended since an earlier date.

## Auth

Keyless.

## Data sources

- KOOP (the Dutch government's publications agency), Basiswettenbestand (BWB)
  — the consolidated register of Dutch national legislation, covering every
  statute, AMvB, ministerial regulation and treaty KOOP publishes.

## Notes

- **The upstream 406s without an explicit `Accept: application/xml` header.**
  No Accept header at all (what a bare `fetch()` sends) and
  `Accept: application/json` both get HTTP 406, even for an otherwise
  well-formed, healthy query. Every request this pack makes sets the header.
- **A title search without a validity-date filter returns one record per
  historical version, not per law** — "Algemene wet bestuursrecht" alone
  comes back 40+ times across its amendment history. Every search here adds
  the as-of date so each regulation resolves to exactly one version.
- **Article full text has no JSON form.** The consolidated text is only
  published as XML, and a single law's document can run several MB (the
  Algemene wet bestuursrecht's current version is ~2.6 MB). There's no XML
  parser in the Workers runtime, so article text is extracted with targeted
  pattern matching over the register's fixed element structure — correct for
  real statute text, not a general-purpose XML parser.
- Article numbers follow the source's own numbering, which is not always
  plain integers — e.g. "1:3" (chapter:article) or "3:9a" (a lettered
  insertion). Use the exact string `nl_wetten_get_law` lists.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "nl-wetten": {
      "url": "https://gateway.pipeworx.io/nl-wetten/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/nl-wetten/mcp` returns the tools in the table
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
curl -X POST https://gateway.pipeworx.io/v1/tools/nl_wetten_search \
  -H 'Content-Type: application/json' \
  -d '{"query":"Algemene wet bestuursrecht"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/nl_wetten_search`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-nl-wetten` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/nl-wetten/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "nl-wetten": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-nl-wetten"]
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
ask_pipeworx({ question: "your question about Nl Wetten data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
