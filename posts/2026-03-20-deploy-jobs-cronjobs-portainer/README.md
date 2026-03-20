# How to Deploy Kubernetes Jobs and CronJobs with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Jobs, CronJobs, Docker, Automation

Description: Learn how to create and manage Kubernetes Jobs and CronJobs through the Portainer interface for batch workloads and scheduled tasks.

---

Kubernetes Jobs run a task to completion and exit, while CronJobs schedule Jobs on a cron schedule. Portainer provides a UI for creating and monitoring both, making it easy to manage batch workloads without writing raw YAML.

---

## Create a Job via Portainer UI

1. In Portainer, navigate to the Kubernetes cluster.
2. Go to **Applications** → **Add application**.
3. Switch to **Advanced mode** for YAML input.
4. Paste the Job manifest:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-migration
  namespace: default
spec:
  completions: 1
  parallelism: 1
  backoffLimit: 3
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: migration
          image: myorg/db-migrator:latest
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: url
```

---

## Create a CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-backup
  namespace: default
spec:
  schedule: "0 2 * * *"  # 2 AM daily
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: myorg/backup-tool:latest
              env:
                - name: S3_BUCKET
                  value: "my-backups"
              resources:
                requests:
                  memory: "128Mi"
                  cpu: "250m"
```

---

## Monitor Jobs in Portainer

1. Navigate to **Applications** → filter by **Job** type.
2. Click on a Job to see Pod status, logs, and completion time.
3. For CronJobs: view last schedule time, active jobs, and history.

---

## Trigger a CronJob Manually

```bash
# Create a Job from a CronJob template

kubectl create job --from=cronjob/nightly-backup manual-backup-$(date +%Y%m%d)
```

---

## Common CronJob Pitfalls

- `concurrencyPolicy: Forbid` - prevents overlapping runs
- `startingDeadlineSeconds` - if the scheduler misses a window, skip rather than backfill
- `successfulJobsHistoryLimit: 3` - keeps logs accessible without indefinite accumulation

---

## Summary

Deploy Kubernetes Jobs for one-off batch tasks and CronJobs for recurring scheduled work via Portainer's YAML editor. Set `backoffLimit` to control retry behavior, `concurrencyPolicy` to prevent overlapping executions, and history limits to control how many completed jobs are retained. Monitor job completion and logs directly in Portainer's Applications view.
