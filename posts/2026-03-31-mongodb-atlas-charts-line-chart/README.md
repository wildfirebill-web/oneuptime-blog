# How to Create a Line Chart in MongoDB Atlas Charts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Charts, Visualization, Line Chart, Time Series

Description: Learn how to build a line chart in MongoDB Atlas Charts to track trends and time-series data directly from your MongoDB Atlas collection.

---

## When to Use a Line Chart

Line charts are ideal for visualizing data that changes over time - think daily active users, revenue trends, error rates, or sensor readings. MongoDB Atlas Charts makes it straightforward to build time-series line charts from your collections without leaving the Atlas console.

## Setting Up Your Data Source

For this guide, use the `sample_mflix.movies` collection available in the Atlas sample datasets, or your own time-series collection.

1. Open Atlas Charts and create a new chart
2. Set the **Chart Type** to **Line**
3. Select your cluster, database, and collection as the data source

## Configuring a Time-Series Line Chart

Drag fields to the encoding channels:

- **X Axis** - drag a `Date` field (e.g., `released`). Set the **Date Granularity** to `Year`, `Month`, `Week`, or `Day` depending on your time range.
- **Y Axis** - drag a numeric field or use `Count` to count documents per time period.

```text
X Axis:  released    (Date field, Granularity: Year)
Y Axis:  _id         (Aggregation: Count)
```

This produces a line chart showing the number of movies released per year.

## Adding Multiple Lines (Series)

To compare multiple categories on the same line chart, add a field to the **Series** channel:

```text
X Axis:   released      (Granularity: Month)
Y Axis:   imdb.rating   (Aggregation: Average)
Series:   genres        (creates one line per genre)
```

Each unique value in the Series field becomes its own colored line.

## Handling Sparse or Missing Time Points

If your data has gaps, Atlas Charts will show breaks in the line by default. To connect points across gaps, enable **Connect Null Data Points** in the Customize tab. This draws a straight line segment between the last known value and the next available data point.

## Smoothing Lines

For noisy metrics, enable **Smooth Line** in the Customize tab to apply a cubic spline interpolation. This is useful for visualizing trends in metrics like CPU usage or response times where individual point values vary significantly.

## Custom Aggregation Pipeline Example

To show a rolling 7-day average of orders, use a custom pipeline:

```javascript
[
  {
    "$group": {
      "_id": {
        "$dateTrunc": { "date": "$createdAt", "unit": "day" }
      },
      "dailyOrders": { "$sum": 1 }
    }
  },
  { "$sort": { "_id": 1 } },
  {
    "$setWindowFields": {
      "sortBy": { "_id": 1 },
      "output": {
        "rollingAvg": {
          "$avg": "$dailyOrders",
          "window": { "documents": [-6, 0] }
        }
      }
    }
  }
]
```

Map `_id` to X and `rollingAvg` to Y for a smoothed trend line.

## Zooming and Time Range Selection

After saving the chart to a dashboard, readers can click and drag on the X axis to zoom into a specific time range. This interactivity is built in to Atlas Charts line charts with no extra configuration.

## Formatting Axis Labels

Under the **Customize** tab:
- Set a **Y Axis Label** (e.g., "Orders per Day")
- Enable **Prefix/Suffix** to show currency symbols or units
- Set **Number Format** to `Compact` for large numbers (e.g., `12.3K`)

## Summary

Line charts in MongoDB Atlas Charts are perfect for time-series and trend visualization. Configure the X axis with a Date field and a granularity, choose an aggregation for the Y axis, and optionally add a Series field for multi-line comparisons. Features like null point connection, smooth curves, and zoom-on-click make Atlas Charts line charts suitable for both operational dashboards and business reporting.
