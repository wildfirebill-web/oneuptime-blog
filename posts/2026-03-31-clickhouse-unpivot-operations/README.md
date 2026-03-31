# How to Implement Unpivot Operations in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Unpivot, ARRAY JOIN, Data Transformation, Analytics

Description: Implement unpivot operations in ClickHouse to transform wide columnar data into long row-based format using ARRAY JOIN and tuple arrays.

---

## What is Unpivot?

Unpivot (the inverse of pivot) transforms wide tables - where each column represents a category - into tall tables with separate rows per category. This is needed when you receive data in wide format but need long format for analytics or charting.

## Method 1: UNION ALL (Simple but Verbose)

The most straightforward approach for small numbers of columns:

```sql
-- Wide format: one row per user with scores per subject
-- Transform to long format using UNION ALL

SELECT user_id, 'math' AS subject, math_score AS score FROM student_scores WHERE math_score IS NOT NULL
UNION ALL
SELECT user_id, 'science' AS subject, science_score AS score FROM student_scores WHERE science_score IS NOT NULL
UNION ALL
SELECT user_id, 'english' AS subject, english_score AS score FROM student_scores WHERE english_score IS NOT NULL
UNION ALL
SELECT user_id, 'history' AS subject, history_score AS score FROM student_scores WHERE history_score IS NOT NULL
ORDER BY user_id, subject;
```

## Method 2: ARRAY JOIN (Preferred ClickHouse Approach)

For tables with many columns to unpivot, ARRAY JOIN is more concise and efficient:

```sql
-- Monthly sales by channel in wide format
-- channel_sales has columns: month, organic, paid_search, social, email, direct

SELECT
    month,
    channel,
    revenue
FROM (
    SELECT
        month,
        ['organic', 'paid_search', 'social', 'email', 'direct'] AS channel_names,
        [organic, paid_search, social, email, direct] AS revenues
    FROM monthly_channel_sales
)
ARRAY JOIN
    channel_names AS channel,
    revenues AS revenue
WHERE revenue > 0
ORDER BY month, channel;
```

## Method 3: Unpivot Wide Metric Rows

For observability or IoT data with multiple metric columns:

```sql
-- sensor_readings has columns: reading_time, temp_c, humidity_pct, pressure_hpa, co2_ppm

SELECT
    reading_time,
    station_id,
    metric_name,
    metric_value
FROM (
    SELECT
        reading_time,
        station_id,
        ['temp_c', 'humidity_pct', 'pressure_hpa', 'co2_ppm'] AS metric_names,
        [temp_c, humidity_pct, pressure_hpa, co2_ppm] AS metric_values
    FROM sensor_readings
    WHERE reading_time >= today() - 7
)
ARRAY JOIN
    metric_names AS metric_name,
    metric_values AS metric_value
ORDER BY reading_time, station_id, metric_name;
```

## Creating a Long-Format Table from Wide Format

```sql
-- Create normalized target table
CREATE TABLE metrics_long
(
    reading_time DateTime,
    station_id UInt32,
    metric_name LowCardinality(String),
    metric_value Float64
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(reading_time)
ORDER BY (station_id, metric_name, reading_time);

-- Insert unpivoted data
INSERT INTO metrics_long
SELECT
    reading_time,
    station_id,
    metric_name,
    metric_value
FROM (
    SELECT
        reading_time,
        station_id,
        ['temp_c', 'humidity_pct', 'pressure_hpa'] AS names,
        [toFloat64(temp_c), toFloat64(humidity_pct), toFloat64(pressure_hpa)] AS values
    FROM sensor_readings
)
ARRAY JOIN names AS metric_name, values AS metric_value;
```

## Dynamic Unpivot in Application Layer

For truly dynamic column sets:

```python
import clickhouse_connect

client = clickhouse_connect.get_client(host='localhost')

def unpivot_query(table, id_cols, value_cols):
    id_select = ', '.join(id_cols)
    names_array = "['" + "', '".join(value_cols) + "']"
    values_array = '[' + ', '.join(f'toFloat64({c})' for c in value_cols) + ']'

    return f"""
    SELECT {id_select}, metric_name, metric_value
    FROM (
        SELECT {id_select},
            {names_array} AS names,
            {values_array} AS vals
        FROM {table}
    )
    ARRAY JOIN names AS metric_name, vals AS metric_value
    """

query = unpivot_query('sensor_readings', ['reading_time', 'station_id'],
                      ['temp_c', 'humidity_pct', 'pressure_hpa'])
result = client.query(query)
```

## Summary

Unpivot operations in ClickHouse are best implemented with ARRAY JOIN using two parallel arrays: column names and their values. This is more concise and efficient than UNION ALL for wide tables with many columns. For normalization, insert unpivoted results into a long-format table with a `metric_name` column, which supports flexible time-series queries across all metric types.
