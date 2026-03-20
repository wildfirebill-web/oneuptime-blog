# How to Back Up NeuVector Configuration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: NeuVector, Backup, Configuration, Security, Kubernetes, Disaster Recovery, SUSE Rancher

Description: Learn how to back up and restore NeuVector security policies, network rules, and configuration using the NeuVector REST API and automated export scripts.

---

NeuVector stores its security policy configuration (network rules, process profiles, response rules, and admission control policies) in its internal data store. Regular backups ensure you can restore your security posture after a cluster failure or accidental policy deletion.

---

## What NeuVector Backup Includes

| Component | Description |
|---|---|
| Network rules | Allow/deny rules between services |
| Process profiles | Allowed processes per container group |
| File access rules | Allowed file system operations |
| Response rules | Automated actions triggered by events |
| Admission control rules | Kubernetes admission policies |
| Custom compliance checks | User-defined compliance rules |

---

## Step 1: Access the NeuVector REST API

NeuVector provides a REST API for configuration export:

```bash
# Get the NeuVector controller service

kubectl get svc -n cattle-neuvector-system | grep controller

# Port-forward to the REST API
kubectl port-forward -n cattle-neuvector-system \
  svc/neuvector-svc-controller 10443:10443 &

# Get an auth token
TOKEN=$(curl -sk -X POST \
  https://localhost:10443/v1/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin"}' \
  | jq -r '.token.token')

echo $TOKEN
```

---

## Step 2: Export the Full Configuration

```bash
# Export all policies and configuration
curl -sk \
  -H "X-Auth-Token: $TOKEN" \
  https://localhost:10443/v1/file/config \
  -o neuvector-backup-$(date +%Y%m%d).json

# Verify the export
jq '.policies | keys' neuvector-backup-$(date +%Y%m%d).json
```

---

## Step 3: Export Individual Policy Types

```bash
# Export network rules only
curl -sk \
  -H "X-Auth-Token: $TOKEN" \
  "https://localhost:10443/v1/policy/rule" \
  | jq . > neuvector-network-rules-$(date +%Y%m%d).json

# Export admission control rules
curl -sk \
  -H "X-Auth-Token: $TOKEN" \
  "https://localhost:10443/v1/admission/rules" \
  | jq . > neuvector-admission-rules-$(date +%Y%m%d).json

# Export groups (service definitions)
curl -sk \
  -H "X-Auth-Token: $TOKEN" \
  "https://localhost:10443/v1/group" \
  | jq . > neuvector-groups-$(date +%Y%m%d).json
```

---

## Step 4: Automate Backups with a CronJob

```yaml
# neuvector-backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: neuvector-backup
  namespace: cattle-neuvector-system
spec:
  schedule: "0 2 * * *"    # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: backup
              image: curlimages/curl:latest
              command:
                - sh
                - -c
                - |
                  # Authenticate
                  TOKEN=$(curl -sk -X POST \
                    https://neuvector-svc-controller:10443/v1/auth \
                    -H "Content-Type: application/json" \
                    -d "{\"username\":\"$NEUVECTOR_USER\",\"password\":\"$NEUVECTOR_PASSWORD\"}" \
                    | grep -o '"token":"[^"]*"' | cut -d'"' -f4)

                  # Export config
                  curl -sk \
                    -H "X-Auth-Token: $TOKEN" \
                    https://neuvector-svc-controller:10443/v1/file/config \
                    -o /backup/neuvector-$(date +%Y%m%d-%H%M%S).json

                  # Logout
                  curl -sk -X DELETE \
                    -H "X-Auth-Token: $TOKEN" \
                    https://neuvector-svc-controller:10443/v1/auth
              env:
                - name: NEUVECTOR_USER
                  valueFrom:
                    secretKeyRef:
                      name: neuvector-backup-secret
                      key: username
                - name: NEUVECTOR_PASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: neuvector-backup-secret
                      key: password
              volumeMounts:
                - name: backup-storage
                  mountPath: /backup
          volumes:
            - name: backup-storage
              persistentVolumeClaim:
                claimName: neuvector-backup-pvc
          restartPolicy: OnFailure
```

---

## Step 5: Restore from Backup

```bash
# Import configuration from a backup file
curl -sk -X POST \
  -H "X-Auth-Token: $TOKEN" \
  -H "Content-Type: application/json" \
  https://localhost:10443/v1/file/config \
  -d @neuvector-backup-20260320.json

# Verify the restore
curl -sk \
  -H "X-Auth-Token: $TOKEN" \
  "https://localhost:10443/v1/policy/rule" \
  | jq '.rules | length'
```

---

## Step 6: Migrate Configuration to Another Cluster

```bash
# Export from source cluster
kubectl port-forward -n cattle-neuvector-system \
  svc/neuvector-svc-controller 10443:10443 &
# (authenticate and export as above)

# Import to destination cluster
kubectl port-forward -n cattle-neuvector-system \
  svc/neuvector-svc-controller 10444:10443 &

# Get token for destination
DEST_TOKEN=$(curl -sk -X POST \
  https://localhost:10444/v1/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin"}' \
  | jq -r '.token.token')

# Import
curl -sk -X POST \
  -H "X-Auth-Token: $DEST_TOKEN" \
  -H "Content-Type: application/json" \
  https://localhost:10444/v1/file/config \
  -d @neuvector-backup-20260320.json
```

---

## Best Practices

- Back up NeuVector configuration before every policy update or NeuVector upgrade - this gives you a rollback point if the new policy causes unexpected behavior.
- Store backup files in an S3 bucket with versioning enabled - this allows you to restore any historical policy state.
- Test your restore procedure in a staging NeuVector instance regularly to confirm backups are valid and the import process works correctly.
