# How to Configure BigQuery Data Transfer Service with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GCP, BigQuery, Data Transfer, OpenTofu, Analytics, ETL

Description: Learn how to configure BigQuery Data Transfer Service with OpenTofu to automate scheduled data imports from Google services and third-party sources into BigQuery.

## Overview

BigQuery Data Transfer Service automates data movement into BigQuery from Google SaaS apps (Google Ads, YouTube, etc.) and external sources on a scheduled basis. OpenTofu manages transfer configurations, schedules, and destinations.

## Step 1: Enable the Data Transfer API

```hcl
# main.tf - Enable BigQuery Data Transfer Service API
resource "google_project_service" "bigquery_transfer" {
  service = "bigquerydatatransfer.googleapis.com"
}
```

## Step 2: Configure a Google Ads Data Transfer

```hcl
# Service account for data transfer
resource "google_service_account" "transfer_sa" {
  account_id   = "bq-transfer-sa"
  display_name = "BigQuery Transfer Service Account"
}

resource "google_project_iam_member" "transfer_sa_bq_admin" {
  project = var.project_id
  role    = "roles/bigquery.dataOwner"
  member  = "serviceAccount:${google_service_account.transfer_sa.email}"
}

# Create destination BigQuery dataset
resource "google_bigquery_dataset" "ads_dataset" {
  dataset_id = "google_ads_data"
  location   = "US"

  labels = {
    source = "google-ads"
  }
}

# Data transfer config for Google Ads
resource "google_bigquery_data_transfer_config" "ads_transfer" {
  display_name           = "Google Ads Campaign Data"
  location               = "US"
  data_source_id         = "google_ads"
  destination_dataset_id = google_bigquery_dataset.ads_dataset.dataset_id

  # Run daily at midnight UTC
  schedule = "every 24 hours"

  params = {
    # Google Ads customer ID (without hyphens)
    customer_id            = var.google_ads_customer_id
    # Which reports to transfer
    table_filter           = "AdGroupBasicStats,CampaignBasicStats,AccountStats"
    # Number of days to transfer
    run_mode               = "INCREMENTAL"
  }

  service_account_name = google_service_account.transfer_sa.email

  depends_on = [
    google_project_service.bigquery_transfer,
    google_project_iam_member.transfer_sa_bq_admin,
  ]
}
```

## Step 3: Scheduled Query Transfer

```hcl
# Scheduled query - runs a SQL query on a schedule and writes results
resource "google_bigquery_dataset" "analytics_dataset" {
  dataset_id = "analytics_aggregates"
  location   = "US"
}

resource "google_bigquery_data_transfer_config" "daily_aggregation" {
  display_name           = "Daily Sales Aggregation"
  location               = "US"
  data_source_id         = "scheduled_query"
  destination_dataset_id = google_bigquery_dataset.analytics_dataset.dataset_id
  schedule               = "every day 06:00"

  params = {
    query = <<-SQL
      SELECT
        DATE(order_date) as date,
        product_category,
        SUM(amount) as total_revenue,
        COUNT(*) as order_count
      FROM `${var.project_id}.raw_data.orders`
      WHERE DATE(order_date) = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)
      GROUP BY 1, 2
    SQL
    destination_table_name_template = "daily_sales_{run_date}"
    write_disposition                = "WRITE_TRUNCATE"
    partitioning_field               = "date"
  }

  service_account_name = google_service_account.transfer_sa.email
}
```

## Summary

BigQuery Data Transfer Service with OpenTofu automates data ingestion from Google services and scheduled SQL transformations into BigQuery. By defining transfer configurations as code, you ensure consistent data pipeline setup across projects and can version-control data ingestion schedules and queries.
