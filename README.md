# Michigan Compiled Laws and Michigan Administrative Code

Michigan Compiled Laws (statutes, `mi_compiled_law`) by citation, and — since fleet
#3957 — the Michigan Administrative Code (agency rules, `mi_admin_rule` and
`mi_admin_rule_search`). Keyless. `750.316` is first degree murder; `R 400.4142` is
the Child Caring Institutions rule on medication storage and dispensing.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1865+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

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

## Administrative Code (fleet #3957)

`mi_admin_rule({citation})` and `mi_admin_rule_search({query})` cover the Michigan
Administrative Code — agency rules, not statutes (demand: a question about medication
administration/storage in a licensed child caring institution routed to
`mi_compiled_law`, which is statutes-only, and returned `no_match`).

**Why a stored copy, not a live proxy (unlike WAC's shape for Washington):** the
official portal for the whole Code — the Administrative Rules System (ARS), run by
the Michigan Office of Administrative Hearings and Rules (MOAHR) — is an Angular
single-page app behind a Cloudflare rule that
answered a GEOFENCED 403 ("Accessing the site from a restricted location") to every
non-US vector tried on 2026-11-02 (a plain fetch, a `wrangler dev --remote` probe
Worker, and a real Chrome browser — all three reported `Country: JP`). It also has
no citation-addressable URL to call per request even from a US origin: a
client-rendered search form, not a REST surface. So this follows the Compiled Laws'
own shape: one JSON object per indexed department rule set in the shared statute
store (R2, `admincode/mi/ch-<ruleset-slug>.json` — same bucket as `statutes/mi/`, a
different prefix because a rule-set slug is a different kind of key than an MCL
chapter number).

**Source, per rule set.** Each department publishes its own rule set as a standalone
PDF booklet on its own department website — the same text MOAHR files with the Secretary of State,
distributed per-department rather than through the ARS search UI. The whole registry
of what this pack indexes is `RULE_SETS` in `scripts/bake-admin-rules.mjs`; today
that is one entry:

| Rule set | Publication | Citations |
|---|---|---|
| Licensing Rules for Child Caring Institutions | BCAL-Pub-452, eff. 6/8/15 | R 400.4101 – R 400.4666 (non-contiguous; rescinded numbers are real gaps) |

A citation outside every indexed rule set is reported `reason: "not_indexed"`, naming
what IS covered — never silently empty, and never claimed as "this rule does not
exist" for a rule this pack simply hasn't been pointed at yet. Adding a rule set is:
find its PDF booklet on the department's own website, add one entry to `RULE_SETS`, re-run the bake.

**No live-fetch fallback.** The shared PDF extractor (`extractPdfText`) was verified
2026-11-02 to drop word spacing and "fi"/"fl" ligatures on this specific
Adobe-InDesign-produced PDF (`"defined"` extracted as `"defned"`), which is not
acceptable for verbatim rule text — using it to answer a cold cache would risk
serving quietly wrong law. A citation with no stored copy available answers
`reason: "unavailable"` (a temporary condition) rather than a degraded live parse.

**`mi_admin_rule_search`** is a literal keyword search over the same stored rule
sets' headings and body text, ranked by how many distinct query words a rule matches
(a rule is never disqualified for missing one word — an English phrase like
"medication administration" often describes a rule using the regulatory synonym
"dispensing medication", which plain substring search cannot bridge, so a missing
word drops rank rather than hiding the rule entirely). Like WAC's own search, this
is literal vocabulary matching, not semantic — "Try fewer or different words; agency
rules use regulatory vocabulary" applies here too.

### Refresh (admin rules)

```
node mcps/michigan-code/scripts/bake-admin-rules.mjs              # every rule set -> mcps/michigan-code/.bake-admin/
node mcps/michigan-code/scripts/bake-admin-rules.mjs --only ccis   # one rule set
node mcps/michigan-code/scripts/bake-admin-rules.mjs --resume      # keep PDFs already downloaded
node mcps/michigan-code/scripts/upload-admin-rules.mjs             # publish to R2 (admincode/mi/), changed files only
```

PDF text extraction is `pdftotext -layout` (poppler) — a build-time-only dependency
of this bake script (`brew install poppler` / `apt install poppler-utils`), not the
Worker, same shape as `curl` in `bake-chapters.mjs`. Typos in the SOURCE document
("R Rule 554." instead of "Rule 554." at R 400.4554, "Room 613." instead of
"Rule 613." at R 400.4613) are preserved verbatim, not corrected. `.bake-admin/` is
gitignored; `upload-admin-rules.mjs` writes only changed files and uses
`wrangler r2 object put --remote` under the operator login, no API token.

### Data source (admin rules)

Michigan Office of Administrative Hearings and Rules, Michigan Administrative Code.
Michigan administrative rules are public record.

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

Both URLs reach the same gateway and the same 1865+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/mi_compiled_law \
  -H 'Content-Type: application/json' \
  -d '{"section":"750.316"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/mi_compiled_law`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-michigan-code` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/michigan-code/mcp` endpoint shown above. It needs network access;
there is no offline mode.

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
ask_pipeworx({ question: "your question about Michigan Code data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
