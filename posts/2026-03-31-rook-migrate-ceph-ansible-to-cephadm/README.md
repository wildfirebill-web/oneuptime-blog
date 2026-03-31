# How to Migrate from ceph-ansible to cephadm

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Migration, cephadm, Ansible

Description: Learn how to migrate a Ceph cluster managed by ceph-ansible to cephadm for containerized daemon management and simplified day-2 operations.

---

ceph-ansible was widely used to automate Ceph deployments. As cephadm has become the standard orchestration tool, migrating from ceph-ansible provides modern lifecycle management, automatic upgrades, and better Rook compatibility.

## Preparation

Before migrating, ensure:
- Cluster is healthy (HEALTH_OK)
- All PGs are active+clean
- You have root SSH access to all nodes

```bash
ceph -s
ceph health detail
ceph pg stat
```

Document the cluster:

```bash
ceph osd tree > /backup/osd-tree.txt
ceph mon dump > /backup/mon-dump.txt
cat /etc/ceph/ceph.conf > /backup/ceph.conf.backup
```

## Understanding cephadm Adoption

cephadm's adopt workflow converts individually running Ceph daemons (installed as packages by ceph-ansible) into container-based daemons managed by cephadm. Data directories are preserved.

## Step 1 - Install cephadm on All Nodes

```bash
# Using Ansible to distribute cephadm
ansible all -m shell -a "
  curl -fsSL https://raw.githubusercontent.com/ceph/ceph/reef/src/cephadm/cephadm \
    -o /usr/local/bin/cephadm && \
  chmod +x /usr/local/bin/cephadm
" -i inventory
```

## Step 2 - Pull the Container Image

```bash
ansible all -m shell -a "
  cephadm pull
" -i inventory
```

## Step 3 - Adopt the First Monitor

```bash
# On the first mon node
cephadm adopt --style legacy --name mon.a
systemctl status ceph-mon@a  # Should show inactive (replaced by container)
cephadm ls | grep mon
```

## Step 4 - Enable the cephadm Orchestrator

```bash
ceph mgr module enable cephadm
ceph orch set backend cephadm
ceph orch status
```

## Step 5 - Adopt Remaining Mons and MGRs

```bash
# For each remaining mon
cephadm adopt --style legacy --name mon.b
cephadm adopt --style legacy --name mon.c

# For each mgr
cephadm adopt --style legacy --name mgr.a
```

## Step 6 - Adopt OSD Nodes

Use an Ansible playbook to adopt all OSDs in parallel:

```yaml
- name: Adopt Ceph OSDs
  hosts: osds
  become: true
  tasks:
  - name: Get OSD IDs on this host
    command: ceph-volume lvm list --format json
    register: osd_list

  - name: Adopt each OSD
    command: cephadm adopt --style legacy --name osd.{{ item }}
    loop: "{{ osd_list.stdout | from_json | dict2items | map(attribute='key') | list }}"
```

## Step 7 - Verify Complete Migration

```bash
ceph orch ps
ceph -s
cephadm ls --no-detail
```

All daemons should show as container-based.

## Disabling ceph-ansible

Remove ceph-ansible configuration management by removing the cron/Ansible Tower jobs that previously ran the playbooks. The cluster is now fully managed by cephadm.

```bash
# Remove old ansible playbook cron if present
ansible all -m cron -a "name='ceph-ansible-run' state=absent" -i inventory
```

## Summary

Migrating from ceph-ansible to cephadm uses the adopt command to convert package-based daemons to containerized ones while preserving all data and configuration. Running the migration host by host with health checks between each step ensures cluster stability. Once complete, cephadm's orchestration replaces Ansible's role in day-2 operations.
