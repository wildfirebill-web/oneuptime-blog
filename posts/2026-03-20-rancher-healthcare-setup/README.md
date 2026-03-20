# How to Set Up Rancher for Healthcare Environments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: rancher, healthcare, hipaa, kubernetes, compliance

Description: A step-by-step guide to configuring Rancher for HIPAA-compliant healthcare environments, covering security hardening, access control, and audit logging.

## Overview

Healthcare organizations running Kubernetes must comply with HIPAA (Health Insurance Portability and Accountability Act) and often HITRUST CSF. Rancher provides a robust platform for managing healthcare workloads with the security controls needed to protect Protected Health Information (PHI). This guide walks through the key configurations required for a HIPAA-ready Rancher deployment.

## Prerequisites

- Rancher v2.7+ installed on a hardened OS (RHEL, Rocky Linux, or Ubuntu 22.04)
- RKE2 clusters using the CIS hardened profile
- A dedicated network segment for healthcare workloads
- PKI infrastructure for TLS certificates
- Backup and DR solution

## Step 1: Use RKE2 with CIS Profile

All healthcare clusters should use RKE2 with the CIS hardening profile:

```yaml
# /etc/rancher/rke2/config.yaml on each node
profile: cis-1.23
selinux: true
secrets-encryption: true
audit-policy-file: /etc/rancher/rke2/audit-policy.yaml
pod-security-admission-config-file: /etc/rancher/rke2/psa.yaml
```

## Step 2: Configure Audit Logging

HIPAA requires audit trails for all access to PHI systems:

```yaml
# /etc/rancher/rke2/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log all access to Secrets (may contain PHI credentials)
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["secrets"]
  # Log all namespace operations
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["namespaces"]
  # Log all Pod executions (access to containers)
  - level: Request
    verbs: ["create"]
    resources:
      - group: ""
        resources: ["pods/exec", "pods/portforward"]
  # Log all other changes at Metadata level
  - level: Metadata
    omitStages:
      - RequestReceived
```

## Step 3: Configure RBAC for Least Privilege

```yaml
# Create a project in Rancher for healthcare workloads
# Assign users to the minimal required roles
# Example: Read-only role for auditors
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: healthcare-auditor
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch"]
```

## Step 4: Enable Network Policies

Isolate PHI workloads with strict network policies:

```yaml
# Deny all ingress/egress by default for healthcare namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: healthcare-prod
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
# Allow only specific communications
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ehr-to-database
  namespace: healthcare-prod
spec:
  podSelector:
    matchLabels:
      app: ehr-application
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
```

## Step 5: Configure etcd Encryption at Rest

```yaml
# /etc/rancher/rke2/config.yaml
secrets-encryption: true
# RKE2 handles key rotation via:
# rke2 secrets-encrypt rotate-keys
```

## Step 6: Set Up NeuVector for Runtime Security

Install NeuVector for HIPAA-required runtime monitoring:

```bash
helm repo add neuvector https://neuvector.github.io/neuvector-helm/
helm install neuvector neuvector/core \
  --namespace neuvector \
  --create-namespace \
  --set controller.replicas=3 \
  --set manager.env.ssl=true
```

Configure NeuVector to alert on PHI-related anomalies:
- Container process violations
- Unexpected network connections from PHI namespaces
- File system access violations

## Step 7: Configure Longhorn for Encrypted Storage

```yaml
# StorageClass with volume encryption for PHI data
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-encrypted
provisioner: driver.longhorn.io
parameters:
  numberOfReplicas: "3"
  encrypted: "true"
  # Encryption key stored in a Kubernetes Secret
  csi.storage.k8s.io/node-publish-secret-name: longhorn-crypto
  csi.storage.k8s.io/node-publish-secret-namespace: longhorn-system
```

## Step 8: Backup and Disaster Recovery

HIPAA requires data backup and recovery procedures:

```yaml
# Longhorn recurring backup to S3
apiVersion: longhorn.io/v1beta2
kind: RecurringJob
metadata:
  name: hipaa-backup
  namespace: longhorn-system
spec:
  cron: "0 1 * * *"    # Daily at 1 AM
  task: backup
  groups:
    - healthcare
  retain: 30             # 30-day retention
  concurrency: 1
```

## Step 9: Enable Rancher Audit Logging

```yaml
# In Rancher global settings (rancher-config ConfigMap)
# Enable audit log level 2 (logs user, action, resource)
audit-level: "2"
audit-log-path: "/var/log/rancher/audit.log"
audit-log-maxage: "90"   # 90-day retention for HIPAA
```

## Step 10: Identity Provider Integration

Configure Rancher with your hospital's Active Directory or SAML identity provider:

```
Rancher UI → Global Settings → Auth Configuration → Active Directory
- Server: ldaps://ad.hospital.internal:636
- Service Account: rancher@hospital.internal
- Search Base: OU=Staff,DC=hospital,DC=internal
- Required Group: CN=Kubernetes-Users,OU=Groups,DC=hospital,DC=internal
```

## Compliance Checklist

- [ ] RKE2 CIS hardened profile enabled
- [ ] Audit logging configured and retained 90+ days
- [ ] etcd encryption at rest enabled
- [ ] Network policies isolating PHI namespaces
- [ ] Volume encryption for PHI persistent data
- [ ] RBAC least privilege applied
- [ ] NeuVector runtime security monitoring active
- [ ] Regular backup schedule (daily, 30-day retention minimum)
- [ ] Active Directory / SAML integration for user management
- [ ] TLS certificates from trusted CA throughout

## Conclusion

Setting up Rancher for healthcare environments requires careful attention to HIPAA requirements including audit logging, data encryption, access controls, and backup procedures. RKE2 with CIS hardening, NeuVector runtime security, and Longhorn encrypted storage provide a comprehensive HIPAA-ready stack. Always consult with your compliance team and conduct regular HIPAA risk assessments to maintain compliance as your environment evolves.
