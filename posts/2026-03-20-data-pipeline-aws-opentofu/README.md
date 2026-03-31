# How to Build a Data Pipeline Architecture with OpenTofu on AWS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: AWS, Data Pipeline, Kinesis, Lambda, S3, Glue, Athena, OpenTofu

Description: Learn how to build a real-time data pipeline on AWS using OpenTofu with Kinesis Data Streams, Lambda processing, S3 data lake, Glue ETL, and Athena for analytics.

## Overview

A data pipeline on AWS ingests streaming data via Kinesis, processes it with Lambda, stores raw and transformed data in S3 (data lake), catalogs it with Glue, and queries it with Athena. OpenTofu provisions the complete pipeline infrastructure.

## Step 1: Kinesis Data Stream (Ingestion)

```hcl
# main.tf - Real-time data ingestion stream

resource "aws_kinesis_stream" "events" {
  name             = "application-events"
  shard_count      = 4  # 4 MB/s write, 8 MB/s read capacity
  retention_period = 24

  shard_level_metrics = [
    "IncomingBytes",
    "OutgoingBytes",
    "IteratorAgeMilliseconds",
  ]

  stream_mode_details {
    stream_mode = "PROVISIONED"  # or ON_DEMAND for variable load
  }

  encryption_type = "KMS"
  kms_key_id      = aws_kms_key.kinesis.arn
}
```

## Step 2: S3 Data Lake with Partitioned Structure

```hcl
# S3 data lake bucket
resource "aws_s3_bucket" "data_lake" {
  bucket = "company-data-lake"
}

resource "aws_s3_bucket_versioning" "data_lake" {
  bucket = aws_s3_bucket.data_lake.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_lifecycle_configuration" "data_lake" {
  bucket = aws_s3_bucket.data_lake.id

  rule {
    id     = "transition-to-ia"
    status = "Enabled"

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 90
      storage_class = "GLACIER"
    }
  }
}

# Kinesis Firehose for raw data delivery to S3
resource "aws_kinesis_firehose_delivery_stream" "raw_events" {
  name        = "raw-events-to-s3"
  destination = "extended_s3"

  kinesis_source_configuration {
    kinesis_stream_arn = aws_kinesis_stream.events.arn
    role_arn           = aws_iam_role.firehose.arn
  }

  extended_s3_configuration {
    role_arn   = aws_iam_role.firehose.arn
    bucket_arn = aws_s3_bucket.data_lake.arn
    prefix     = "raw/events/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/"

    buffering_size     = 128  # MB
    buffering_interval = 300  # seconds

    # Convert JSON to Parquet for cost-efficient queries
    data_format_conversion_configuration {
      input_format_configuration {
        deserializer { hive_json_ser_de {} }
      }
      output_format_configuration {
        serializer { parquet_ser_de {} }
      }
      schema_configuration {
        database_name = aws_glue_catalog_database.events.name
        table_name    = aws_glue_catalog_table.events.name
        role_arn      = aws_iam_role.firehose.arn
      }
    }
  }
}
```

## Step 3: Lambda for Real-Time Processing

```hcl
# Lambda processes events from Kinesis for real-time transformations
resource "aws_lambda_event_source_mapping" "kinesis" {
  event_source_arn              = aws_kinesis_stream.events.arn
  function_name                 = aws_lambda_function.stream_processor.arn
  starting_position             = "LATEST"
  batch_size                    = 100
  parallelization_factor        = 4  # 4 concurrent Lambda per shard
  bisect_batch_on_function_error = true

  destination_config {
    on_failure {
      destination_arn = aws_sqs_queue.dlq.arn
    }
  }
}
```

## Step 4: Glue Catalog and Crawler

```hcl
# Glue catalog database
resource "aws_glue_catalog_database" "events" {
  name = "events_db"
}

# Crawler to discover schema from S3 data
resource "aws_glue_crawler" "events" {
  name          = "events-crawler"
  role          = aws_iam_role.glue.arn
  database_name = aws_glue_catalog_database.events.name

  s3_target {
    path = "s3://${aws_s3_bucket.data_lake.bucket}/processed/"
  }

  schedule = "cron(0 1 * * ? *)"  # Run daily at 1 AM
}

# Glue ETL job for data transformation
resource "aws_glue_job" "transform" {
  name         = "events-transform"
  role_arn     = aws_iam_role.glue.arn
  glue_version = "4.0"

  command {
    name            = "glueetl"
    script_location = "s3://${aws_s3_bucket.scripts.bucket}/transform.py"
    python_version  = "3"
  }

  default_arguments = {
    "--enable-job-bookmarks"        = "enable"
    "--enable-continuous-cloudwatch-log" = "true"
    "--TempDir"                     = "s3://${aws_s3_bucket.data_lake.bucket}/temp/"
  }

  number_of_workers = 10
  worker_type       = "G.1X"
}
```

## Step 5: Athena for Analytics

```hcl
# Athena workgroup for query execution
resource "aws_athena_workgroup" "analytics" {
  name = "analytics"

  configuration {
    enforce_workgroup_configuration    = true
    publish_cloudwatch_metrics_enabled = true

    result_configuration {
      output_location = "s3://${aws_s3_bucket.data_lake.bucket}/athena-results/"
      encryption_configuration {
        encryption_option = "SSE_S3"
      }
    }

    bytes_scanned_cutoff_per_query = 10 * 1024 * 1024 * 1024  # 10 GB limit per query
  }
}
```

## Summary

A data pipeline on AWS built with OpenTofu ingests real-time events via Kinesis, stores them in a partitioned S3 data lake in Parquet format for cost-efficient querying, and makes them queryable via Athena without provisioning any database servers. Glue crawlers automatically update the schema catalog as data formats evolve, and Glue ETL jobs handle complex transformations at scale using Apache Spark.
