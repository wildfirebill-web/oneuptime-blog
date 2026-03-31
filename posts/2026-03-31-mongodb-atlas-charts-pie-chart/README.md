# How to Create a Pie Chart in MongoDB Atlas Charts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Charts, Visualization, Pie Chart, Dashboard

Description: Learn how to create a pie chart in MongoDB Atlas Charts to display proportional distributions across categories in your MongoDB data.

---

## When to Use a Pie Chart

Pie charts work best when you want to show how a whole is divided into parts - for example, revenue by product category, user distribution by region, or order status breakdown. Atlas Charts supports both standard pie charts and donut charts.

## Creating a Pie Chart

1. Inside your Atlas Charts dashboard, click **Add Chart**
2. Choose **Chart Type: Circular - Pie**
3. Select your data source (cluster, database, collection)

For this tutorial, use the `sample_supplies.sales` collection.

## Configuring the Label and Value

Drag fields onto the two primary encoding channels:

- **Label** - the categorical field that defines each slice (e.g., `storeLocation`)
- **Arc** - the numeric field that determines slice size (e.g., `items.price` with `Sum` aggregation)

```text
Label:  storeLocation    (Category)
Arc:    items.price      (Aggregation: Sum)
```

Each unique `storeLocation` value becomes a slice, sized proportionally by total sales.

## Switching to a Donut Chart

In the **Customize** tab, set the **Inner Radius** percentage to any value above 0 to convert the pie to a donut. A donut chart leaves the center empty, which is useful for placing a KPI metric or total in the center via a dashboard text widget.

## Limiting the Number of Slices

If your category field has many unique values, the chart becomes unreadable. Limit slices in two ways:

**Option 1 - Use the Top N filter:**
In the Label encoding channel, enable **Limit Results** and set it to `10`. All remaining categories are grouped into an "Other" slice.

**Option 2 - Use a custom pipeline:**

```javascript
[
  { "$unwind": "$items" },
  {
    "$group": {
      "_id": "$items.name",
      "totalRevenue": { "$sum": "$items.price" }
    }
  },
  { "$sort": { "totalRevenue": -1 } },
  { "$limit": 8 }
]
```

This approach gives you exact control over which slices appear.

## Adding Percentage Labels

In the **Customize** tab, enable **Show Label as Percentage** to display the percentage share on each slice. You can also choose to show the raw value, the category name, or both.

## Filtering Data Before Aggregation

Use the **Filter** tab to restrict which documents are counted. For example, show only completed orders:

```javascript
{ "status": "completed" }
```

Filters are applied before the aggregation that sizes slices, so they directly affect proportions.

## Color Configuration

Atlas Charts assigns colors automatically. To customize:
1. Click the **Customize** tab
2. Expand **Color**
3. Click each category label to assign a specific hex color

This is useful for brand colors or when aligning with a corporate color scheme.

## Combining with Dashboard Filters

When your pie chart is on a dashboard with a shared filter, it responds to filter changes automatically. For example, a date range filter on the dashboard narrows the data feeding all charts, including the pie. See the Atlas Charts Filters and Drill-Down guide for how to set this up.

## Pie Chart Limitations

- Avoid pie charts when there are more than 7-8 categories - bar charts are more readable at that point.
- Pie charts do not work well for comparing small differences between slices. Use a bar chart if precise comparisons matter.
- Atlas Charts pie charts do not support nested/hierarchical breakdowns. Use a treemap for that.

## Summary

Creating a pie chart in MongoDB Atlas Charts requires mapping a categorical field to the Label channel and a numeric field with an aggregation to the Arc channel. Donut conversion, percentage labels, and Top-N slice limiting are available in the Customize tab. For production dashboards, limit slices to fewer than 8 and combine pie charts with dashboard-level filters to keep the visualization meaningful and readable.
