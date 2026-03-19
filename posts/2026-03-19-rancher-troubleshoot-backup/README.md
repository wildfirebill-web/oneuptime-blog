# How to Troubleshoot Backup and Restore Issues in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Backup, Restore

Description: Learn how to diagnose and fix common backup and restore problems in Rancher including operator failures, storage issues, and restore errors.

Backups are only valuable if they work when you need them. When Rancher backup or restore operations fail, quick diagnosis is essential. This guide covers the most common issues you will encounter with the Rancher Backup Operator and etcd snapshots, along with step-by-step solutions.

## Common Backup Operator Issues

### Issue 1: Backup Operator Pod Not Starting

Check the pod status:

```bash
kubectl get pods -n cattle-resources-system
```

If the pod is in `CrashLoopBackOff` or `Error` state, check the logs:

```bash
kubectl logs -n cattle-resources-system -l app.kubernetes.io/name=rancher-backup --previous
```

Common causes and solutions:

**Insufficient RBAC permissions**: Reinstall the CRDs:

```bash
helm upgrade rancher-backup-crd rancher-charts/rancher-backup-crd \
  -n cattle-resources-system
```

**Image pull errors**: Check if the image is accessible:

```bash
kubectl describe pod -n cattle-resources-system -l app.kubernetes.io/name=rancher-backup
```

Look for `ImagePullBackOff` events and verify your registry connectivity.

### Issue 2: Backup Stuck in Progress

If a backup resource shows status `InProgress` for an extended period:

```bash
kubectl get backups.resources.cattle.io -o wide
```

Check the operator logs for errors:

```bash
kubectl logs -n cattle-resources-system -l app.kubernetes.io/name=rancher-backup -f
```

If the backup is stuck, delete it and retry:

```bash
kubectl delete backups.resources.cattle.io stuck-backup-name
kubectl apply -f backup.yaml
```

### Issue 3: S3 Authentication Failures

Symptoms: Backup fails with access denied or authentication errors.

Verify the credentials secret:

```bash
kubectl get secret s3-creds -n cattle-resources-system -o jsonpath='{.data.accessKey}' | base64 -d
kubectl get secret s3-creds -n cattle-resources-system -o jsonpath='{.data.secretKey}' | base64 -d
```

Test S3 connectivity from within the cluster:

```bash
kubectl run s3-test --rm -it --image=amazon/aws-cli --restart=Never -- \
  s3 ls s3://rancher-backups/ \
  --endpoint-url https://s3.amazonaws.com \
  --region us-east-1
```

Recreate the secret if the credentials are wrong:

```bash
kubectl delete secret s3-creds -n cattle-resources-system
kubectl create secret generic s3-creds \
  -n cattle-resources-system \
  --from-literal=accessKey=CORRECT_ACCESS_KEY \
  --from-literal=secretKey=CORRECT_SECRET_KEY
```

### Issue 4: Backup File Too Large

Large Rancher installations may produce backup files that exceed storage limits or take too long.

Check the backup size:

```bash
kubectl exec -n cattle-resources-system deploy/rancher-backup -- \
  ls -lh /var/lib/backups/
```

Solutions:
- Increase the PersistentVolume size for the operator.
- Use external storage (S3) which has no practical size limit.
- Check for excessive custom resources that may be inflating the backup.

### Issue 5: Scheduled Backups Not Running

If scheduled backups are not firing:

```bash
kubectl describe backups.resources.cattle.io scheduled-backup-name
```

Check that the cron expression is valid. Common mistakes include wrong field order or unsupported expressions.

Verify the operator is running and not in a restart loop:

```bash
kubectl get pods -n cattle-resources-system -w
```

## Common Restore Issues

### Issue 6: Restore Fails with Version Mismatch

The Rancher version on the target must match the version when the backup was taken. Check the backup metadata:

```bash
kubectl logs -n cattle-resources-system -l app.kubernetes.io/name=rancher-backup | grep version
```

Install the matching Rancher version before attempting the restore.

### Issue 7: Restore Fails with Encryption Error

If you see decryption errors during restore:

```bash
kubectl get restores.resources.cattle.io restore-name -o yaml
```

Verify the encryption secret exists and contains the correct key:

```bash
kubectl get secret rancher-backup-encryption -n cattle-resources-system
```

Ensure you are using the exact same encryption configuration that was used during backup. Key names and values must match exactly.

### Issue 8: Resources Conflict During Restore

If the restore encounters existing resources:

```bash
kubectl logs -n cattle-resources-system -l app.kubernetes.io/name=rancher-backup | grep conflict
```

The operator attempts to update existing resources, but some conflicts may require manual intervention. You can delete the conflicting resources before retrying:

```bash
kubectl delete clusters.management.cattle.io conflicting-cluster-name
```

### Issue 9: Rancher Not Starting After Restore

If Rancher pods fail to start after a restore:

```bash
kubectl get pods -n cattle-system
kubectl describe pod -n cattle-system -l app=rancher
kubectl logs -n cattle-system -l app=rancher
```

Common causes:
- **Certificate mismatch**: The hostname or certificates may differ between source and target. Reinstall Rancher with the correct hostname.
- **Webhook issues**: Delete webhook configurations that may be blocking:

```bash
kubectl delete mutatingwebhookconfigurations rancher.cattle.io
kubectl delete validatingwebhookconfigurations rancher.cattle.io
```

Then restart Rancher:

```bash
kubectl rollout restart deployment rancher -n cattle-system
```

## etcd Snapshot Issues

### Issue 10: etcd Snapshot Fails

Check the etcd logs on the control plane node:

```bash
journalctl -u rke2-server | grep etcd
```

Common causes:
- **Disk full**: Check available disk space on control plane nodes:

```bash
df -h /var/lib/rancher/
```

- **etcd database too large**: Check the database size:

```bash
kubectl -n kube-system exec -it etcd-NODE -- \
  etcdctl endpoint status --write-out=table \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

If the database is too large, consider defragmenting:

```bash
kubectl -n kube-system exec -it etcd-NODE -- \
  etcdctl defrag \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### Issue 11: etcd Restore Breaks Cluster

If a cluster is unhealthy after an etcd restore:

1. Verify etcd member health:

```bash
kubectl -n kube-system exec -it etcd-NODE -- \
  etcdctl member list \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

2. Check for stale members and remove them.
3. On multi-node clusters, ensure only one node was restored first, then others re-joined.

## Diagnostic Commands Summary

```bash
# Backup Operator status
kubectl get pods -n cattle-resources-system
kubectl logs -n cattle-resources-system -l app.kubernetes.io/name=rancher-backup

# List backups and restores
kubectl get backups.resources.cattle.io
kubectl get restores.resources.cattle.io

# Detailed backup status
kubectl describe backups.resources.cattle.io BACKUP_NAME

# etcd health
kubectl get cs
kubectl -n kube-system exec -it etcd-NODE -- etcdctl endpoint health

# Rancher health
kubectl get pods -n cattle-system
kubectl logs -n cattle-system -l app=rancher --tail=50
```

## Conclusion

Most backup and restore issues in Rancher come down to authentication problems, version mismatches, storage constraints, or configuration errors. By systematically checking operator logs, verifying credentials, and understanding the expected behavior, you can resolve these issues quickly. Always test your backup and restore procedures in a non-production environment to catch issues before they matter.
