# How to Update Edge Agents with Automatic Rollback

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Edge Agent, Updates, Rollback, Business Edition

Description: Use Portainer Business Edition's edge update schedules with automatic rollback to safely update edge agents across remote environments.

## Introduction

Updating edge agents across dozens or hundreds of remote devices requires a safe, automated process. Portainer Business Edition provides edge update schedules with automatic rollback — if an update fails, the agent automatically reverts to the previous working version.

## Prerequisites

- Portainer Business Edition
- Edge environments configured and connected
- Sufficient disk space on edge devices for rollback image

## Creating an Update Schedule

### Via Web UI

1. Go to **Edge Compute** → **Edge Update Schedules**
2. Click **Add schedule**
3. Configure:

```
Name:          Quarterly Agent Update
Update type:   Portainer agent
Version:       2.21.0 (or latest)
Schedule:      Specific date/time or immediate

Target:
  ● All edge environments
  ○ Selected environments
  ○ Edge groups

Rollback:
  ✓ Automatically rollback on failure
  Failure detection: 5 minutes (timeout)
```

### Via API

```bash
TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Create update schedule
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/edge/update_schedules \
  -d '{
    "name": "Q1-2026-Agent-Update",
    "type": 1,
    "version": "2.21.0",
    "scheduledTime": "2026-01-15T02:00:00Z",
    "environments": [],
    "groupIds": [1, 2],
    "rollback": true,
    "rollbackTimeout": 300
  }'
```

## Monitoring Update Progress

```bash
# Get update schedule status
SCHEDULE_ID=1

curl -s \
  -H "Authorization: Bearer $TOKEN" \
  "https://portainer.example.com/api/edge/update_schedules/${SCHEDULE_ID}" \
  | python3 -c "
import sys, json
s = json.load(sys.stdin)
print(f'Schedule: {s[\"Name\"]}')
print(f'Status: {s.get(\"Status\")}')
print(f'Progress: {s.get(\"SuccessCount\",0)} succeeded, {s.get(\"FailedCount\",0)} failed')
"

# List all schedules
curl -s \
  -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/edge/update_schedules \
  | python3 -m json.tool
```

## How Rollback Works

1. Before update, current agent version is snapshotted
2. New agent version is deployed
3. Agent checks in with Portainer within the timeout window
4. If check-in succeeds → update marked successful
5. If check-in fails within timeout → rollback triggered
6. Old agent version is restored from snapshot

## Staged Rollout

For large fleets, deploy updates in stages:

```bash
# Stage 1: Update 10% of devices (canary)
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/edge/update_schedules \
  -d '{
    "name": "Canary-Update",
    "environments": [1, 2, 3],
    "rollback": true
  }'

# Wait 24 hours, then Stage 2: Update all remaining
```

## Conclusion

Edge update schedules with automatic rollback eliminate the risk of bricking remote devices during updates. The health-check based rollback ensures devices revert to a working state if the update causes issues. For large fleets, use staged rollouts to validate updates on a small set before full deployment.
