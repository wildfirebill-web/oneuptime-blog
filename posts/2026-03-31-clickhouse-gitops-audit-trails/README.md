# How to Build GitOps Audit Trails in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, GitOps, Audit, DevOps, Kubernetes

Description: Store and query GitOps pipeline events in ClickHouse to build immutable audit trails for deployments, config changes, and drift detections across Kubernetes clusters.

## Introduction

GitOps tools like ArgoCD, Flux, and Helm Operator continuously reconcile the desired state stored in Git with the actual state running in Kubernetes. Every sync, deployment, config change, and drift detection event is an audit record that your security and platform teams need to query. These events accumulate fast across hundreds of clusters and thousands of applications.

ClickHouse is a natural fit for audit trail storage because it is append-only by design, handles high write throughput, and executes time-range and aggregation queries instantly. You can answer questions like "which engineer triggered the most failed deployments last month?" or "which clusters drifted from Git state in the past 24 hours?" in milliseconds.

## Schema Design

### Deployment Events

```sql
CREATE TABLE gitops_deployment_events
(
    event_id         UUID DEFAULT generateUUIDv4(),
    occurred_at      DateTime64(3),
    cluster_id       LowCardinality(String),
    namespace        LowCardinality(String),
    application      String,
    environment      LowCardinality(String),  -- dev, staging, production
    event_type       LowCardinality(String),  -- sync, rollback, delete, prune
    status           LowCardinality(String),  -- success, failed, progressing, degraded
    triggered_by     String,                  -- user or automated
    trigger_source   LowCardinality(String),  -- manual, webhook, schedule, policy
    git_repo         LowCardinality(String),
    git_branch       LowCardinality(String),
    git_commit       FixedString(40),
    previous_commit  FixedString(40),
    duration_ms      UInt32,
    error_message    String
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(occurred_at)
ORDER BY (cluster_id, environment, application, occurred_at);
```

### Configuration Change Events

```sql
CREATE TABLE gitops_config_changes
(
    change_id       UUID DEFAULT generateUUIDv4(),
    occurred_at     DateTime64(3),
    cluster_id      LowCardinality(String),
    namespace       LowCardinality(String),
    resource_kind   LowCardinality(String),  -- Deployment, ConfigMap, Secret, Service
    resource_name   String,
    operation       LowCardinality(String),  -- create, update, delete, patch
    changed_by      String,
    git_commit      FixedString(40),
    diff_summary    String,
    is_drift        UInt8                    -- 1 if change was not from Git
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(occurred_at)
ORDER BY (cluster_id, namespace, resource_kind, occurred_at);
```

### Drift Detection Events

```sql
CREATE TABLE gitops_drift_events
(
    drift_id            UUID DEFAULT generateUUIDv4(),
    detected_at         DateTime64(3),
    resolved_at         Nullable(DateTime64(3)),
    cluster_id          LowCardinality(String),
    namespace           LowCardinality(String),
    application         String,
    resource_kind       LowCardinality(String),
    resource_name       String,
    drift_type          LowCardinality(String),  -- missing, extra, modified
    expected_value      String,
    actual_value        String,
    auto_remediated     UInt8
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(detected_at)
ORDER BY (cluster_id, application, detected_at);
```

### Policy Violation Events

```sql
CREATE TABLE gitops_policy_violations
(
    violation_id    UUID DEFAULT generateUUIDv4(),
    occurred_at     DateTime64(3),
    cluster_id      LowCardinality(String),
    namespace       LowCardinality(String),
    policy_name     LowCardinality(String),
    severity        LowCardinality(String),  -- low, medium, high, critical
    resource_kind   LowCardinality(String),
    resource_name   String,
    git_commit      FixedString(40),
    blocked         UInt8,
    message         String
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(occurred_at)
ORDER BY (cluster_id, severity, occurred_at);
```

## Inserting Events

```sql
INSERT INTO gitops_deployment_events
    (occurred_at, cluster_id, namespace, application, environment, event_type, status,
     triggered_by, trigger_source, git_repo, git_branch, git_commit, previous_commit, duration_ms)
VALUES
    (now64(), 'prod-us-east-1', 'payments', 'payment-api', 'production', 'sync',     'success', 'alice@corp.com', 'webhook',  'github.com/corp/backend', 'main', repeat('a', 40), repeat('b', 40), 4200),
    (now64(), 'prod-us-east-1', 'auth',     'auth-service', 'production', 'sync',    'failed',  'bob@corp.com',   'manual',   'github.com/corp/backend', 'main', repeat('c', 40), repeat('d', 40), 8100),
    (now64(), 'staging-eu',     'frontend', 'web-app',      'staging',    'rollback','success', 'ci-bot',         'schedule', 'github.com/corp/frontend','main', repeat('e', 40), repeat('f', 40), 1800);
```

## Deployment Success Rate by Cluster

```sql
SELECT
    cluster_id,
    environment,
    count()                                     AS total_deployments,
    countIf(status = 'success')                 AS successful,
    countIf(status = 'failed')                  AS failed,
    round(countIf(status = 'success') / count() * 100, 2) AS success_rate_pct
FROM gitops_deployment_events
WHERE occurred_at >= now() - INTERVAL 30 DAY
GROUP BY cluster_id, environment
ORDER BY environment, success_rate_pct;
```

## Mean Time to Deploy (P50 and P95)

```sql
SELECT
    cluster_id,
    application,
    count()                                     AS deploys,
    median(duration_ms)                         AS p50_ms,
    quantile(0.95)(duration_ms)                 AS p95_ms,
    avg(duration_ms)                            AS avg_ms
FROM gitops_deployment_events
WHERE status = 'success'
  AND occurred_at >= now() - INTERVAL 30 DAY
GROUP BY cluster_id, application
ORDER BY p95_ms DESC
LIMIT 20;
```

## Change Frequency Per Application

```sql
SELECT
    application,
    environment,
    toDate(occurred_at)  AS day,
    count()              AS deployments
FROM gitops_deployment_events
WHERE occurred_at >= now() - INTERVAL 90 DAY
GROUP BY application, environment, day
ORDER BY application, day;
```

## Who Triggered the Most Failed Deployments

```sql
SELECT
    triggered_by,
    count()                        AS total_triggers,
    countIf(status = 'failed')     AS failures,
    round(countIf(status = 'failed') / count() * 100, 2) AS failure_rate_pct
FROM gitops_deployment_events
WHERE occurred_at >= now() - INTERVAL 30 DAY
  AND trigger_source = 'manual'
GROUP BY triggered_by
ORDER BY failures DESC
LIMIT 20;
```

## Drift Detection Summary

```sql
SELECT
    cluster_id,
    count()                             AS total_drifts,
    countIf(drift_type = 'modified')    AS modified,
    countIf(drift_type = 'missing')     AS missing,
    countIf(drift_type = 'extra')       AS extra,
    countIf(auto_remediated = 1)        AS auto_fixed,
    countIf(resolved_at IS NULL)        AS still_open
FROM gitops_drift_events
WHERE detected_at >= now() - INTERVAL 7 DAY
GROUP BY cluster_id
ORDER BY total_drifts DESC;
```

## Out-of-Band Changes (Cluster Mutations Not From Git)

Any config change where `is_drift = 1` means someone or something modified a Kubernetes resource directly instead of through Git:

```sql
SELECT
    cluster_id,
    namespace,
    resource_kind,
    resource_name,
    changed_by,
    occurred_at,
    diff_summary
FROM gitops_config_changes
WHERE is_drift = 1
  AND occurred_at >= now() - INTERVAL 24 HOUR
ORDER BY occurred_at DESC
LIMIT 100;
```

## Policy Violation Trend

```sql
SELECT
    toStartOfDay(occurred_at) AS day,
    policy_name,
    severity,
    count()                   AS violations,
    countIf(blocked = 1)      AS blocked_count
FROM gitops_policy_violations
WHERE occurred_at >= now() - INTERVAL 30 DAY
GROUP BY day, policy_name, severity
ORDER BY day DESC, violations DESC;
```

## Commit Impact Analysis

Which Git commits caused the most failures or drifts?

```sql
SELECT
    git_commit,
    git_repo,
    count()                            AS deploys,
    countIf(status = 'failed')         AS failures,
    round(countIf(status = 'failed') / count() * 100, 2) AS failure_rate_pct,
    any(error_message)                 AS sample_error
FROM gitops_deployment_events
WHERE occurred_at >= now() - INTERVAL 14 DAY
GROUP BY git_commit, git_repo
HAVING failures > 0
ORDER BY failure_rate_pct DESC
LIMIT 20;
```

## Rollback Frequency Per Environment

High rollback frequency in production indicates poor pre-production gating:

```sql
SELECT
    environment,
    toStartOfWeek(occurred_at) AS week,
    count()                    AS rollbacks
FROM gitops_deployment_events
WHERE event_type = 'rollback'
  AND occurred_at >= now() - INTERVAL 90 DAY
GROUP BY environment, week
ORDER BY environment, week;
```

## Immutability and Retention

Because audit trails must be tamper-evident, never allow deletes or updates to audit tables. Use TTL only for archival, not deletion:

```sql
ALTER TABLE gitops_deployment_events
    MODIFY TTL occurred_at + INTERVAL 7 YEAR TO DISK 'cold_storage';
```

Combine this with ClickHouse role-based access control to prevent anyone from issuing `ALTER TABLE ... DELETE` on the audit tables:

```sql
CREATE ROLE audit_reader;
GRANT SELECT ON gitops_deployment_events TO audit_reader;
GRANT SELECT ON gitops_config_changes TO audit_reader;
GRANT SELECT ON gitops_drift_events TO audit_reader;
GRANT SELECT ON gitops_policy_violations TO audit_reader;
```

## Conclusion

ClickHouse gives platform and security teams a queryable, long-lived audit trail for GitOps pipelines. Its append-only write model aligns naturally with audit log requirements, and its fast aggregations make it practical to answer compliance questions across years of deployment history in seconds. By combining structured schemas, materialized views, and access controls, you can build an audit system that satisfies both operational and regulatory needs.

**Related Reading:**

- [What Is ClickHouse MergeTree and How It Works](https://oneuptime.com/blog/post/2026-03-31-clickhouse-mergetree-explained/view)
- [What Is TTL in ClickHouse and How Data Lifecycle Works](https://oneuptime.com/blog/post/2026-03-31-clickhouse-ttl-data-lifecycle/view)
- [What Is the Difference Between Mutations and Lightweight Deletes in ClickHouse](https://oneuptime.com/blog/post/2026-03-31-clickhouse-mutations-vs-lightweight-deletes/view)
