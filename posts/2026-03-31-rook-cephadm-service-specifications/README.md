# How to Deploy Ceph Services with cephadm Service Specifications

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, cephadm, ServiceSpec, Storage, Deployment

Description: Learn how to use cephadm service specification files to declaratively deploy and configure all Ceph service types including mon, osd, mgr, rgw, and mds.

---

## What Are Service Specifications

Service specifications (service specs) are YAML files that describe the desired state of a Ceph service: which hosts to place it on, how many instances, and service-specific configuration. The cephadm orchestrator continuously reconciles the actual cluster state with the desired state defined in specs.

Using specs enables GitOps-style cluster management - check your spec files into version control and apply them to reproduce or modify your Ceph deployment.

## Monitor Spec

```yaml
# mon.yaml
service_type: mon
placement:
  hosts:
    - node1
    - node2
    - node3
```

```bash
ceph orch apply -i mon.yaml
```

## Manager Spec

```yaml
# mgr.yaml
service_type: mgr
placement:
  hosts:
    - node1
    - node2
  count: 2
```

## OSD Spec

OSD specs support multiple ways to select storage devices:

```yaml
# osd.yaml
service_type: osd
service_id: default
placement:
  host_pattern: "node[2-4]"
data_devices:
  all: true
db_devices:
  model: Samsung_SSD_970
wal_devices:
  rotational: 0
```

Specify device paths explicitly:

```yaml
service_type: osd
service_id: manual-osds
placement:
  hosts:
    - node2
    - node3
data_devices:
  paths:
    - /dev/sdb
    - /dev/sdc
```

## RGW (Object Gateway) Spec

```yaml
# rgw.yaml
service_type: rgw
service_id: production
placement:
  hosts:
    - node1
    - node2
  count: 2
spec:
  rgw_realm: myrealm
  rgw_zone: us-east-1
  rgw_frontend_port: 8080
  ssl: true
  rgw_frontend_ssl_certificate: |
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----
```

## MDS (CephFS Metadata Server) Spec

```yaml
# mds.yaml
service_type: mds
service_id: myfilesystem
placement:
  hosts:
    - node1
    - node2
  count: 2
```

## NFS Spec

```yaml
# nfs.yaml
service_type: nfs
service_id: mynfs
placement:
  hosts:
    - node1
spec:
  pool: nfs-ganesha
  namespace: mynfs
```

## Applying Multiple Specs at Once

Combine specs into a single file using YAML document separators:

```yaml
# full-cluster.yaml
---
service_type: mon
placement:
  hosts: [node1, node2, node3]
---
service_type: mgr
placement:
  count: 2
---
service_type: osd
service_id: all-drives
placement:
  host_pattern: "node*"
data_devices:
  all: true
---
service_type: rgw
service_id: default
placement:
  count: 2
  hosts: [node1, node2]
```

```bash
# Apply all specs at once
ceph orch apply -i full-cluster.yaml

# Dry-run to preview changes
ceph orch apply -i full-cluster.yaml --dry-run
```

## Exporting Current Service Specs

Export existing services as spec files for version control:

```bash
# Export all service specs
ceph orch ls --export > current-specs.yaml

# Export a specific service
ceph orch ls --export mon > mon-spec.yaml
```

## Removing a Service

```bash
# Remove a service (stops and removes all daemons)
ceph orch rm rgw.production
```

## Summary

cephadm service specifications provide a declarative, file-based interface for deploying every Ceph service type. Specs define placement through host lists, count targets, or host patterns, and service-specific configuration like RGW realm/zone settings or OSD device selectors. Using spec files enables reproducible deployments, GitOps workflows, and easy cluster reconfiguration. The `--dry-run` flag and `--export` option make it safe to preview changes and maintain current cluster state in version control.
