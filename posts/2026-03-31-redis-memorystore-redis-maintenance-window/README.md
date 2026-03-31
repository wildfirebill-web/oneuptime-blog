# How to Configure Memorystore Redis Maintenance Window

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, GCP, Memorystore, Maintenance, Configuration

Description: Learn how to configure maintenance windows for Memorystore for Redis so scheduled updates occur during low-traffic periods with minimal impact to your application.

---

Memorystore for Redis performs periodic maintenance for minor version updates and infrastructure patches. Configuring a maintenance window lets you schedule these operations during periods of low traffic.

## Maintenance Window Options

- **Day of week**: Any day (MONDAY through SUNDAY)
- **Start time**: Hour of day in UTC (0-23)
- **Duration**: Up to 1 hour

During maintenance on Standard HA instances, failover occurs to a replica and the primary is updated. Expect a brief reconnect (typically under 30 seconds).

## Setting a Maintenance Window

```bash
# Schedule maintenance for Sundays at 02:00 UTC
gcloud redis instances update prod-cache \
  --region=us-central1 \
  --maintenance-window-day=SUNDAY \
  --maintenance-window-hour=2
```

## Checking Current Maintenance Config

```bash
gcloud redis instances describe prod-cache \
  --region=us-central1 \
  --format="json(maintenancePolicy,maintenanceSchedule)"
```

Output:

```json
{
  "maintenancePolicy": {
    "weeklyMaintenanceWindow": [{
      "day": "SUNDAY",
      "startTime": {
        "hours": 2
      }
    }]
  },
  "maintenanceSchedule": {
    "startTime": "2026-04-06T02:00:00Z",
    "endTime": "2026-04-06T03:00:00Z"
  }
}
```

## Terraform Configuration

```hcl
resource "google_redis_instance" "prod" {
  name           = "prod-cache"
  region         = "us-central1"
  memory_size_gb = 10
  tier           = "STANDARD_HA"
  redis_version  = "REDIS_7_0"

  maintenance_policy {
    weekly_maintenance_window {
      day = "SUNDAY"
      start_time {
        hours   = 2
        minutes = 0
        seconds = 0
        nanos   = 0
      }
    }
  }
}
```

## Deferring Upcoming Maintenance

If scheduled maintenance conflicts with a planned event, you can defer it:

```bash
gcloud redis instances reschedule-maintenance prod-cache \
  --region=us-central1 \
  --reschedule-type=SPECIFIC_TIME \
  --schedule-time=2026-04-13T02:00:00Z
```

Or defer to the next available window:

```bash
gcloud redis instances reschedule-maintenance prod-cache \
  --region=us-central1 \
  --reschedule-type=NEXT_AVAILABLE_WINDOW
```

## Application Retry During Maintenance

```python
import redis
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(6),
    wait=wait_exponential(multiplier=1, min=2, max=30)
)
def redis_get(client: redis.Redis, key: str):
    return client.get(key)
```

## Monitoring Maintenance Events

```bash
gcloud logging read \
  'resource.type="redis_instance" AND protoPayload.methodName="maintenance"' \
  --limit=10 \
  --project=my-project
```

## Summary

Memorystore maintenance windows let you schedule updates for Sunday nights or other low-traffic periods. Configure the window using `gcloud redis instances update` or Terraform. For Standard HA instances, maintenance triggers a brief failover - add retry logic in your application to handle the short reconnect period gracefully.
