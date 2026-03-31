# How to Use ClickHouse with Retool

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Retool, Low-Code, Dashboard, Internal Tool, Analytics

Description: Connect Retool to ClickHouse to build internal analytics tools and dashboards using a low-code interface backed by fast columnar queries.

---

Retool is a low-code platform for building internal tools. Connecting it to ClickHouse allows operations, product, and data teams to query analytics data and build dashboards without writing application code.

## Adding ClickHouse as a Resource in Retool

Retool does not have a native ClickHouse connector, but ClickHouse exposes an HTTP interface that works with Retool's REST API resource type.

1. In Retool, go to Resources and click "Create new resource."
2. Select "REST API."
3. Set the Base URL to your ClickHouse HTTP endpoint: `http://your-host:8123`
4. Add a header: `X-ClickHouse-User: default`
5. Add a header: `X-ClickHouse-Key: your-password` (leave blank if no password)

## Running Queries via the HTTP Interface

ClickHouse accepts SQL queries as HTTP POST requests with the query in the body.

In a Retool REST API query:

- Method: `POST`
- URL: `/`
- Body type: `Raw`
- Body content:

```sql
SELECT
  toDate(created_at) AS day,
  count() AS events
FROM default.events
WHERE created_at >= today() - 30
GROUP BY day
ORDER BY day
```

Add `?default_format=JSONEachRow` to the URL for easy JSON parsing.

Full URL example:

```text
http://your-host:8123/?default_format=JSONEachRow&database=default
```

## Displaying Results in a Table

1. Add a Table component to the Retool canvas.
2. Set its Data source to your ClickHouse query.
3. The JSON array returned by ClickHouse maps directly to Retool table rows.

## Building a Chart Component

1. Add a Chart component.
2. Set Dataset to `{{ query1.data }}`.
3. Map X-axis to `day` and Y-axis to `events`.
4. Select Bar or Line chart type.

## Adding User Input Filters

Use a Date Range Picker component and reference it in the query:

```sql
SELECT
  toDate(created_at) AS day,
  count() AS events
FROM default.events
WHERE created_at BETWEEN '{{ dateRange.value[0] }}' AND '{{ dateRange.value[1] }}'
GROUP BY day
ORDER BY day
```

Enable "Run query automatically when inputs change" to make the chart refresh on filter updates.

## Parameterized Queries for Security

Avoid string concatenation by using Retool's query parameter syntax where possible, or validate inputs before embedding them in SQL strings.

```text
?query=SELECT+count()+FROM+events+WHERE+user_id+%3D+{{ userId.value }}
```

## Summary

Retool's REST API resource type gives you a flexible way to connect to ClickHouse's HTTP interface. Once connected, you can build internal dashboards, data exploration tools, and operational reports that leverage ClickHouse's speed without writing any backend application code.
