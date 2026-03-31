# How to Use ClickHouse with PySpark

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, PySpark, Spark, Python, Big Data

Description: Connect PySpark to ClickHouse using the JDBC connector to read and write data, enabling distributed processing of ClickHouse data with Spark's computation engine.

---

## When to Use PySpark with ClickHouse

ClickHouse handles analytical queries on a single cluster very well. PySpark makes sense when you need to join ClickHouse data with data in S3, HDFS, or other sources, or when you need Spark's ML libraries on top of ClickHouse data.

## Setting Up the JDBC Connection

Download the ClickHouse JDBC driver:

```bash
wget https://github.com/ClickHouse/clickhouse-java/releases/download/v0.6.0/clickhouse-jdbc-0.6.0-all.jar
```

Initialize Spark with the JDBC jar:

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("ClickHouseSpark") \
    .config("spark.jars", "/opt/spark/jars/clickhouse-jdbc-0.6.0-all.jar") \
    .getOrCreate()
```

## Reading from ClickHouse

```python
JDBC_URL = "jdbc:clickhouse://localhost:8123/default"

df = spark.read \
    .format("jdbc") \
    .option("url", JDBC_URL) \
    .option("dbtable", "events") \
    .option("user", "default") \
    .option("password", "") \
    .option("driver", "com.clickhouse.jdbc.ClickHouseDriver") \
    .load()

df.printSchema()
df.show(5)
```

## Pushing Down Filters

Use `query` option instead of `dbtable` to push predicates to ClickHouse:

```python
df_recent = spark.read \
    .format("jdbc") \
    .option("url", JDBC_URL) \
    .option("query", "SELECT * FROM events WHERE event_date = today() AND user_id < 10000") \
    .option("user", "default") \
    .option("driver", "com.clickhouse.jdbc.ClickHouseDriver") \
    .load()
```

This runs the filter in ClickHouse before transferring data to Spark, reducing network traffic.

## Parallel Reads with Partitioning

For large tables, read multiple partitions in parallel:

```python
df_parallel = spark.read \
    .format("jdbc") \
    .option("url", JDBC_URL) \
    .option("dbtable", "events") \
    .option("partitionColumn", "user_id") \
    .option("lowerBound", "0") \
    .option("upperBound", "1000000") \
    .option("numPartitions", "10") \
    .option("driver", "com.clickhouse.jdbc.ClickHouseDriver") \
    .load()
```

## Processing with Spark

```python
from pyspark.sql import functions as F

result = df_recent \
    .groupBy("event_type") \
    .agg(
        F.count("*").alias("count"),
        F.countDistinct("user_id").alias("unique_users")
    ) \
    .orderBy("count", ascending=False)

result.show()
```

## Writing Back to ClickHouse

```python
result.write \
    .format("jdbc") \
    .option("url", JDBC_URL) \
    .option("dbtable", "event_summary") \
    .option("user", "default") \
    .option("driver", "com.clickhouse.jdbc.ClickHouseDriver") \
    .mode("append") \
    .save()
```

## Summary

PySpark connects to ClickHouse via the JDBC driver, enabling distributed reads with partition pushdown and seamless Spark DataFrame operations. Push aggregations to ClickHouse when possible and use PySpark for complex joins with external sources or Spark ML workloads.
