# How to Create a Table Chart in MongoDB Atlas Charts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Charts, Visualization, Table, Dashboard

Description: Create a table chart in MongoDB Atlas Charts to display raw or aggregated data in a structured grid with sorting, pagination, and conditional formatting.

---

## When to Use a Table Chart

Table charts are useful when exact values matter more than visual patterns. Use them for:
- Displaying the top N records (top customers, slowest queries, recent errors)
- Showing aggregated summaries per category (revenue by store, error count by service)
- Providing an exportable data grid within a dashboard

## Creating a Table Chart

1. Click **Add Chart** inside your dashboard
2. Select **Chart Type: Table**
3. Choose your data source

For this guide, use the `sample_supplies.sales` collection.

## Configuring Columns

Drag fields from the **Fields** panel to the **Columns** channel. Each field becomes one column in the table.

```text
Columns:
  - storeLocation     (Group by - Category)
  - purchaseMethod    (Group by - Category)
  - items.price       (Aggregation: Sum  - displays as "Total Revenue")
  - _id               (Aggregation: Count - displays as "Order Count")
```

When you add a Group By field, all non-grouped numeric fields are automatically aggregated per group.

## Renaming Column Headers

Click any column in the Columns list and enter a custom label. Rename `_id (Count)` to `Orders` and `items.price (Sum)` to `Total Revenue` for a cleaner table.

## Sorting Columns

Click a column header in the preview to sort the table by that column. Click again to reverse the sort order. The default sort is preserved when the chart is saved.

## Pagination

Atlas Charts table charts automatically paginate large result sets. Use the **Rows per Page** setting in the Customize tab to control how many rows appear per page (default is 10, maximum is 1000).

## Conditional Formatting

In the **Customize** tab, enable **Conditional Formatting** for numeric columns to highlight cells based on value thresholds:

- Revenue below 1000 - red background
- Revenue between 1000 and 5000 - yellow
- Revenue above 5000 - green

This makes it easy to spot underperforming and overperforming rows at a glance.

## Displaying Raw Documents

To show individual documents rather than aggregated rows, use a custom aggregation pipeline with no `$group` stage:

```javascript
[
  { "$match": { "status": "failed" } },
  { "$sort": { "createdAt": -1 } },
  { "$limit": 50 },
  {
    "$project": {
      "orderId": 1,
      "customer": 1,
      "amount": 1,
      "errorMessage": 1,
      "createdAt": 1
    }
  }
]
```

This returns the 50 most recent failed orders as individual rows in the table - useful for an operational error-tracking dashboard.

## Linking Out from Table Rows

Atlas Charts does not natively support hyperlinks in cells, but you can encode a URL in a string field in your collection and display it as a column. Use a computed field in your pipeline:

```javascript
[
  {
    "$addFields": {
      "detailUrl": {
        "$concat": ["https://app.example.com/orders/", "$orderId"]
      }
    }
  }
]
```

Dashboard users can copy and open the URL from the cell.

## Exporting Table Data

Viewers can export the table to CSV by clicking the three-dot menu on the chart. This is particularly useful when the table serves as a data export point within a larger analytical dashboard.

## Combining with Dashboard Filters

Table charts respect all dashboard-level filters. A date picker filter on the dashboard will automatically narrow the rows in the table to the selected time range without any additional configuration.

## Summary

Table charts in MongoDB Atlas Charts display data in a structured grid with configurable columns, custom labels, and auto-pagination. Group By fields aggregate rows, while custom pipelines let you show raw documents for operational use cases. Conditional formatting highlights important values, and CSV export gives dashboard users a convenient way to take data offline for further analysis.
