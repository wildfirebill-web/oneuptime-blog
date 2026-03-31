# How to Use Filters and Drill-Down in MongoDB Atlas Charts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Charts, Visualization, Filter, Dashboard

Description: Learn how to add interactive filters and drill-down controls to MongoDB Atlas Charts dashboards so users can slice and explore data on their own.

---

## Why Interactive Filtering Matters

Static charts tell a story once. Interactive filters let every dashboard viewer answer their own questions without needing a data analyst to create a new chart. Atlas Charts supports both chart-level filters (fixed conditions baked into the chart) and dashboard-level filters (interactive controls that affect multiple charts simultaneously).

## Chart-Level Filters

Chart-level filters are applied before the aggregation and are invisible to viewers. They restrict a chart to a fixed subset of data.

In the chart builder, click the **Filter** tab and enter a standard MongoDB query:

```javascript
{ "status": "completed", "region": "us-east" }
```

You can also use expressions:

```javascript
{
  "createdAt": {
    "$gte": { "$date": "2024-01-01T00:00:00Z" },
    "$lt":  { "$date": "2025-01-01T00:00:00Z" }
  },
  "amount": { "$gt": 100 }
}
```

Chart-level filters are useful for charts that should always show a specific subset, such as "Q1 Sales Only" or "Critical Errors Only."

## Dashboard-Level Filters

Dashboard filters are interactive controls that viewers can change. They affect all charts on the dashboard that share the same data source.

### Adding a Filter

1. Open your dashboard in edit mode
2. Click **Add Filter** in the top bar
3. Choose a field (e.g., `storeLocation`) and filter type

Filter types include:
- **Category filter**: dropdown or multi-select for string fields
- **Date filter**: date range picker for date fields
- **Number filter**: range slider for numeric fields

### Connecting Filters to Charts

By default, a new dashboard filter applies to all compatible charts. To restrict which charts a filter affects:
1. Click the filter control
2. Under **Applied Charts**, uncheck charts that should not respond to this filter

This is useful when you have charts from different collections on the same dashboard.

## Drill-Down Charts

Drill-down lets viewers click a chart element to filter the dashboard by that value. For example, clicking a bar for "New York" in a bar chart filters all other charts to show only New York data.

### Enabling Drill-Down

1. Open the chart you want to act as a source
2. In the chart builder, go to **Customize - Interactions**
3. Enable **Cross-Filter** (also called "Filter on Click")

When a viewer clicks a bar, slice, or point, Atlas Charts adds a temporary dashboard filter for that value. Clicking again removes the filter.

## Date Drill-Down (Time Hierarchies)

Line charts and bar charts with date X axes support time hierarchy drill-down:

- Click a Year bar to drill into Months
- Click a Month bar to drill into Days

Enable this in Customize by selecting **Enable Drill-Down on Time Axis**. This is particularly powerful for revenue trend charts where you want to see annual totals but can click through to monthly or daily breakdowns.

## Using the Atlas Charts Search Bar

When a dashboard has a Category filter, viewers can type in the search box to quickly find specific values in a long dropdown list. This is automatically enabled for all Category filter types.

## Combining Filters for Complex Analysis

A well-designed dashboard typically has:
1. A **date range filter** at the top to set the analysis period
2. One or two **category filters** (region, product type, status)
3. Cross-filter enabled on primary charts so clicking a segment drills into the others

For example, in an e-commerce dashboard:
- Date range filter: last 30 days
- Region filter: multi-select dropdown
- Click a product category bar to filter the customer table and revenue line chart to that category

## Resetting Filters

Dashboard viewers can click **Reset Filters** or click the active filter value again to clear it. All charts immediately return to their unfiltered state.

## Sharing Filtered Dashboard State

When a viewer sets dashboard filters, the URL updates with the filter state encoded as query parameters. Copying and sharing this URL sends the recipient directly to the same filtered view.

## Summary

Atlas Charts supports chart-level filters for static data restrictions and dashboard-level interactive filters (date, category, number) that viewers can change in real time. Cross-filter drill-down lets viewers click chart elements to filter the rest of the dashboard, and time hierarchy drill-down enables year-to-month-to-day exploration on date charts. Combining these features turns a static report into a self-service analytics tool.
