# How to Deploy Services with cephadm Service Specifications

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, cephadm, Service, Deployment, YAML, Orchestration

Description: Use cephadm service specification YAML files to declaratively deploy and configure Ceph services including OSDs, monitors, RGW, and MDS.

---

## What Are Service Specifications?

cephadm service specifications are YAML files that define the desired state of Ceph services. Instead of running individual `ceph orch daemon add` commands, you declare the full configuration in YAML and apply it. This is the recommended approach for production deployments.

## Applying a Service Spec

```bash
ceph orch apply -i service-spec.yaml
```

You can apply multiple specs at once from a single file using YAML document separators (`---`).

## Monitor Service Spec

Deploy monitors on specific hosts:

```yaml
service_type: mon
placement:
  hosts:
    - node1
    - node2
    - node3
```

Or by label:

```yaml
service_type: mon
placement:
  label: mon
```

## OSD Service Spec

Deploy OSDs using all available unformatted drives:

```yaml
service_type: osd
service_id: default
placement:
  host_pattern: '*'
spec:
  data_devices:
    all: true
```

Target specific drives:

```yaml
service_type: osd
service_id: ssd-tier
placement:
  hosts:
    - node1
    - node2
spec:
  data_devices:
    rotational: 0
  db_devices:
    model: Samsung 980 Pro
```

## RGW Service Spec

Deploy a Ceph Object Gateway:

```yaml
service_type: rgw
service_id: objectstore
placement:
  count: 2
  label: rgw
spec:
  rgw_realm: production
  rgw_zone: us-east-1
  rgw_frontend_port: 7480
```

## MDS Service Spec

Deploy Ceph Filesystem MDS daemons:

```yaml
service_type: mds
service_id: cephfs
placement:
  count: 2
  label: mds
```

## Verifying Applied Specs

```bash
# List all service specs
ceph orch ls

# Get the spec for a specific service
ceph orch ls --service-name rgw.objectstore --export

# Check daemon status
ceph orch ps
```

## Exporting Existing Configuration

Export the current running config as YAML for backup or reuse:

```bash
ceph orch ls --export > cluster-specs.yaml
```

## Removing a Service

```bash
ceph orch rm rgw.objectstore
```

## Summary

cephadm service specifications provide a declarative, YAML-based approach to deploying Ceph services. Define placement using host names, labels, or patterns, and apply with `ceph orch apply -i`. Export existing configs with `ceph orch ls --export` for GitOps workflows or disaster recovery documentation.
