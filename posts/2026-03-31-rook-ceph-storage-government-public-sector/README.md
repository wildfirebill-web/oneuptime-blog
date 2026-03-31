# How to Configure Ceph Storage for Government and Public Sector

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Government, Compliance, FedRAMP, FISMA, Encryption, Air Gap

Description: Configure Rook/Ceph storage for government and public sector workloads with FedRAMP, FISMA, and NIST compliance, air-gapped deployments, and data sovereignty requirements.

---

## Government Storage Requirements

Government IT storage must address:
- **Compliance**: FedRAMP, FISMA, NIST SP 800-53, CMMC
- **Air-gapped deployment**: No internet connectivity
- **Data sovereignty**: Data must remain within jurisdiction
- **FIPS 140-2 encryption**: Cryptographic modules must be FIPS-validated
- **Access control**: Role-based access with CAC/PIV integration
- **Audit logging**: Complete, tamper-proof audit trails

## Deploying Rook in an Air-Gapped Environment

In air-gapped environments, images must be mirrored to an internal registry:

```bash
# Mirror Rook and Ceph images to internal registry
INTERNAL_REGISTRY="registry.gov.internal"
ROOK_VERSION="v1.16.0"
CEPH_VERSION="v19.2.0"

# Tag and push Rook operator
docker pull rook/ceph:${ROOK_VERSION}
docker tag rook/ceph:${ROOK_VERSION} ${INTERNAL_REGISTRY}/rook/ceph:${ROOK_VERSION}
docker push ${INTERNAL_REGISTRY}/rook/ceph:${ROOK_VERSION}

# Update operator deployment to use internal registry
helm install rook-ceph rook-ceph/rook-ceph \
  --set image.repository=${INTERNAL_REGISTRY}/rook/ceph \
  --set csi.rbdPluginImage=${INTERNAL_REGISTRY}/cephcsi/cephcsi:v3.12.0
```

## FIPS-Compliant Encryption Configuration

Configure Vault with FIPS 140-2 compliant keys for KMS integration:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rook-ceph-operator-config
  namespace: rook-ceph
data:
  CSI_ENABLE_ENCRYPTION: "true"
  KMS_PROVIDER: "vault"
  VAULT_ADDR: "https://vault.gov.internal:8200"
  VAULT_AUTH_METHOD: "kubernetes"
  VAULT_AUTH_MOUNT_PATH: "kubernetes"
  VAULT_ROLE: "rook-ceph-role"
  VAULT_BACKEND_PATH: "secret/rook-ceph"
  VAULT_TLS_CA_CERT: "/etc/vault-tls/ca.crt"
```

## Network Encryption (Msgr2)

Enable encryption for all Ceph daemon communication:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  network:
    connections:
      encryption:
        enabled: true
      requireMsgr2: true
```

## RGW for FedRAMP Data Storage

Configure RGW with TLS and bucket policies for data classification:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: gov-store
  namespace: rook-ceph
spec:
  gateway:
    securePort: 443
    sslCertificateRef: gov-tls-cert
    instances: 3
```

Apply bucket policies to enforce classification boundaries:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": "arn:aws:s3:::classified-data/*",
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

## Audit Logging Configuration

Enable comprehensive RGW access logging:

```bash
# Enable per-bucket logging
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  radosgw-admin bucket logging enable \
  --bucket classified-data \
  --target-bucket audit-logs \
  --target-prefix classified-data/

# Verify logging configuration
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  radosgw-admin bucket logging get --bucket classified-data
```

## RBAC for Multi-Agency Access

Use RGW IAM to create per-agency users with scoped permissions:

```bash
# Create an agency-specific user
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  radosgw-admin user create \
  --uid agency-dod-user \
  --display-name "DoD Agency User" \
  --max-buckets 50

# Apply agency-scoped quota
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  radosgw-admin quota set \
  --quota-scope user \
  --uid agency-dod-user \
  --max-size 10T
```

## Summary

Rook/Ceph meets government storage requirements through air-gapped image mirroring, FIPS-compliant encryption via Vault KMS, Msgr2 wire encryption, S3 bucket policies for data classification enforcement, and comprehensive RGW access logging for audit trails. The platform's open-source nature also allows organizations to perform their own security reviews, which is increasingly required for FedRAMP authorization.
