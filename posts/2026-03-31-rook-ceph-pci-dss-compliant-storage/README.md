# How to Set Up Ceph for PCI-DSS Compliant Storage

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, PCI-DSS, Compliance, Security, Encryption

Description: Configure Ceph storage to meet PCI-DSS requirements including strong encryption, network segmentation, access control, and audit logging for cardholder data.

---

PCI-DSS (Payment Card Industry Data Security Standard) imposes strict requirements on systems that store, process, or transmit cardholder data. Ceph can be hardened to meet these requirements across multiple control areas.

## Requirement 3: Protect Stored Cardholder Data

PCI-DSS Requirement 3 mandates encryption of cardholder data at rest. Enable dmcrypt encryption on all OSDs that will store PCI data:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  storage:
    storageClassDeviceSets:
      - name: pci-osds
        count: 6
        encrypted: true
        volumeClaimTemplates:
          - metadata:
              name: data
            spec:
              resources:
                requests:
                  storage: 1Ti
              storageClassName: fast-local
              volumeMode: Block
              accessModes:
                - ReadWriteOnce
```

## Requirement 4: Encrypt Transmission of Cardholder Data

Force encrypted communication between all Ceph components:

```bash
ceph config set global ms_cluster_mode secure
ceph config set global ms_service_mode secure
ceph config set global ms_client_mode secure
ceph config set global ms_mon_cluster_mode secure
```

For S3/RGW access, enforce TLS-only:

```bash
# Disable plaintext HTTP port
ceph config set client.rgw rgw_frontends "beast ssl_port=443 ssl_certificate=/etc/ceph/rgw.crt ssl_private_key=/etc/ceph/rgw.key"
```

## Requirement 7: Restrict Access to System Components

Implement least-privilege access for all Ceph users:

```bash
# Create a restricted application user
radosgw-admin user create \
  --uid=pci-app \
  --display-name="PCI Application" \
  --max-buckets=10

# Restrict to specific bucket via bucket policy
cat > pci-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam:::user/pci-app"},
    "Action": ["s3:GetObject", "s3:PutObject"],
    "Resource": "arn:aws:s3:::cardholder-data/*"
  }]
}
EOF

aws s3api put-bucket-policy \
  --bucket cardholder-data \
  --policy file://pci-policy.json \
  --endpoint-url https://rgw.example.com
```

## Requirement 10: Log and Monitor All Access

Enable comprehensive audit logging:

```bash
ceph config set global rgw_enable_ops_log true
ceph config set global rgw_enable_usage_log true
ceph config set global rgw_log_http_headers "HTTP_X_FORWARDED_FOR,HTTP_USER_AGENT"

# Forward logs to syslog for SIEM ingestion
ceph config set global log_to_syslog true
ceph config set global err_to_syslog true
```

## Requirement 11: Test Security Controls

Run regular vulnerability scans against RGW endpoints using standard tools:

```bash
# Check TLS configuration
openssl s_client -connect rgw.example.com:443 -tls1_2

# Verify encryption cipher strength
nmap --script ssl-enum-ciphers -p 443 rgw.example.com

# Confirm no weak protocols are enabled
sslscan rgw.example.com:443
```

## Network Segmentation

Isolate Ceph from the cardholder data environment using Kubernetes NetworkPolicies:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ceph-pci-isolation
  namespace: rook-ceph
spec:
  podSelector:
    matchLabels:
      app: rook-ceph-rgw
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              pci-zone: "true"
      ports:
        - protocol: TCP
          port: 443
```

## Summary

PCI-DSS compliance with Ceph requires enabling OSD-level dmcrypt encryption, enforcing TLS for all communication, implementing strict bucket policies and user access controls, enabling comprehensive audit logging, and isolating the storage cluster via network segmentation. Together these controls address the key PCI-DSS requirements applicable to storage infrastructure.
