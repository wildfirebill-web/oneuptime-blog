# How to Use cephadm Adoption to Convert Existing Clusters

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, cephadm, Migration, Kubernetes

Description: Learn how to use cephadm adoption to convert an existing Ceph cluster to cephadm management without data loss or downtime.

---

## What is cephadm Adoption?

cephadm adoption is the process of converting a Ceph cluster that was deployed by another method (such as ceph-ansible or manual deployment) into one managed by cephadm. This gives you access to modern Ceph orchestration, easier upgrades, and containerized daemons.

## Prerequisites

Before starting adoption, verify your cluster health:

```bash
ceph status
ceph health detail
```

Ensure all OSDs are up and the cluster is in HEALTH_OK or HEALTH_WARN state only. Resolve any critical issues before proceeding.

Install cephadm on the primary monitor node:

```bash
curl --silent --remote-name --location https://github.com/ceph/ceph/raw/quincy/src/cephadm/cephadm
chmod +x cephadm
mv cephadm /usr/local/bin/
```

## Running the Adoption

Bootstrap cephadm on the existing cluster using the `--skip-pull` flag to avoid re-pulling images:

```bash
cephadm adopt \
  --style legacy \
  --name mon.$(hostname) \
  --legacy-dir /var/lib/ceph
```

For clusters running in containers already, use the `--style` flag to specify the existing deployment method:

```bash
cephadm adopt \
  --style ceph-volume \
  --name osd.0
```

After adoption of the first monitor, cephadm will generate a new SSH key and distribute it to other hosts:

```bash
ceph cephadm get-pub-key > ~/ceph.pub
ssh-copy-id -f -i ~/ceph.pub root@mon2
ssh-copy-id -f -i ~/ceph.pub root@mon3
```

## Adding Hosts to cephadm

Register all cluster hosts with cephadm:

```bash
ceph orch host add mon1 192.168.1.10
ceph orch host add mon2 192.168.1.11
ceph orch host add mon3 192.168.1.12
ceph orch host add osd1 192.168.1.20
```

Verify host connectivity:

```bash
ceph orch host ls
```

## Adopting Remaining Daemons

List all daemons not yet managed by cephadm:

```bash
ceph orch ps --status not-managed
```

Adopt monitors on remaining nodes:

```bash
cephadm adopt --style legacy --name mon.mon2 --legacy-dir /var/lib/ceph
cephadm adopt --style legacy --name mon.mon3 --legacy-dir /var/lib/ceph
```

Adopt MGR daemons:

```bash
cephadm adopt --style legacy --name mgr.$(hostname)
```

## Redeploy OSDs Under cephadm

Once monitors and managers are adopted, redeploy OSDs:

```bash
ceph orch apply osd --all-available-devices
```

Or use a specific drive spec:

```yaml
service_type: osd
service_id: osd-spec
placement:
  host_pattern: "osd*"
spec:
  data_devices:
    all: true
```

Apply the spec:

```bash
ceph orch apply -i osd-spec.yaml
```

## Verify Adoption Success

Check that all services are now managed:

```bash
ceph orch ps
ceph orch ls
ceph status
```

All daemons should show `managed` in the orchestrator output.

## Summary

cephadm adoption converts existing Ceph clusters to orchestrator management without data loss by migrating daemons one at a time. After adoption, you gain access to streamlined upgrades, automated daemon management, and containerized deployment. Always verify cluster health at each stage to ensure a safe migration.
