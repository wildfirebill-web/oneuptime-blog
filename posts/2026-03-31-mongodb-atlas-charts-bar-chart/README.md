# How to Create a Bar Chart in MongoDB Atlas Charts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Charts, Visualization, Bar Chart, Dashboard

Description: Step-by-step guide to creating a bar chart in MongoDB Atlas Charts to visualize categorical data and compare values across groups or time periods.

---

## What Is MongoDB Atlas Charts

MongoDB Atlas Charts is the built-in data visualization tool for MongoDB Atlas. It connects directly to your Atlas cluster and lets you build charts, dashboards, and reports without exporting data to a separate BI tool. Bar charts are one of the most commonly used chart types for comparing quantities across categories.

## Prerequisites

Before you start, you need:
- An active MongoDB Atlas account
- A cluster with data loaded (you can use the Sample Datasets from Atlas)
- At minimum the "Project Data Access Read Only" role

## Navigating to Atlas Charts

1. Log in to [cloud.mongodb.com](https://cloud.mongodb.com)
2. Select your project and cluster
3. Click **Charts** in the left sidebar
4. Click **Add Dashboard**, give it a name, and click **Create**
5. Inside the dashboard, click **Add Chart**

## Selecting the Bar Chart Type

In the chart builder:

1. Under **Chart Type**, click the bar chart icon (horizontal bars) or the column chart icon (vertical bars)
2. Select your **Data Source** - choose the Atlas cluster, database, and collection
3. For this example, use the `sample_supplies.sales` collection

## Configuring the Axes

Drag fields from the **Fields** panel to the encoding channels:

- **X Axis** - drag `items.name` (a string field) for categories
- **Y Axis** - drag `items.price` and set the aggregation to **Sum**
- **Series** (optional) - drag `storeLocation` to color bars by location

```
X Axis:   items.name      (Category - no aggregation)
Y Axis:   items.price     (Aggregation: Sum)
Series:   storeLocation   (optional - creates grouped/stacked bars)
```

## Switching Between Grouped and Stacked

When a Series field is present, toggle between **Grouped** and **Stacked** using the toolbar. Grouped bars place each series side by side; stacked bars combine them into a single bar showing totals.

## Filtering the Data

Click the **Filter** tab to add conditions before aggregation. For example, to show only sales from 2023:

```javascript
{ "saleDate": { "$gte": { "$date": "2023-01-01T00:00:00Z" }, "$lt": { "$date": "2024-01-01T00:00:00Z" } } }
```

Filters in Atlas Charts use standard MongoDB query syntax.

## Customizing Appearance

Under the **Customize** tab you can:
- Change the color palette
- Set axis labels and titles
- Enable or disable data labels on bars
- Set a chart title and description
- Adjust bar orientation (horizontal vs. vertical)

## Sorting Bars

By default, bars are sorted by the category label alphabetically. To sort by value (highest to lowest), click the Y Axis encoding channel and choose **Sort by Value - Descending**.

## Using a Custom Aggregation Pipeline

For more complex data transformations, switch to the **Aggregation** tab and write a pipeline:

```javascript
[
  { "$unwind": "$items" },
  {
    "$group": {
      "_id": "$items.name",
      "totalRevenue": { "$sum": { "$multiply": ["$items.price", "$items.quantity"] } }
    }
  },
  { "$sort": { "totalRevenue": -1 } },
  { "$limit": 10 }
]
```

This gives you the top 10 products by total revenue to power your bar chart.

## Saving and Adding to Dashboard

Click **Save and Close** to add the chart to your dashboard. You can then resize and reposition it by dragging.

## Summary

Creating a bar chart in MongoDB Atlas Charts involves selecting a data source, dragging category and numeric fields onto the X and Y axes, and optionally adding a Series field for grouped or stacked views. Built-in filter support and custom aggregation pipelines give you full control over what data is visualized. Atlas Charts handles all the aggregation server-side, so even large collections render quickly.
