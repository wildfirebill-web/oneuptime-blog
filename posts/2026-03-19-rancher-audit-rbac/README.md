# How to Audit RBAC Permissions in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, RBAC, Permissions, Security, Role

Description: A step-by-step guide to auditing RBAC permissions in Rancher to identify excessive access, orphaned bindings, and security risks.

Regular RBAC audits are critical for maintaining the security of your Rancher-managed Kubernetes clusters. Over time, role bindings accumulate, users change teams, and permissions drift from their intended state. This guide provides a systematic approach to auditing RBAC at every level in Rancher.

## Prerequisites

- Rancher v2.7+ with administrator access
- kubectl access to the Rancher management cluster
- jq installed for JSON processing
- Basic familiarity with Rancher's role model

## Step 1: Audit Global Role Assignments

Start at the broadest scope. List all users with global roles:

```bash
# List all global role bindings

kubectl get globalrolebindings -o json | \
  jq -r '.items[] | "\(.userName // .groupPrincipalName)\t\(.globalRoleName)\t\(.metadata.creationTimestamp)"' | \
  column -t -s $'\t'
```

Focus on high-privilege roles:

```bash
# Find all administrators
kubectl get globalrolebindings -o json | \
  jq -r '.items[] | select(.globalRoleName == "admin") | "\(.userName) - Created: \(.metadata.creationTimestamp)"'

# Count admins
echo "Total administrators:"
kubectl get globalrolebindings -o json | \
  jq '[.items[] | select(.globalRoleName == "admin")] | length'
```

Flag any accounts that should not have administrator access and document them for remediation.

## Step 2: Audit Cluster Role Assignments

Check who has access to each cluster:

```bash
#!/bin/bash
# audit-cluster-roles.sh

echo "=== Cluster Role Audit ==="
echo ""

for cluster in $(kubectl get clusters.management.cattle.io -o jsonpath='{.items[*].metadata.name}'); do
  display=$(kubectl get clusters.management.cattle.io $cluster -o jsonpath='{.spec.displayName}')
  echo "--- Cluster: $display ($cluster) ---"

  # List cluster role bindings
  kubectl get clusterroletemplatebindings -n $cluster -o json | \
    jq -r '.items[] | "\(.userName // .groupPrincipalName // "unknown")\t\(.roleTemplateId)\t\(.metadata.creationTimestamp)"' | \
    column -t -s $'\t'

  # Count cluster owners
  owners=$(kubectl get clusterroletemplatebindings -n $cluster -o json | \
    jq '[.items[] | select(.roleTemplateId == "cluster-owner")] | length')
  echo "  Cluster Owners: $owners"
  echo ""
done
```

## Step 3: Audit Project Role Assignments

```bash
#!/bin/bash
# audit-project-roles.sh

echo "=== Project Role Audit ==="
echo ""

for project in $(kubectl get projects.management.cattle.io --all-namespaces -o json | jq -r '.items[] | "\(.metadata.namespace)/\(.metadata.name)/\(.spec.displayName)"'); do
  cluster=$(echo $project | cut -d'/' -f1)
  proj_id=$(echo $project | cut -d'/' -f2)
  proj_name=$(echo $project | cut -d'/' -f3)

  bindings=$(kubectl get projectroletemplatebindings -n $cluster -o json | \
    jq -r "[.items[] | select(.projectName == \"$cluster:$proj_id\")] | length")

  if [ "$bindings" -gt 0 ]; then
    echo "--- Project: $proj_name ---"
    kubectl get projectroletemplatebindings -n $cluster -o json | \
      jq -r ".items[] | select(.projectName == \"$cluster:$proj_id\") | \"\(.userName // .groupPrincipalName // \"unknown\")\t\(.roleTemplateId)\"" | \
      column -t -s $'\t'
    echo ""
  fi
done
```

## Step 4: Identify Orphaned Role Bindings

Orphaned bindings belong to users who no longer exist or groups that have been removed:

```bash
# Find bindings with no matching user
kubectl get clusterroletemplatebindings --all-namespaces -o json | \
  jq -r '.items[] | select(.userName != null and .userName != "") | .userName' | sort -u | while read user; do
  exists=$(kubectl get users.management.cattle.io $user 2>/dev/null)
  if [ -z "$exists" ]; then
    echo "Orphaned user: $user"
    kubectl get clusterroletemplatebindings --all-namespaces -o json | \
      jq -r ".items[] | select(.userName == \"$user\") | \"  Binding: \(.metadata.name) in \(.metadata.namespace)\""
  fi
done
```

## Step 5: Check for Overly Permissive Custom Roles

Review custom role templates for wildcard permissions or broad access:

```bash
# Find roles with wildcard permissions
kubectl get roletemplates -o json | \
  jq -r '.items[] | select(.rules[]? | .apiGroups[]? == "*" or .resources[]? == "*" or .verbs[]? == "*") | "WARNING - \(.metadata.name) (\(.displayName)): has wildcard permissions"'

# Find roles that grant delete on all resources
kubectl get roletemplates -o json | \
  jq -r '.items[] | select(.rules[]? | .verbs[]? == "delete" and .resources[]? == "*") | "WARNING - \(.metadata.name): grants delete on all resources"'
```

## Step 6: Audit Kubernetes-Level RBAC

Rancher's RBAC layer sits on top of Kubernetes RBAC. Audit the underlying bindings too:

```bash
# List all ClusterRoleBindings in a downstream cluster
kubectl get clusterrolebindings -o json | \
  jq -r '.items[] | select(.metadata.name | startswith("cattle-") | not) | "\(.metadata.name)\t\(.roleRef.name)\t\(.subjects[]? | "\(.kind):\(.namespace // "cluster")/\(.name)")"' | \
  column -t -s $'\t'

# Find bindings to the cluster-admin role
kubectl get clusterrolebindings -o json | \
  jq -r '.items[] | select(.roleRef.name == "cluster-admin") | "\(.metadata.name): \(.subjects[]? | "\(.kind)/\(.name)")"'
```

## Step 7: Enable and Review Audit Logs

Enable Rancher's audit logging to track RBAC-related actions:

1. Go to **Global Settings** in the Rancher UI.
2. Set `audit-level` to `2` for detailed logging.
3. Configure the audit log destination.

Via Helm values during Rancher installation:

```yaml
auditLog:
  level: 2
  destination: sidecar
  hostPath: /var/log/rancher/audit/
  maxAge: 30
  maxBackup: 10
  maxSize: 100
```

Review audit logs for RBAC changes:

```bash
# Search for role binding creation events
grep "clusterroletemplatebindings" /var/log/rancher/audit/audit.log | \
  jq -r 'select(.verb == "create" or .verb == "delete") | "\(.requestReceivedTimestamp) \(.verb) \(.objectRef.name) by \(.user.username)"'
```

## Step 8: Generate a Comprehensive Audit Report

Create a complete audit report script:

```bash
#!/bin/bash
# rbac-audit-report.sh

REPORT_FILE="rbac-audit-$(date +%Y%m%d).txt"

{
  echo "RBAC Audit Report - $(date)"
  echo "=================================="
  echo ""

  echo "1. GLOBAL ROLE SUMMARY"
  echo "----------------------"
  kubectl get globalrolebindings -o json | \
    jq -r '.items | group_by(.globalRoleName) | .[] | "\(.[0].globalRoleName): \(length) bindings"'
  echo ""

  echo "2. ADMINISTRATOR ACCOUNTS"
  echo "-------------------------"
  kubectl get globalrolebindings -o json | \
    jq -r '.items[] | select(.globalRoleName == "admin") | "  \(.userName)"'
  echo ""

  echo "3. CLUSTER ACCESS SUMMARY"
  echo "-------------------------"
  for cluster in $(kubectl get clusters.management.cattle.io -o jsonpath='{.items[*].metadata.name}'); do
    display=$(kubectl get clusters.management.cattle.io $cluster -o jsonpath='{.spec.displayName}')
    count=$(kubectl get clusterroletemplatebindings -n $cluster -o json | jq '.items | length')
    owners=$(kubectl get clusterroletemplatebindings -n $cluster -o json | \
      jq '[.items[] | select(.roleTemplateId == "cluster-owner")] | length')
    echo "  $display: $count total bindings, $owners owners"
  done
  echo ""

  echo "4. CUSTOM ROLES WITH ELEVATED PERMISSIONS"
  echo "------------------------------------------"
  kubectl get roletemplates -o json | \
    jq -r '.items[] | select(.builtin != true) | select(.rules[]? | .verbs[]? == "*" or .resources[]? == "*") | "  WARNING: \(.displayName) has wildcard permissions"'
  echo ""

  echo "5. ROLES GRANTING SECRET ACCESS"
  echo "-------------------------------"
  kubectl get roletemplates -o json | \
    jq -r '.items[] | select(.rules[]? | .resources[]? == "secrets") | "  \(.displayName) (\(.metadata.name)): grants access to secrets"'

} > "$REPORT_FILE"

echo "Report saved to $REPORT_FILE"
```

## Step 9: Automate Periodic Audits

Schedule the audit script to run regularly:

```bash
# Add to crontab - runs monthly on the 1st at 6 AM
0 6 1 * * /opt/scripts/rbac-audit-report.sh && mail -s "Monthly RBAC Audit" security-team@example.com < /opt/reports/rbac-audit-*.txt
```

## Step 10: Remediate Findings

For each finding in your audit, take appropriate action:

```bash
# Remove an orphaned cluster role binding
kubectl delete clusterroletemplatebinding <binding-name> -n <cluster-id>

# Downgrade a user from cluster-owner to cluster-member
# First remove the owner binding
kubectl delete clusterroletemplatebinding <owner-binding> -n <cluster-id>

# Then create a member binding (use the Rancher UI or API for this)
```

## Best Practices

- **Schedule quarterly audits**: At minimum, audit RBAC every quarter.
- **Automate detection**: Script the identification of anomalies and run it on a schedule.
- **Track changes**: Use Rancher audit logs to monitor RBAC changes in real time.
- **Document exceptions**: When elevated access is justified, document why and set a review date.
- **Integrate with offboarding**: Remove access as part of your employee offboarding process.
- **Version control roles**: Store custom role templates in Git and apply them through CI/CD.

## Conclusion

Auditing RBAC in Rancher is an ongoing process that combines automated tooling with human review. By systematically checking global, cluster, and project role assignments, identifying orphaned bindings, and flagging overly permissive roles, you maintain a secure environment. Automate as much as possible and integrate RBAC audits into your regular security review cycle.
