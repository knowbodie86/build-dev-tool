---
name: install-build-dev-tool
description: >
  Installs the Build / Dev Tool interactive artifact into the user's Cowork sidebar
  as a persistent tool. Use this skill when someone says "install build dev tool",
  "install the build/dev tool", "set up the build tool", "add the build dev tool to
  my sidebar", "install build dev", or any similar phrasing. Run this once after
  installing the plugin — it will add a live tool to the sidebar that pastes an
  Airtable ticket URL, pulls the sprint card and its parent backlog ticket, reads
  any links in the ticket (Google Docs/Drive, Confluence, Slack, Snowflake, Cordial,
  Braze), researches context across Glean/Confluence/Cordial, and builds the
  requested deliverable as paste-ready code plus an implementation guide.
metadata:
  version: "1.0.0"
---

# Install Build / Dev Tool

Install the Build / Dev Tool artifact into the user's Cowork sidebar so they can run it from a persistent panel rather than pasting HTML into chat.

## Steps

1. Read the artifact HTML from `${CLAUDE_PLUGIN_ROOT}/artifacts/build-dev-tool.html` using the Read tool. Do not summarize or truncate it — pass it through verbatim.

2. Call `mcp__cowork__create_artifact` with:
   - `id`: `"build-dev-tool"`
   - `html_path`: the absolute path you just read from (`${CLAUDE_PLUGIN_ROOT}/artifacts/build-dev-tool.html`)
   - `description`: `"Build / Dev Tool for CRMOPS. Paste an Airtable ticket URL — fetches the sprint card AND its parent CRM Product Backlog ticket, reads any links in the ticket (Google Docs/Drive, Confluence, Slack, Snowflake, Cordial, Braze), researches supporting context (Glean/Confluence/Cordial), then builds the requested deliverable (Cordial Smarty/HTML modules, Braze Liquid, Snowflake SQL, configs) and displays paste-ready code plus a manual implementation guide. Does NOT make changes in the Cordial or Braze UI — display only."`
   - `mcp_tools`: the array of every MCP tool the artifact calls — Airtable `list_records_for_table`; Glean `chat` + `read_document`; Atlassian `getConfluencePage` + `searchConfluenceUsingCql`; Google Drive `read_file_content`; Slack `slack_read_thread`; Snowflake `sql-query`; Cordial `get_content_include` + `get_automation_template`; Braze `get_campaign_details` + `get_canvas_details`. Use the fully-qualified `mcp__<server>__<tool>` names as they appear in this Cowork session.

3. Confirm to the user that the tool has been installed and is now available in their sidebar. Mention the key connectors (Airtable, Atlassian, Glean, Google Drive, Slack, Snowflake, Cordial, Braze) and tell them to connect any that aren't already linked — any source whose connector isn't connected is simply skipped and flagged rather than breaking the build.

## Important

- If `mcp__cowork__create_artifact` is not available, let the user know that this feature requires Cowork mode and stop.
- If an artifact with the same id (`build-dev-tool`) already exists, Cowork will update it in place rather than erroring — this is safe to run again.
- Do not attempt to modify the HTML before registering it. The artifact is self-contained.

## What this tool does

The installed artifact is a self-contained build generator with the following capabilities:

- **Airtable URL fetcher (child + parent)** — parses record IDs out of classic `/rec…` URLs and from interface URLs where the `rowId` is nested in the base64-encoded `?detail=` query param. Fetches whichever record the URL points at, and when it's a Sprint backlog card, resolves and pulls the linked parent CRM Product Backlog ticket so the full brief is in scope.
- **Link reading** — scans the ticket (and parent) text for URLs, classifies each (Google Docs/Drive, Confluence, Slack, Snowflake, Cordial, Braze, or other), and reads the contents in parallel using the matching connector. Snowflake worksheet URLs can't be auto-read and are flagged for manual review.
- **Supplementary research** — runs Glean chat (cross-source synthesis), a Confluence CQL search on the ticket's distinctive title tokens, and Cordial content-include lookups for dashed identifiers found in the brief.
- **Build generation** — feeds the ticket, the link contents, and the research bundle to Claude with instructions to produce the actual deliverable: Cordial Smarty/HTML modules, Braze Liquid, Snowflake SQL, supplement specs, or config — grounded in the inputs and cited to source. The output format adapts to each ticket (sections are added, dropped, reordered, or renamed as needed); only a meta header, the code blocks, and an assumptions/open-questions section (when relevant) are constant.
- **Display only** — the tool never makes changes in the Cordial or Braze UI. It renders paste-ready code blocks with per-block Copy buttons and a Copy-all button, plus a manual implementation guide describing the steps the developer performs themselves.
- **Realtor.com brand styling** — full-bleed red header with the realtor.com SVG logo pill, Poppins + JetBrains Mono typography, brand-colored panels and code blocks.
