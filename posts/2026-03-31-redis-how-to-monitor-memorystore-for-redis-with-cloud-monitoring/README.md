# How to Monitor Memorystore for Redis with Cloud Monitoring

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Memorystore, GCP, Cloud Monitoring, Observability

Description: Learn how to monitor Google Cloud Memorystore for Redis using Cloud Monitoring metrics, alerting policies, and dashboards to track performance and health.

---

## Available Metrics for Memorystore

Google Cloud Monitoring automatically collects Memorystore metrics under the `redis.googleapis.com` namespace. Key categories:

**Memory:**
- `redis.googleapis.com/stats/memory/usage` - bytes used for caching
- `redis.googleapis.com/stats/memory/usage_ratio` - memory used vs. total available
- `redis.googleapis.com/stats/memory/maxmemory` - configured max memory

**Commands:**
- `redis.googleapis.com/commands/calls` - command calls per second by command name
- `redis.googleapis.com/commands/usec_per_call` - average latency per command

**Hit Rate:**
- `redis.googleapis.com/stats/keyspace_hits` - successful key lookups
- `redis.googleapis.com/stats/keyspace_misses` - failed key lookups

**Replication:**
- `redis.googleapis.com/replication/master/slaves/lag` - replica lag in seconds
- `redis.googleapis.com/replication/master/slaves/offset` - replication offset

**Connections:**
- `redis.googleapis.com/clients/connected` - current connected clients
- `redis.googleapis.com/clients/blocked` - clients blocked waiting for a command

## Viewing Metrics via gcloud

```bash
# List all available Memorystore metrics
gcloud monitoring metrics list \
  --filter="metric.type:redis.googleapis.com" \
  --project=my-project

# Read memory usage ratio for a specific instance
gcloud monitoring read \
  "redis.googleapis.com/stats/memory/usage_ratio" \
  --project=my-project \
  --filter="resource.labels.instance_id=my-redis-instance" \
  --align-period=60s \
  --aligner=ALIGN_MEAN
```

## Creating Alerting Policies

### High Memory Usage Alert

```bash
# Create alert when memory usage exceeds 80%
gcloud alpha monitoring policies create \
  --policy-from-file=- << 'EOF'
{
  "displayName": "Redis High Memory Usage",
  "conditions": [
    {
      "displayName": "Memory usage above 80%",
      "conditionThreshold": {
        "filter": "resource.type = \"redis_instance\" AND metric.type = \"redis.googleapis.com/stats/memory/usage_ratio\"",
        "comparison": "COMPARISON_GT",
        "thresholdValue": 0.8,
        "duration": "300s",
        "aggregations": [
          {
            "alignmentPeriod": "60s",
            "perSeriesAligner": "ALIGN_MEAN"
          }
        ]
      }
    }
  ],
  "alertStrategy": {
    "autoClose": "1800s"
  },
  "notificationChannels": [
    "projects/my-project/notificationChannels/YOUR_CHANNEL_ID"
  ]
}
EOF
```

### High Replication Lag Alert

```bash
gcloud alpha monitoring policies create \
  --policy-from-file=- << 'EOF'
{
  "displayName": "Redis High Replication Lag",
  "conditions": [
    {
      "displayName": "Replication lag above 10 seconds",
      "conditionThreshold": {
        "filter": "resource.type = \"redis_instance\" AND metric.type = \"redis.googleapis.com/replication/master/slaves/lag\"",
        "comparison": "COMPARISON_GT",
        "thresholdValue": 10,
        "duration": "60s",
        "aggregations": [
          {
            "alignmentPeriod": "60s",
            "perSeriesAligner": "ALIGN_MAX"
          }
        ]
      }
    }
  ],
  "notificationChannels": [
    "projects/my-project/notificationChannels/YOUR_CHANNEL_ID"
  ]
}
EOF
```

## Terraform: Alerting Policies

```hcl
# Notification channel for alerts
resource "google_monitoring_notification_channel" "email" {
  display_name = "Redis Ops Email"
  type         = "email"
  labels = {
    email_address = "ops@example.com"
  }
}

# High memory alert
resource "google_monitoring_alert_policy" "redis_memory" {
  display_name = "Redis Memory Usage High"
  combiner     = "OR"

  conditions {
    display_name = "Memory usage > 80%"
    condition_threshold {
      filter          = "resource.type = \"redis_instance\" AND metric.type = \"redis.googleapis.com/stats/memory/usage_ratio\""
      comparison      = "COMPARISON_GT"
      threshold_value = 0.8
      duration        = "300s"

      aggregations {
        alignment_period   = "60s"
        per_series_aligner = "ALIGN_MEAN"
      }
    }
  }

  notification_channels = [google_monitoring_notification_channel.email.name]

  documentation {
    content = "Redis memory usage is above 80%. Consider scaling up the instance or reviewing key TTLs."
  }
}

# Replication lag alert
resource "google_monitoring_alert_policy" "redis_replication_lag" {
  display_name = "Redis Replication Lag High"
  combiner     = "OR"

  conditions {
    display_name = "Replication lag > 10s"
    condition_threshold {
      filter          = "resource.type = \"redis_instance\" AND metric.type = \"redis.googleapis.com/replication/master/slaves/lag\""
      comparison      = "COMPARISON_GT"
      threshold_value = 10
      duration        = "60s"

      aggregations {
        alignment_period   = "60s"
        per_series_aligner = "ALIGN_MAX"
      }
    }
  }

  notification_channels = [google_monitoring_notification_channel.email.name]
}
```

## Python Monitoring Client

```python
from google.cloud import monitoring_v3
from datetime import datetime, timedelta

def get_redis_metrics(project_id, instance_id, metric_type, hours=1):
    client = monitoring_v3.MetricServiceClient()
    project_name = f"projects/{project_id}"

    interval = monitoring_v3.TimeInterval({
        "end_time": {"seconds": int(datetime.utcnow().timestamp())},
        "start_time": {"seconds": int((datetime.utcnow() - timedelta(hours=hours)).timestamp())}
    })

    results = client.list_time_series(
        request={
            "name": project_name,
            "filter": (
                f'metric.type="{metric_type}" '
                f'AND resource.labels.instance_id="{instance_id}"'
            ),
            "interval": interval,
            "view": monitoring_v3.ListTimeSeriesRequest.TimeSeriesView.FULL,
        }
    )

    data_points = []
    for ts in results:
        for point in ts.points:
            data_points.append({
                "time": datetime.fromtimestamp(point.interval.end_time.seconds),
                "value": point.value.double_value or point.value.int64_value
            })

    return sorted(data_points, key=lambda x: x["time"])

def get_cache_hit_ratio(project_id, instance_id, hours=1):
    hits = get_redis_metrics(
        project_id, instance_id,
        "redis.googleapis.com/stats/keyspace_hits",
        hours
    )
    misses = get_redis_metrics(
        project_id, instance_id,
        "redis.googleapis.com/stats/keyspace_misses",
        hours
    )

    total_hits = sum(p["value"] for p in hits)
    total_misses = sum(p["value"] for p in misses)
    total = total_hits + total_misses

    if total == 0:
        return 0.0
    return (total_hits / total) * 100

ratio = get_cache_hit_ratio("my-project", "my-redis-instance")
print(f"Cache hit ratio: {ratio:.1f}%")
```

## Creating a Monitoring Dashboard

```bash
# Create a dashboard via the Monitoring API
cat > redis-dashboard.json << 'EOF'
{
  "displayName": "Memorystore Redis Overview",
  "mosaicLayout": {
    "columns": 12,
    "tiles": [
      {
        "width": 6,
        "height": 4,
        "widget": {
          "title": "Memory Usage Ratio",
          "xyChart": {
            "dataSets": [{
              "timeSeriesQuery": {
                "timeSeriesFilter": {
                  "filter": "resource.type=\"redis_instance\" metric.type=\"redis.googleapis.com/stats/memory/usage_ratio\"",
                  "aggregation": {
                    "alignmentPeriod": "60s",
                    "perSeriesAligner": "ALIGN_MEAN"
                  }
                }
              }
            }]
          }
        }
      },
      {
        "width": 6,
        "height": 4,
        "xPos": 6,
        "widget": {
          "title": "Cache Hit Rate",
          "xyChart": {
            "dataSets": [
              {
                "timeSeriesQuery": {
                  "timeSeriesFilter": {
                    "filter": "resource.type=\"redis_instance\" metric.type=\"redis.googleapis.com/stats/keyspace_hits\"",
                    "aggregation": {"alignmentPeriod": "60s", "perSeriesAligner": "ALIGN_RATE"}
                  }
                }
              }
            ]
          }
        }
      }
    ]
  }
}
EOF

gcloud monitoring dashboards create --config-from-file=redis-dashboard.json \
  --project=my-project
```

## Summary

Google Cloud Memorystore metrics are automatically available in Cloud Monitoring under `redis.googleapis.com`. The most critical metrics to alert on are `memory/usage_ratio` (alert above 80%), `replication/master/slaves/lag` (alert above 10 seconds), and command latency via `commands/usec_per_call`. Use Terraform to codify alerting policies with appropriate notification channels, and build dashboards combining memory, hit rate, and connection metrics for at-a-glance health visibility.
