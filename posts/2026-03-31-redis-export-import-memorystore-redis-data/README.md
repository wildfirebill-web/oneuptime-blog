# How to Export and Import Memorystore Redis Data

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, GCP, Memorystore, Export, Import

Description: Learn how to export Memorystore for Redis data to Google Cloud Storage as an RDB snapshot and import it back to migrate or restore Redis datasets.

---

Memorystore supports exporting Redis data as an RDB file to Cloud Storage and importing RDB files from Cloud Storage. This enables data migration between instances, disaster recovery, and pre-seeding new environments.

## Prerequisites

- Memorystore Standard HA instance (Basic tier does not support export)
- A Cloud Storage bucket in the same region
- Memorystore service account must have Storage Object Admin on the bucket

## Granting Storage Access

```bash
# Get the Memorystore service account
gcloud redis instances describe prod-cache \
  --region=us-central1 \
  --format="value(persistenceConfig.rdbSnapshotPeriod)"

# Grant storage access
REDIS_SA="service-$(gcloud projects describe my-project --format='value(projectNumber)')@cloud-redis.iam.gserviceaccount.com"

gsutil iam ch serviceAccount:$REDIS_SA:objectAdmin gs://my-redis-backups
```

## Exporting Redis Data

```bash
gcloud redis instances export prod-cache \
  --region=us-central1 \
  --gcs-bucket=gs://my-redis-backups

# Monitor the export operation
gcloud redis operations list \
  --region=us-central1 \
  --filter="metadata.target:prod-cache"
```

This creates a file like `gs://my-redis-backups/dump.rdb` with a timestamp.

## Importing Redis Data

```bash
# Import into a new or existing instance
gcloud redis instances import new-cache \
  --region=us-central1 \
  --gcs-bucket=gs://my-redis-backups/dump.rdb
```

Note: Import replaces ALL data in the target instance. Use with caution on production instances.

## Full Migration Workflow

```bash
# 1. Export from source
gcloud redis instances export source-cache \
  --region=us-central1 \
  --gcs-bucket=gs://my-redis-migration

# 2. Create new instance (if needed)
gcloud redis instances create target-cache \
  --region=us-east1 \
  --size=10 \
  --tier=STANDARD_HA \
  --redis-version=redis_7_0

# 3. Import to target
gcloud redis instances import target-cache \
  --region=us-east1 \
  --gcs-bucket=gs://my-redis-migration/export-20260331-120000.rdb

# 4. Verify key count matches
gcloud redis instances describe target-cache --region=us-east1
```

## Terraform - Automated Export Schedule

Use Cloud Scheduler + Cloud Functions to automate periodic exports:

```hcl
resource "google_cloud_scheduler_job" "redis_backup" {
  name      = "redis-daily-export"
  region    = "us-central1"
  schedule  = "0 2 * * *"  # 2am daily
  time_zone = "UTC"

  http_target {
    uri         = google_cloudfunctions2_function.redis_export.url
    http_method = "POST"
  }
}
```

## Verifying the Export

```bash
# List exported files
gsutil ls -l gs://my-redis-backups/

# Check RDB file size
gsutil du -sh gs://my-redis-backups/dump.rdb
```

## Summary

Memorystore export saves your Redis dataset as an RDB file to Cloud Storage, and import loads it into any Memorystore instance. Use exports for cross-region migration, disaster recovery, or pre-seeding new environments. Schedule periodic exports with Cloud Scheduler for ongoing backup. Grant the Memorystore service account Storage Object Admin on your bucket before starting.
