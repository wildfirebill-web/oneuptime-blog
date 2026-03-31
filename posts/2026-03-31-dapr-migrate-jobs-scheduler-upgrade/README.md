# How to Migrate Jobs During Dapr Scheduler Upgrade

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Scheduler, Migration, Upgrade, Kubernetes

Description: Migrate scheduled jobs safely during a Dapr Scheduler upgrade by exporting job definitions, validating them, and re-importing after the upgrade completes.

---

## Why Job Migration Is Needed

When upgrading Dapr Scheduler between major versions, the embedded etcd data format or schema may change. To prevent job loss, export all job definitions before upgrading and re-import them after the new version is running. This is especially important when going from Dapr 1.13 to 1.14 or later.

## Step 1 - Export Existing Jobs

Before upgrading, list and export all scheduled jobs using the Dapr Jobs API:

```bash
#!/bin/bash
# export-jobs.sh
DAPR_PORT=3500
APP_ID="my-scheduler-app"

# Get job list via Dapr sidecar
JOB_NAMES=("daily-report" "hourly-sync" "cleanup-task" "notification-batch")

mkdir -p ./job-exports

for JOB in "${JOB_NAMES[@]}"; do
  echo "Exporting job: $JOB"
  curl -s http://localhost:${DAPR_PORT}/v1.0-alpha1/jobs/${JOB} \
    -o ./job-exports/${JOB}.json
done

echo "Export complete. Jobs saved to ./job-exports/"
```

## Step 2 - Validate Exported Jobs

Verify the exported JSON before proceeding:

```bash
for f in ./job-exports/*.json; do
  echo "Validating: $f"
  python3 -m json.tool "$f" > /dev/null && echo "OK" || echo "INVALID"
done
```

## Step 3 - Upgrade the Scheduler

Perform the Helm upgrade:

```bash
helm upgrade dapr dapr/dapr \
  --namespace dapr-system \
  --version 1.14.0 \
  --reuse-values

kubectl rollout status statefulset/dapr-scheduler -n dapr-system
```

## Step 4 - Re-Import Jobs

After the upgrade, re-create the jobs:

```bash
#!/bin/bash
# import-jobs.sh
DAPR_PORT=3500

for f in ./job-exports/*.json; do
  JOB_NAME=$(basename "$f" .json)
  echo "Importing job: $JOB_NAME"
  curl -X POST http://localhost:${DAPR_PORT}/v1.0-alpha1/jobs/${JOB_NAME} \
    -H "Content-Type: application/json" \
    -d @"$f"
  echo ""
done

echo "Import complete."
```

## Step 5 - Verify Jobs Are Active

```bash
# Check each imported job
for JOB in daily-report hourly-sync cleanup-task notification-batch; do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
    http://localhost:3500/v1.0-alpha1/jobs/$JOB)
  echo "$JOB: HTTP $STATUS"
done
```

## Rollback Plan

If the upgrade fails, restore from the backup and roll back:

```bash
helm rollback dapr -n dapr-system
kubectl rollout status statefulset/dapr-scheduler -n dapr-system
# Re-import jobs from export directory
```

## Summary

Migrating jobs during a Dapr Scheduler upgrade requires a careful export-upgrade-import sequence. Export all job definitions before upgrading, perform the Helm upgrade, verify the Scheduler is healthy, then re-import jobs from the saved definitions. Always test the migration procedure in a staging environment before applying to production.
