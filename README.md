# Michigan Compiled Laws

Michigan Compiled Laws by citation. Keyless. `750.316` is first degree murder.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1882+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

See `docs/state-law-probe.md` for how this source was verified: every state pack
confirms a MISS is distinguishable from a HIT before shipping, because several
state sites answer HTTP 200 for statutes that do not exist.

## What `text` contains (fleet #3598)

`text` is the statute: the section's catchline and its body, as the Michigan Legislature
publishes the Compiled Laws. It does **not** include the History line (the public acts that
enacted or amended the section, with their effective dates), editor's notes or commentary.
Those used to arrive in `text` when every call was a live read, so a caller that parsed the
History line now gets less than it did; nothing was lost from the law itself. Every answer
carries a `text_scope` field saying which it is: the stored copy (`served_from: "stored-copy"`)
is statute text only, while the live fallback returns the page text from the section number
onward and can still include History and editor's notes. A repealed section is stored with its
repeal note as its text.

## Stored copy and `data_as_of` (fleet #3546)

The Legislature's site answers US data centres but refuses or stalls from Singapore: over 7 days external callers saw 3 successes against 98 "down" and 23 "forbidden" from there. A cache cannot help a first touch, so the Michigan Compiled Laws is stored, one JSON
object per MCL chapter number (the part of a citation before the dot), in the shared statute store (R2, `statutes/mi/`) that the Illinois,
Indiana, North Carolina, Wisconsin and Wyoming packs also use. The gateway hands the
pack its R2 binding and the pack reads only the object a call names (a very large
chapter is a small head plus parts, and a call reads the head and one part). A section
found there is answered with no upstream request; the response says
`served_from: "stored-copy"` and carries `data_as_of` (when that chapter was last read
from the source). A section the copy cannot answer (chapter missing, object unreadable,
or listed with no stored text) falls back to the live read. Absence is never remembered:
a chapter published later is served as soon as it exists.

Nothing is loaded at module scope: the data never enters the gateway bundle.

## Refresh

```
node mcps/michigan-code/scripts/bake-chapters.mjs            # all chapters -> mcps/michigan-code/.bake/
node mcps/michigan-code/scripts/bake-chapters.mjs --only X   # one chapter
node mcps/michigan-code/scripts/bake-chapters.mjs --resume   # keep source pages already fetched
node mcps/michigan-code/scripts/upload-chapters.mjs          # publish to R2 (statutes/mi/), changed files only
```

The source is the Legislature's published text of the Compiled Laws, one file per chapter, read under its own policy for automated collection (one request per second at most; this refresh makes one request per chapter file, two seconds apart). Repealed sections are stored with their repeal note. Sequential, with a delay between requests, an honest user agent, bounded
retries, and a stop on the first 429/403/451. Exit 0 only if every chapter was fetched
and parsed and the totals clear the floors (default 200 chapters, 15,000 sections with text); anything less exits 1 with the
counts. A chapter whose text is unchanged keeps its previous `data_as_of`. The baked
files are a working copy, not committed (`.bake/` is gitignored); `upload-chapters.mjs`
writes only the files whose content hash differs from the one recorded in
`statutes/mi/_manifest.json`, so a refresh that finds nothing new writes nothing. It
uses `wrangler r2 object put --remote` under the operator login; no API token.

## Data source

Michigan Legislature, Michigan Compiled Laws. Michigan statutes are public record.

## Freshness of the live fallback

On the live path a section is read once a day at most. When the legislature site cannot be reached, an earlier copy of the same section answers and the response says so (`served_from: "stale-cache"`, `stale: true`, `data_as_of`). A section with no earlier copy fails with an error that names a firewall block as one.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "michigan-code": {
      "url": "https://gateway.pipeworx.io/michigan-code/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/michigan-code/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1882+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/mi_compiled_law \
  -H 'Content-Type: application/json' \
  -d '{"section":"750.316"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/mi_compiled_law`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "michigan-code": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-michigan-code"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-michigan-code
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Michigan Code data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
