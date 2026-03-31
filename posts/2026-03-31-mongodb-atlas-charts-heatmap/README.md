# How to Create a Heatmap in MongoDB Atlas Charts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Charts, Visualization, Heatmap, Dashboard

Description: Build a heatmap in MongoDB Atlas Charts to visualize data density and intensity across two categorical or time-based dimensions.

---

## When to Use a Heatmap

Heatmaps visualize a matrix of values where color intensity represents magnitude. Common use cases include:
- Activity by hour and day of the week (support ticket volumes, API call patterns)
- Error rates by service and region
- Product sales by category and month
- User engagement by cohort and week

## Creating a Heatmap in Atlas Charts

1. Click **Add Chart** inside your Atlas dashboard
2. Select **Chart Type: Heatmap**
3. Choose your data source (cluster, database, collection)

For this guide, use the `sample_mflix.movies` collection or a custom events collection.

## Configuring the Axes

A heatmap requires two categorical (or date binned) dimensions and one numeric measure:

```
X Axis:  released     (Date field, Granularity: Month)
Y Axis:  genres       (Category)
Intensity: _id        (Aggregation: Count)
```

This produces a calendar-style grid where each cell's color represents the number of movies in that genre released in that month.

## Activity Heatmap by Hour and Day

A common operational use case is visualizing when events happen:

```javascript
// Sample document in api_requests collection
{
  _id: ObjectId("..."),
  endpoint: "/api/orders",
  statusCode: 200,
  durationMs: 45,
  timestamp: ISODate("2024-03-15T14:23:00Z")
}
```

Use a custom aggregation pipeline to extract hour and day:

```javascript
[
  {
    "$addFields": {
      "hour": { "$hour": "$timestamp" },
      "dayOfWeek": { "$dayOfWeek": "$timestamp" }
    }
  },
  {
    "$group": {
      "_id": { "hour": "$hour", "dayOfWeek": "$dayOfWeek" },
      "requestCount": { "$sum": 1 },
      "avgDuration": { "$avg": "$durationMs" }
    }
  }
]
```

Then map:
```
X Axis:     _id.hour         (Numeric - 0 to 23)
Y Axis:     _id.dayOfWeek    (Numeric - 1 to 7)
Intensity:  requestCount     (Sum)
```

## Color Schemes

In the **Customize** tab, choose a color palette for the intensity gradient:
- **Sequential** (e.g., white-to-blue): best for data with a meaningful zero baseline
- **Diverging** (e.g., blue-white-red): best when values can be above or below a midpoint (like positive/negative change)

## Adjusting the Color Scale

By default, Atlas Charts maps the minimum value to the lightest color and the maximum to the darkest. If a few extreme outliers dominate the scale, go to **Customize - Color Scale** and set a custom min/max to cap the range. This prevents most cells from appearing light because one cell has a very high value.

## Cell Labels

Enable **Show Cell Values** in the Customize tab to display the raw number inside each cell. This is useful when the exact value matters, not just the relative intensity.

## Filtering

Use the Filter tab to restrict the heatmap to a relevant slice of data:

```javascript
{ "statusCode": { "$gte": 500 } }
```

This changes the heatmap to show only error request patterns, making it easy to see if errors spike at specific hours.

## Performance Considerations

Heatmaps work best with pre-aggregated or moderately sized datasets. For very large collections (hundreds of millions of documents), add a compound index on the two dimension fields so the aggregation runs efficiently:

```javascript
db.api_requests.createIndex({ timestamp: 1 });
db.api_requests.createIndex({ "timestamp": 1, "endpoint": 1 });
```

## Summary

Heatmaps in MongoDB Atlas Charts visualize a two-dimensional matrix where color intensity encodes a numeric measure. Configure two dimension fields on the X and Y axes and a numeric aggregation for intensity. Custom pipelines let you extract time components like hour and day-of-week for operational activity maps. Adjust the color scale to prevent outliers from washing out the rest of the gradient, and enable cell labels when exact values are needed.
