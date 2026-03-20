# How to Build a Data Analytics Platform with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Data Analytics, S3, Glue, Athena, Redshift, Infrastructure as Code

Description: Learn how to build a data analytics platform on AWS with OpenTofu using S3 data lake, Glue for ETL, Athena for SQL queries, and Redshift for data warehousing.

## Introduction

A data analytics platform on AWS uses S3 as the data lake foundation, Glue for data catalog and ETL, Athena for ad-hoc SQL queries, and optionally Redshift for complex analytical workloads. This guide builds the entire platform with OpenTofu.

## S3 Data Lake Structure

```hcl
locals {
  data_lake_layers = ["raw", "processed", "curated"]
}

resource "aws_s3_bucket" "data_lake" {
  for_each = toset(local.data_lake_layers)
  bucket   = "mycompany-datalake-${each.key}-${data.aws_caller_identity.current.account_id}"
}

resource "aws_s3_bucket_versioning" "data_lake" {
  for_each = toset(local.data_lake_layers)
  bucket   = aws_s3_bucket.data_lake[each.key].id

  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_lifecycle_configuration" "raw" {
  bucket = aws_s3_bucket.data_lake["raw"].id

  rule {
    id     = "archive-old-raw-data"
    status = "Enabled"

    transition {
      days          = 90
      storage_class = "INTELLIGENT_TIERING"
    }
  }
}
```

## AWS Glue Data Catalog

```hcl
resource "aws_glue_catalog_database" "analytics" {
  name        = "myapp_analytics_${var.environment}"
  description = "Analytics data catalog for MyApp"
}

resource "aws_glue_crawler" "events" {
  name          = "myapp-events-crawler"
  role          = aws_iam_role.glue_crawler.arn
  database_name = aws_glue_catalog_database.analytics.name

  s3_target {
    path = "s3://${aws_s3_bucket.data_lake["processed"].bucket}/events/"
  }

  schema_change_policy {
    delete_behavior = "LOG"
    update_behavior = "UPDATE_IN_DATABASE"
  }

  schedule = "cron(0 6 * * ? *)"  # daily at 6 AM UTC
}
```

## Glue ETL Job

```hcl
resource "aws_glue_job" "transform_events" {
  name         = "myapp-transform-events"
  role_arn     = aws_iam_role.glue_job.arn
  glue_version = "4.0"
  worker_type  = "G.1X"
  number_of_workers = 5

  command {
    name            = "glueetl"
    script_location = "s3://${aws_s3_bucket.scripts.bucket}/glue/transform_events.py"
    python_version  = "3"
  }

  default_arguments = {
    "--enable-metrics"                = "true"
    "--enable-observability-metrics"  = "true"
    "--enable-spark-ui"               = "true"
    "--job-language"                  = "python"
    "--SOURCE_BUCKET"                 = aws_s3_bucket.data_lake["raw"].bucket
    "--DESTINATION_BUCKET"            = aws_s3_bucket.data_lake["processed"].bucket
  }

  execution_property {
    max_concurrent_runs = 1
  }
}
```

## Athena Workgroup

```hcl
resource "aws_athena_workgroup" "analytics" {
  name        = "myapp-analytics-${var.environment}"
  description = "Analytics workgroup with cost controls"

  configuration {
    enforce_workgroup_configuration    = true
    publish_cloudwatch_metrics_enabled = true

    result_configuration {
      output_location = "s3://${aws_s3_bucket.athena_results.bucket}/query-results/"

      encryption_configuration {
        encryption_option = "SSE_KMS"
        kms_key           = aws_kms_key.analytics.arn
      }
    }

    engine_version {
      selected_engine_version = "Athena engine version 3"
    }

    bytes_scanned_cutoff_per_query = 10737418240  # 10 GB limit per query
  }
}
```

## Redshift Serverless (Optional)

```hcl
resource "aws_redshiftserverless_namespace" "analytics" {
  namespace_name      = "myapp-analytics-${var.environment}"
  db_name             = "analytics"
  admin_username      = "admin"
  manage_admin_password = true  # Redshift manages in Secrets Manager
}

resource "aws_redshiftserverless_workgroup" "analytics" {
  namespace_name = aws_redshiftserverless_namespace.analytics.namespace_name
  workgroup_name = "myapp-analytics-${var.environment}"
  base_capacity  = 8  # RPUs (Redshift Processing Units)

  config_parameter {
    parameter_key   = "max_query_execution_time"
    parameter_value = "3600"  # 1 hour max query time
  }

  publicly_accessible = false
  subnet_ids          = module.vpc.private_subnets
  security_group_ids  = [aws_security_group.redshift.id]
}
```

## Summary

A data analytics platform on AWS uses a layered S3 data lake (raw → processed → curated), Glue for cataloging and ETL transformations, Athena for serverless SQL queries against S3 data, and optionally Redshift Serverless for complex multi-table analytics. Set bytes_scanned_cutoff on Athena workgroups to control costs, use Intelligent Tiering on the raw layer for automatic storage optimization, and schedule Glue crawlers to keep the data catalog current as new data arrives.
