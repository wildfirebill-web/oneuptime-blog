# How to Create Aggregate UDFs in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, UDF, Aggregate Function, Custom Aggregation, Analytics

Description: Learn how to create custom aggregate UDFs in ClickHouse to implement domain-specific aggregations not covered by built-in aggregate functions.

---

ClickHouse's built-in aggregate functions cover most use cases, but sometimes you need custom aggregation logic - for example, a weighted median, a custom scoring formula, or a domain-specific rollup. Executable aggregate UDFs let you implement these in an external script.

## How Aggregate Executable UDFs Work

ClickHouse groups rows by the GROUP BY key, serializes each group, pipes rows to the script's stdin, and expects a single aggregated value per group on stdout.

## Defining an Aggregate Executable UDF

Create `/etc/clickhouse-server/user_defined/weighted_avg.xml`:

```text
<functions>
    <function>
        <name>weightedAvg</name>
        <type>executable_pool</type>
        <return_type>Float64</return_type>
        <argument>
            <type>Float64</type>
            <name>value</name>
        </argument>
        <argument>
            <type>Float64</type>
            <name>weight</name>
        </argument>
        <format>TabSeparated</format>
        <command>python3 /var/lib/clickhouse/user_scripts/weighted_avg.py</command>
        <send_chunk_header>true</send_chunk_header>
    </function>
</functions>
```

## Writing the Aggregation Script

```text
#!/usr/bin/env python3
import sys

def weighted_average(pairs):
    total_weight = sum(w for _, w in pairs)
    if total_weight == 0:
        return 0.0
    return sum(v * w for v, w in pairs) / total_weight

for line in sys.stdin:
    line = line.strip()
    if not line:
        continue
    parts = line.split('\t')
    if len(parts) == 1:
        # chunk header: number of rows
        n = int(parts[0])
        rows = []
        for _ in range(n):
            row = sys.stdin.readline().strip().split('\t')
            rows.append((float(row[0]), float(row[1])))
        print(weighted_average(rows))
        sys.stdout.flush()
```

## SQL-Level Aggregate UDFs

ClickHouse 23.4+ supports aggregate UDFs defined fully in SQL using `CREATE AGGREGATE FUNCTION` for simple stateful aggregations. For most custom logic, the executable approach above is used.

## Calling the Aggregate UDF

```sql
SELECT
    product_id,
    weightedAvg(review_score, review_count) AS weighted_score
FROM product_reviews
GROUP BY product_id
ORDER BY weighted_score DESC
LIMIT 10;
```

## Using with combinators

ClickHouse aggregate combinators also work with custom UDFs in some cases:

```sql
SELECT
    product_id,
    weightedAvgIf(review_score, review_count, review_count > 5) AS filtered_score
FROM product_reviews
GROUP BY product_id;
```

## Monitoring Executable UDF Processes

```sql
SELECT name, origin
FROM system.functions
WHERE name = 'weightedAvg';
```

## Practical Example - Custom Scoring

```sql
CREATE FUNCTION scoreEvent AS (clicks, impressions, conversions) ->
    (conversions * 10.0 + clicks * 1.0) / greatest(impressions, 1);

SELECT
    campaign_id,
    sum(scoreEvent(clicks, impressions, conversions)) AS total_score
FROM ad_events
GROUP BY campaign_id
ORDER BY total_score DESC;
```

## Summary

Aggregate UDFs in ClickHouse enable custom grouping logic beyond built-in functions. For simple expression-based aggregations, SQL UDFs combined with `sum`, `avg`, and combinators cover most needs. For fully custom stateful aggregations, executable pool UDFs let you implement any logic in Python or another language and call it from GROUP BY queries.
