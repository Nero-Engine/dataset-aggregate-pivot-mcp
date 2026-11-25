# Dataset Aggregate & Pivot (Remote MCP Server)

**SQL GROUP BY and spreadsheet pivot tables for messy JSON rows, in a single tool call.** Hand it a list of rows from a scraper, an API or a spreadsheet, say what to group by and what to compute, and it hands back one clean summary row per group plus an exact account of anything it skipped.

Built for AI agents. No install, no API key, no signup. Connect by URL and call it.

```
https://dataset-aggregate-pivot.nerolabs.workers.dev/mcp
```

Free to use while in early access.

## What it does

One call runs the whole summary, in this order:

1. **Group** the rows by one or several fields (dot paths like `address.city` work), or leave the group fields empty to summarise every row into one. Add a **date bucket** to group a date or timestamp by day, ISO week, month, quarter or year (`orderedAt` becomes `orderedAt_month` = `2026-08`).
2. **Aggregate** each group with any of 11 functions: count, countDistinct, sum, avg, min, max, median, first, last, list and listDistinct, each with its own output column name.
3. **Pivot** one field's distinct values into columns: group by `region`, pivot on `product`, fill the cells with the sum of `amount`, and get one row per region with a column per product, zero-filled where a combination has no rows.
4. **Sort** by any output column, keep the **top N** groups, and add a **grand-total row** that still covers every input row.

Messy data is the normal case. `South`, `south ` and `SOUTH` land in one group with one label. `"$1,234.50"`, `"49 USD"`, `"1.234,50"` and `"(300)"` are read as numbers. Values that genuinely are not numbers, like `"n/a"`, are never guessed at: they are left out and counted in the summary, and a misspelled field name comes back as a warning instead of a silently empty result.

## Tools

| Tool | What it does |
|---|---|
| `list_capabilities` | Lists the 11 aggregation functions, the date bucket formats, the labels used for blank, invalid-date and total rows, and the limits per call. Processes no data. |
| `aggregate_rows` | Groups, aggregates, pivots, sorts and totals the rows you pass, and returns the summary rows plus a report of groups found, groups dropped by top N, skipped values and warnings. |

## Connect

**Claude Code**

```bash
claude mcp add --transport http dataset-aggregate-pivot https://dataset-aggregate-pivot.nerolabs.workers.dev/mcp
```

**Claude Desktop / claude.ai:** Settings, Connectors, Add custom connector, paste the URL above.

**Cursor, Windsurf, VS Code and other MCP clients**

```json
{
  "mcpServers": {
    "dataset-aggregate-pivot": {
      "url": "https://dataset-aggregate-pivot.nerolabs.workers.dev/mcp"
    }
  }
}
```

## Example

Eight messy order rows go in, with orders per region and each product's revenue pivoted into its own column:

```json
{
  "rows": [
    {"orderId": 1001, "region": "North", "product": "Widget", "amount": "$1,200.00", "orderedAt": "2026-07-03"},
    {"orderId": 1002, "region": "North", "product": "Gadget", "amount": 350, "orderedAt": "2026-07-18"},
    {"orderId": 1003, "region": "South", "product": "Widget", "amount": "890.50", "orderedAt": "2026-07-22"},
    {"orderId": 1004, "region": "south", "product": "Gizmo", "amount": 120, "orderedAt": "2026-08-02"},
    {"orderId": 1005, "region": "East", "product": "Widget", "amount": 2400, "orderedAt": "2026-08-05"},
    {"orderId": 1006, "region": "East", "product": "Gadget", "amount": "n/a", "orderedAt": "2026-08-09"},
    {"orderId": 1007, "region": "North", "product": "Gizmo", "amount": 75, "orderedAt": "2026-08-11"},
    {"orderId": 1008, "region": "", "product": "Widget", "amount": 410, "orderedAt": "2026-08-14"}
  ],
  "groupByFields": ["region"],
  "aggregations": [{"function": "count", "alias": "orders"}],
  "pivotField": "product",
  "pivotValueField": "amount",
  "pivotFunction": "sum"
}
```

Four summary rows come out. `South` and `south` became one group, `"$1,200.00"` summed as 1200, the row with no region is kept visibly as `(blank)`, and the `"n/a"` amount was skipped and reported rather than treated as a number:

```json
{
  "rows": [
    {"region": "(blank)", "orders": 1, "Gadget": 0, "Gizmo": 0, "Widget": 410},
    {"region": "East", "orders": 2, "Gadget": 0, "Gizmo": 0, "Widget": 2400},
    {"region": "North", "orders": 3, "Gadget": 350, "Gizmo": 75, "Widget": 1200},
    {"region": "South", "orders": 2, "Gadget": 0, "Gizmo": 120, "Widget": 890.5}
  ],
  "summary": {
    "inputRowCount": 8,
    "groupCount": 4,
    "outputRowCount": 4,
    "pivot": {"field": "product", "valueField": "amount", "function": "sum", "distinctValues": 3},
    "columns": ["region", "orders", "Gadget", "Gizmo", "Widget"],
    "skippedValues": {"pivot:Gadget": 1}
  }
}
```

Add `"sortBy": "orders", "sortDirection": "desc", "topN": 10, "includeTotalsRow": true` to the same call for a top 10 with a grand total, or `"dateBucketField": "orderedAt"` for one row per region per month.

## Skills

This server also publishes ready-made playbooks (Skills over MCP), so Claude and other agents know the steps for the whole job, not just each tool. Each is a `SKILL.md` at `skill://<name>/SKILL.md`, listed by `skills/list` and `resources/list`, read with `resources/read`.

- `summarise-dataset-pivot`: Summarise a list of JSON rows like a pivot table or SQL GROUP BY: totals, counts, averages and medians per group, per month or week, with a totals row and top N.

## Limits

- Up to **500 rows per call**. Anything larger returns a clear message rather than failing silently. Batches can be summarised separately for sum, count, min and max, and an average rebuilt as total sum divided by total count, but a median or a distinct count cannot be combined across batches.
- Up to **50 pivot columns** and **5,000 pivot cells** (groups multiplied by pivot columns) per call, so a pivot on a near-unique field such as an ID is refused with advice instead of returning hundreds of columns.
- Up to **20 aggregations** per call.
- Rows are passed inline as JSON.

## Privacy

Your rows are processed in memory and never stored. To see which tools get used, each call records the tool name, row counts, whether it succeeded, the client name your app reports, the country and a one-way hashed caller ID. Your data, your arguments and your IP address are never kept in that log.

## Also available

The same engine runs on the Apify Store as [Dataset Aggregate, Group By & Pivot](https://apify.com/nerolabs/dataset-aggregate-pivot), which also reads Apify datasets, CSV, TSV, Excel, JSON and JSON Lines files and Google Sheets by URL, handles up to 200,000 rows per run, exports the summary as a CSV or Excel file, appends it to a named dataset that accumulates across scheduled runs, and posts it to a webhook.

Built by **Nero Labs**.
