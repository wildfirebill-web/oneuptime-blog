# How to Migrate NeuVector Policies Between Clusters - Policy Migration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: NeuVector, Policy Migration, Kubernetes, Security, Multi-Cluster, SUSE Rancher

Description: Learn how to export NeuVector security policies from one cluster and import them into another cluster for consistent security posture across environments and disaster recovery.

---

Migrating NeuVector policies between clusters ensures consistent security posture across development, staging, and production environments. NeuVector provides built-in export and import capabilities for its policy configurations.

---

## What Can Be Migrated

- Network rules (ingress/egress rules per group)
- Process profile rules
- File access rules
- Admission control rules
- Group definitions
- Response rules
- DLP rules
- WAF rules

---

## Step 1: Export Policies via NeuVector UI

1. In the NeuVector UI, go to **Policy**
2. For each policy type, use the export (download) button
3. Policies are exported as JSON files

Or export via the NeuVector REST API:

```bash
# Get NeuVector access token

TOKEN=$(curl -sk -X POST \
  https://neuvector.example.com/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin"}' \
  | jq -r '.token.token')

# Export all network rules
curl -sk \
  -H "X-Auth-Token: $TOKEN" \
  https://neuvector.example.com/v1/policy/rule?scope=local \
  > network-rules-export.json

# Export groups
curl -sk \
  -H "X-Auth-Token: $TOKEN" \
  https://neuvector.example.com/v1/group?scope=local \
  > groups-export.json
```

---

## Step 2: Export via NeuVector CLI

```bash
# Install the NeuVector CLI (neuvector-ctl)
# Export complete policy package
kubectl exec -n neuvector \
  $(kubectl get pod -n neuvector -l app=neuvector-manager-pod -o name) \
  -- /usr/local/bin/cli export -o /tmp/nv-policy-export.conf

# Copy the export file
kubectl cp neuvector/$(kubectl get pod -n neuvector -l app=neuvector-manager-pod -o jsonpath='{.items[0].metadata.name}'):/tmp/nv-policy-export.conf ./nv-policy-export.conf
```

---

## Step 3: Review and Sanitize the Export

Before importing to another cluster, review the export for cluster-specific references:

```bash
# Check for IP addresses that may be cluster-specific
grep -E '"ip_range":|"cidr":' nv-policy-export.conf

# Check for namespace references that differ between clusters
grep '"namespace":' nv-policy-export.conf | sort | uniq
```

---

## Step 4: Import Policies into the Target Cluster

```bash
# Get token for the target cluster
TARGET_TOKEN=$(curl -sk -X POST \
  https://neuvector-target.example.com/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin"}' \
  | jq -r '.token.token')

# Import network rules
curl -sk -X PUT \
  -H "X-Auth-Token: $TARGET_TOKEN" \
  -H "Content-Type: application/json" \
  https://neuvector-target.example.com/v1/policy/rule?scope=local \
  -d @network-rules-export.json
```

---

## Step 5: Verify Migration

After import, verify policies are active in the target cluster:

```bash
# Check rule count
curl -sk \
  -H "X-Auth-Token: $TARGET_TOKEN" \
  https://neuvector-target.example.com/v1/policy/rule \
  | jq '.rules | length'

# Compare with source
curl -sk \
  -H "X-Auth-Token: $TOKEN" \
  https://neuvector.example.com/v1/policy/rule \
  | jq '.rules | length'
```

---

## Best Practices

- Always migrate policies to a staging cluster first and validate before applying to production.
- Store exported policy files in Git for version history and change tracking.
- Use NeuVector's **multi-cluster federation** for ongoing policy synchronization rather than manual migrations.
- Export policies before every NeuVector upgrade as a backup.
