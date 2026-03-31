# How to Use MongoDB with Apache Spark via the MongoDB Spark Connector

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Apache Spark, Connector, Big Data, Analytics

Description: Learn how to use the MongoDB Spark Connector to read MongoDB collections into Spark DataFrames and write Spark results back to MongoDB for large-scale data processing.

---

The MongoDB Spark Connector lets Apache Spark read from and write to MongoDB collections as DataFrames. This enables large-scale data processing, ETL pipelines, and machine learning on MongoDB data using the full Spark ecosystem.

## Setup

Add the connector to your Spark job. For PySpark, install the package or add it as a JAR:

```bash
pip install pyspark
pyspark \
  --packages org.mongodb.spark:mongo-spark-connector_2.12:10.3.0
```

Or add to `build.sbt` for Scala:

```text
libraryDependencies += "org.mongodb.spark" %% "mongo-spark-connector" % "10.3.0"
```

## Read a MongoDB Collection into a DataFrame

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("MongoDBSparkExample") \
    .config("spark.mongodb.read.connection.uri",
            "mongodb://localhost:27017/salesdb.orders") \
    .config("spark.mongodb.write.connection.uri",
            "mongodb://localhost:27017/salesdb.processed_orders") \
    .getOrCreate()

# Read entire collection
df = spark.read.format("mongodb").load()
df.printSchema()
df.show(5)
```

## Filter Data with Aggregation Pipelines

The connector can push aggregation pipelines down to MongoDB before loading data into Spark, reducing data transfer:

```python
pipeline = '[{"$match": {"status": "completed"}}, {"$project": {"_id": 1, "amount": 1, "category": 1}}]'

df = spark.read.format("mongodb") \
    .option("aggregation.pipeline", pipeline) \
    .load()
```

## Transform Data with Spark SQL

```python
df.createOrReplaceTempView("orders")

result = spark.sql("""
    SELECT
        category,
        COUNT(*) AS order_count,
        SUM(amount) AS total_revenue,
        AVG(amount) AS avg_order_value
    FROM orders
    GROUP BY category
    ORDER BY total_revenue DESC
""")

result.show()
```

## Write Results Back to MongoDB

```python
result.write.format("mongodb") \
    .mode("overwrite") \
    .option("spark.mongodb.write.connection.uri",
            "mongodb://localhost:27017/salesdb.category_summary") \
    .save()
```

## Use with MongoDB Atlas

For Atlas, use the `mongodb+srv` connection string:

```python
atlas_uri = "mongodb+srv://<user>:<password>@cluster0.example.mongodb.net/salesdb.orders"

df = spark.read.format("mongodb") \
    .option("spark.mongodb.read.connection.uri", atlas_uri) \
    .load()
```

## Run a Full ETL Pipeline

```python
# Read raw events
raw = spark.read.format("mongodb") \
    .option("spark.mongodb.read.connection.uri",
            "mongodb://localhost:27017/events.raw") \
    .load()

# Transform
from pyspark.sql.functions import col, to_date, sum as spark_sum

daily = raw \
    .withColumn("event_date", to_date(col("timestamp"))) \
    .groupBy("event_date", "event_type") \
    .agg(spark_sum("count").alias("total"))

# Write
daily.write.format("mongodb") \
    .mode("append") \
    .option("spark.mongodb.write.connection.uri",
            "mongodb://localhost:27017/events.daily_aggregates") \
    .save()
```

## Summary

The MongoDB Spark Connector enables large-scale processing of MongoDB data using Spark DataFrames and SQL. Push aggregation pipelines to MongoDB to reduce data transfer, use Spark for complex transformations and joins, and write results back to MongoDB collections. This pattern is well-suited for nightly ETL jobs, reporting aggregations, and feature engineering for machine learning pipelines.
