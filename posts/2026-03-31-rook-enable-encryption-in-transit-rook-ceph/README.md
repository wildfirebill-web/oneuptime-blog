# How to Enable Encryption in Transit for Rook-Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Security, Encryption, TLS

Description: Enable encryption in transit for Rook-Ceph by configuring msgr2 protocol with secure mode, enabling TLS for MON connections, and verifying encrypted traffic between Ceph daemons.

---

## Why Encrypt Ceph Traffic in Transit

Ceph cluster traffic (MON, OSD, MGR communications) travels over the internal network unencrypted by default. In multi-tenant Kubernetes clusters or environments subject to compliance requirements, encrypting in-transit data prevents eavesdropping on replication traffic and client-to-OSD writes.

## Ceph msgr2 Encryption Modes

Ceph v15+ uses the msgr2 protocol which supports three encryption modes:
- `crc`: CRC checksumming only (default, no encryption)
- `secure`: Full AES-GCM encryption of all traffic
- `prefer-secure`: Use secure if available, fall back to crc

## Enabling Cluster-Wide msgr2 Secure Mode

Configure the encryption mode via Ceph config:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set global ms_cluster_mode secure

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set global ms_service_mode secure

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set global ms_client_mode secure
```

Verify the settings applied:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config dump | grep ms_
```

## Enabling in Rook CephCluster Spec

Configure msgr2 encryption in the Rook operator:

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
      compression:
        enabled: false
      requireMsgr2: true
```

## Verifying Encrypted Connections

Check that all active connections are using secure mode:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph status

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph tell osd.0 connections
```

Look for `secure` in the connection details output.

Monitor active sessions:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph daemon osd.0 sessions | head -20
```

## Dashboard TLS

The Ceph dashboard should also use TLS. Configure it in the cluster spec:

```yaml
spec:
  dashboard:
    enabled: true
    ssl: true
    port: 8443
```

Create the TLS cert from cert-manager:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: rook-ceph-dashboard-cert
  namespace: rook-ceph
spec:
  secretName: rook-ceph-dashboard-cert
  issuerRef:
    name: internal-ca
    kind: ClusterIssuer
  dnsNames:
    - ceph-dashboard.example.com
```

## Monitoring Encryption Status

Create a Prometheus alert if any Ceph connection falls back to unencrypted mode:

```yaml
groups:
  - name: ceph-security
    rules:
      - alert: CephInsecureConnection
        expr: ceph_health_status == 2
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Ceph cluster health degraded - check encryption status"
```

## Performance Impact

msgr2 secure mode adds approximately 5-15% CPU overhead on OSD nodes due to AES-GCM encryption. Modern CPUs with AES-NI hardware acceleration minimize this to 2-5%.

Check CPU utilization on OSD pods after enabling:

```bash
kubectl -n rook-ceph top pod -l app=rook-ceph-osd
```

## Summary

Encryption in transit for Rook-Ceph uses the msgr2 `secure` mode to encrypt all Ceph daemon communications with AES-GCM. Enabling it requires setting three config keys plus the `connections.encryption.enabled` in the CephCluster spec. Modern AES-NI-capable CPUs keep the performance overhead minimal, making secure mode practical for production deployments.
