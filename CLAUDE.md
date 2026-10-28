# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Power BI Project (PBIP, saved in Power BI Desktop "DevMode") that charts US stock indices (S&P 500, NASDAQ Composite, Dow Jones) using daily close data from FRED. There's no build, lint or test tooling. You open `Stocks Index.pbip` in Power BI Desktop, which loads the model and refreshes it from the web. Every artifact is a text file (TMDL for the model, PBIR JSON for the report), so changes are made by editing those files directly.

## Structure

- `Stocks Index.SemanticModel/definition/` holds the TMDL semantic model:
  - `model.tmdl` lists the tables (`ref table ...`) and the Power Query load order (`PBI_QueryOrder`). A new table needs entries in both places plus its own file in `tables/`.
  - `tables/index_*.tmdl` define one import table per index. Each partition is an M query that calls `Csv.Document(Web.Contents(<fredgraph.csv URL>))`, promotes headers and sets types. The FRED series is chosen by the `id=` parameter in the URL, and the date range by `cosd=`/`coed=`. These dates are hard-coded, so updating the data means editing the URL.
  - `tables/dim_Calendar.tmdl` is a calculated table, `CALENDARAUTO()`, so its range follows the earliest and latest dates across all tables. NASDAQ starts in 1971, so the calendar does too.
  - `relationships.tmdl` relates each `index_*.observation_date` to `dim_Calendar.Date` (many-to-one). The index tables never relate to each other directly; shared filtering goes through `dim_Calendar`.
- `Stocks Index.Report/definition/` holds the PBIR report. Pages and visuals live in folders with generated IDs (`pages/<id>/page.json`, `visuals/<id>/visual.json`), and `pages.json` sets the page order. There's currently one page, "Evolución" (1920×1080), with a line chart of `dim_Calendar.Date` × the index values.

## Conventions / gotchas

- Index tables use the pattern `index_<name>` with columns `observation_date` (date) plus the FRED series ID as the value column (`SP500`, `NASDAQCOM`, `DJIA`). Value columns are typed `Currency.Type` in M, have a `$` format string and use `summarizeBy: sum`.
- There are no DAX measures yet. Visuals aggregate the raw columns.
- When adding TMDL objects by hand, give each one a unique `lineageTag` (a GUID). Relationship names are also GUIDs.
- TMDL uses tab indentation, and the indentation is significant.
- The comment in `index_nasdaq.tmdl` gives a wrong source link (`PCU2122212122210`). The query itself actually fetches `NASDAQCOM`.
- `.pbi/localSettings.json` and `.pbi/cache.abf` are gitignored, local-only files. Don't edit them.
- The report page and UI text are in Spanish.

# Mandatory Project Instructions (Power BI & TMDL Safeguards)

## Golden Rule Against Raw Code Fallbacks
* **STRICTLY PROHIBITED:** Do not manually edit, modify, or rewrite Power BI metadata files (`.tmdl`, `.bim`, `.pbi`, or model folders) using generic text-editing tools (such as `write_to_file`, `edit_file`, or line replacements).
* Every single change regarding DAX measures, table relationships, or semantic model structures **MUST** be executed exclusively by invoking the tools provided by the `powerbi-modeling-mcp` server.

## Operational Protocol
1. When asked to create, update, or optimize a DAX measure, you must first utilize the MCP server tools to inspect the current state of the model.
2. Apply structural adjustments solely through the Model Context Protocol (MCP) server endpoints connected to Power BI.
3. If the action requested is unsupported by the MCP server or if the server goes offline, **STOP immediately** and alert the user. Do not attempt to fallback into editing the raw TMDL or BIM files as text.