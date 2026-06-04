# Build / Dev Tool

A Cowork plugin that installs the Build / Dev Tool into your sidebar. Paste an Airtable ticket URL, the tool fetches the ticket (and its parent backlog ticket), reads any documents linked in the ticket, researches supporting context, then **builds the deliverable the ticket is asking for** and displays it as paste-ready code plus a manual implementation guide.

## What it does

1. **Fetch (child + parent)** — paste a link to any Airtable ticket in the RAM base (CRM Product Backlog or Sprint backlog - Pod). The tool extracts the record ID from the URL (classic `/rec…` links and interface URLs with `?detail=` both work). If the record is a sprint card, it also resolves and pulls the linked **parent CRM Product Backlog ticket** so the full brief is in scope.
2. **Read links** — scans the ticket and parent for URLs, classifies each, and reads the contents in parallel:
   - **Google Docs / Drive** — `read_file_content` (with a Glean fallback).
   - **Confluence** — `getConfluencePage` by page ID.
   - **Slack** — `slack_read_thread` from an archives link.
   - **Cordial** — `get_content_include` for content-include keys found in the URL.
   - **Braze** — `get_campaign_details` / `get_canvas_details` from an ID in the URL.
   - **Snowflake** — worksheet URLs can't be auto-read, so they are flagged for manual review.
   - **Other** — attempted via Glean `read_document`.
3. **Research** — runs in parallel before building:
   - **Glean** cross-source synthesis + **Confluence** CQL search on the ticket's title tokens.
   - **Cordial content includes** — follows every `content:` include referenced (in the brief, builder notes, or linked docs) and pulls its full source, so the build understands how the data is consumed downstream (housing template, sub-cards, which fields/flags they key off).
   - **Cordial supplements** — for every supplement (data table) referenced, fetches config (index fields), total row count, `lastDataUpdate` freshness (flags ⚠️ STALE if >14 days old), and a sample to expose the field list and any always-null fields.
   - **Snowflake schema + view DDL** — for every fully-qualified `db.schema.table`, pulls real columns and, when it's a VIEW, the full view definition (so the build can ground SQL and catch filter/label logic).
4. **Snowflake schema pre-fetch** — before building, the tool detects any fully-qualified `db.schema.table` named in the ticket or its linked docs and pulls the table's real columns from `INFORMATION_SCHEMA.COLUMNS`, then hands those to the build step as authoritative — so generated SQL uses real column names instead of guesses.
5. **Build** — feeds the ticket + link contents + research (including the live Snowflake schemas) to Claude with instructions to produce the actual deliverable (Cordial Smarty/HTML modules, Braze Liquid, Snowflake SQL, supplement specs, config), grounded in the inputs and cited to source. The report format **adapts to each ticket** — sections are added, dropped, reordered, or renamed as the build needs; only a meta header, the code blocks, and an assumptions/open-questions section (when relevant) are constant.
6. **Display only (Cordial/Braze)** — the tool **never** makes changes in the Cordial or Braze UI. It renders code blocks with per-block Copy buttons and a Copy-all button, alongside a manual implementation guide of the steps the developer performs themselves.
   - **Run in Snowflake** — SQL blocks get a "Run in Snowflake" button that executes the query via the Snowflake connector and renders the results as a table inline, right under the code. Before executing, it auto-checks the query's columns against the real table schema and **refuses to run** (showing the bad identifier + the real columns, with a "Run anyway" override) if it finds a column that doesn't exist. Read-only `SELECT`/`WITH` runs immediately; anything that could modify data/schema prompts for confirmation first.
   - **Verify columns** — a companion button looks up the real columns of every fully-qualified table in the query (via `INFORMATION_SCHEMA.COLUMNS`), lists them inline, and flags any identifier the query references that isn't a real column — catching invalid-identifier errors before you run.
7. **Publish to Confluence** — a button at the top of the report publishes it as a page in the CRMOPS space under the build folder (`parentId 118948921497`). Page title convention: `[airtable ticket name] - [ASSIGNEES] - [CURRENT DATE]` (assignees come only from the child sprint card's Assignees field — never the parent or Product Owner).

## Install

Drop the `.plugin` file into Cowork's plugin install (or place this folder in your Claude plugins directory). Then, in a new chat, say:

> install the build/dev tool

The tool will appear in your sidebar and persist across sessions.

## Required connectors

The artifact calls live MCP tools — connect each of these in Cowork before running the tool. Any source whose connector isn't connected is simply skipped and flagged rather than breaking the build.

| Category      | What it does                                                                  |
|---------------|-------------------------------------------------------------------------------|
| Airtable      | Pulls the source ticket + parent (RAM base, CRM Product Backlog + Sprint backlog) |
| Atlassian     | Reads linked Confluence pages, searches Confluence for context, and publishes the finished report as a page |
| Glean         | Cross-source research and fallback document reading                           |
| Google Drive  | Reads Google Docs / Sheets / Drive files linked in the ticket                 |
| Slack         | Reads Slack threads linked in the ticket                                      |
| Snowflake     | Available for running SQL (worksheet URLs are flagged, not auto-read)         |
| Cordial       | Reads content-includes / automation templates referenced in the ticket       |
| Braze         | Reads campaigns / canvases referenced in the ticket                           |

## Configuration (baked in)

These IDs are hard-coded for the CRMOPS Move Inc. instance:

- Airtable base: `appasWDNUZc8yQFnh` (RAM)
- Atlassian cloud: `cf0dc8c2-47a8-4929-8d48-2e03205ce9da`
- Confluence space: `117400502402` (CRMOPS)
- Published-page parent folder: `118948921497`

If you fork this for another team, edit the constants near the top of the `<script>` in `artifacts/build-dev-tool.html`.

## License

Internal use only. CRMOPS / Move Inc.
