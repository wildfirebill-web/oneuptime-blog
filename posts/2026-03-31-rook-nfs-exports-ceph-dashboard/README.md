# How to Create NFS Exports via Ceph Dashboard in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, NFS, Dashboard, Kubernetes

Description: Learn how to create and manage NFS exports using the Ceph Dashboard in a Rook cluster, with step-by-step configuration guidance.

---

## The Ceph Dashboard as an NFS Management Interface

The Ceph Dashboard provides a web-based GUI for managing NFS-Ganesha exports alongside all other Ceph resources. For teams that prefer a visual interface over CLI commands, the dashboard makes it easy to create, view, and modify NFS exports with form-based inputs. It communicates with the Ceph Manager NFS module directly, so changes made in the dashboard are identical to those made via the CLI.

## Accessing the Ceph Dashboard

First, ensure the dashboard is enabled and accessible. Check its service in Rook:

```bash
kubectl -n rook-ceph get service rook-ceph-mgr-dashboard
```

If you need external access, expose it via port-forward:

```bash
kubectl -n rook-ceph port-forward service/rook-ceph-mgr-dashboard 8443:8443
```

Retrieve the admin password:

```bash
kubectl -n rook-ceph get secret rook-ceph-dashboard-password \
  -o jsonpath='{.data.password}' | base64 -d
```

Open `https://localhost:8443` in your browser and log in as `admin`.

## Navigating to NFS Exports

In the Ceph Dashboard:

1. Click **NFS** in the left navigation menu
2. Select **NFS** under the Ganesha section
3. The list of existing NFS clusters and exports is displayed

## Creating a New Export via Dashboard

Click **Create** to open the export creation form and fill in:

```text
Cluster: my-nfs
Storage Backend: CephFS
Filesystem: my-fs
Path: /
Pseudo Path: /data
Access Type: RW
Squash: None
Protocols: NFSv4
Transports: TCP
```

For an object store-backed export, change **Storage Backend** to RGW and specify the bucket name.

Click **Submit** to create the export. The dashboard sends the configuration to the Ceph Manager NFS module, which writes it to the RADOS pool and notifies Ganesha instances.

## Viewing and Editing Exports

After creation, the export appears in the NFS list. Click the export row to see its full JSON configuration. Click **Edit** to modify access type, squash settings, or client restrictions.

To add a client restriction that allows only a specific CIDR:

```text
Clients: 192.168.10.0/24
Access Type: RO
Squash: Root
```

This limits read-only access to clients on the specified subnet.

## Deleting an Export

Select the export in the list and click **Delete**. The dashboard will prompt for confirmation before removing the export configuration from RADOS and notifying Ganesha.

## Monitoring NFS via Dashboard

The NFS section also shows daemon status. Navigate to **NFS > Daemons** to see:

```text
Daemon: nfs.my-nfs.0
Status: running
Host: node-1
```

This provides a quick health check without needing `kubectl` access.

## Summary

The Ceph Dashboard is a convenient GUI alternative to the `ceph nfs` CLI for managing exports in Rook. Access it by exposing the dashboard service, navigate to the NFS section, and use the create form to define CephFS or RGW-backed exports. All changes are stored in RADOS and applied live to Ganesha daemons, making the dashboard and CLI fully interchangeable.
