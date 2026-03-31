# How to Create a Gauge Chart in MongoDB Atlas Charts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Charts, Visualization, Gauge Chart, KPI

Description: Build a gauge chart in MongoDB Atlas Charts to display a single KPI value against a target range for at-a-glance status monitoring.

---

## When to Use a Gauge Chart

Gauge charts display a single aggregate value on a dial or arc, making it immediately clear whether a metric is in a good, warning, or critical zone. They are ideal for:
- SLA compliance percentage (target: above 99.9%)
- Average response time (target: below 200ms)
- CPU utilization across a fleet
- Order fulfillment rate (target: above 95%)

## Creating a Gauge Chart

1. In your Atlas Dashboard, click **Add Chart**
2. Select **Chart Type: Gauge**
3. Choose your data source

## Configuring the Value

The gauge requires a single numeric value. Drag a field to the **Value** channel and choose an aggregation:

```
Value:  responseTimeMs    (Aggregation: Average)
```

Or use a Count aggregation to show total document count:

```
Value:  _id               (Aggregation: Count)
```

## Setting Min, Max, and Target

In the **Customize** tab, configure the gauge range:

```
Minimum:   0
Maximum:   500
Target:    200
```

The gauge needle or arc fill will indicate whether the current value is above or below the target. Setting a meaningful maximum is important - a response time gauge capped at 500ms gives a different visual reading than one capped at 5000ms.

## Configuring Color Bands

Divide the gauge arc into color bands to create visual thresholds:

```
0 - 200ms:    Green   (healthy)
200 - 350ms:  Yellow  (warning)
350 - 500ms:  Red     (critical)
```

In the Customize tab, click **Add Color Band** and configure each range with a color and label. This immediately communicates status without requiring the viewer to interpret the exact number.

## Example KPI for Uptime Monitoring

If you store uptime check results in MongoDB:

```javascript
// uptime_checks collection
{
  _id: ObjectId("..."),
  service: "api-gateway",
  status: "up",         // "up" | "down"
  responseTimeMs: 145,
  checkedAt: ISODate("2024-03-15T12:00:00Z")
}
```

Use a custom pipeline to calculate the uptime percentage for the last 24 hours:

```javascript
[
  {
    "$match": {
      "service": "api-gateway",
      "checkedAt": { "$gte": { "$date": { "$subtract": [{ "$date": "$$NOW" }, 86400000] } } }
    }
  },
  {
    "$group": {
      "_id": null,
      "total": { "$sum": 1 },
      "upCount": {
        "$sum": { "$cond": [{ "$eq": ["$status", "up"] }, 1, 0] }
      }
    }
  },
  {
    "$addFields": {
      "uptimePercent": {
        "$multiply": [{ "$divide": ["$upCount", "$total"] }, 100]
      }
    }
  }
]
```

Map `uptimePercent` to the Value channel. Set Min: 0, Max: 100, Target: 99.9. Configure bands: 99.9-100 green, 99-99.9 yellow, 0-99 red.

## Number Formatting

In Customize, configure the value display:
- **Suffix**: `%` for percentages or `ms` for latency
- **Decimal Places**: 2 for percentages, 0 for counts
- **Number Format**: `Compact` for large values (e.g., `12.3K`)

## Using Gauge Charts in a KPI Row

A common dashboard pattern is a horizontal row of 3-4 gauge charts at the top, each showing a different KPI. Atlas Charts supports this layout through flexible resize and positioning in dashboard edit mode.

## Refreshing Interval

Set the dashboard auto-refresh interval (30 seconds, 1 minute, etc.) so gauge charts update automatically. In Atlas Charts, go to Dashboard Settings and configure the refresh rate.

## Summary

Gauge charts in MongoDB Atlas Charts visualize a single aggregate metric against a target range. Configure the Value channel with an aggregation, set meaningful Min/Max bounds, and add color bands for green/yellow/red thresholds. Custom aggregation pipelines let you compute complex KPIs like uptime percentage before the value reaches the gauge. Use gauge charts as status indicators at the top of operational dashboards for at-a-glance visibility.
