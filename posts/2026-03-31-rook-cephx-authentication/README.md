# How to Understand CephX Authentication (Shared Secrets, Tickets, Session Keys)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephX, Authentication, Security, Kubernetes

Description: Learn how CephX uses shared secrets, session tickets, and session keys to authenticate Ceph clients and services without transmitting passwords in the clear.

---

## What Is CephX

CephX is Ceph's built-in authentication system, modeled after Kerberos. It provides mutual authentication between clients and Ceph services (monitors, OSDs, MDS) using shared secret keys. No passwords are ever transmitted over the network - instead, challenges and encrypted tickets prove identity.

In Rook deployments, CephX is enabled by default and Kubernetes Secrets store the generated keyrings.

## Core Concepts

### Shared Secrets

Every principal (client, OSD, monitor) has a secret key stored in a keyring file. The monitor holds copies of all keys and acts as the authentication authority (similar to a KDC in Kerberos).

```bash
kubectl exec -it rook-ceph-tools -n rook-ceph -- bash

# View the admin keyring
cat /etc/ceph/keyring

# List all CephX entities
ceph auth ls
```

### Authentication Tickets

When a client wants to communicate with an OSD, it follows this flow:

1. Client presents its secret key to the monitor
2. Monitor verifies the key and issues an encrypted ticket (session authenticator)
3. Client presents the ticket to the OSD
4. OSD decrypts the ticket using a shared key it received from the monitor
5. Both parties derive a shared session key for encrypting the I/O session

```bash
# Get a keyring for a specific client
ceph auth get client.admin

# Create a new client with specific capabilities
ceph auth get-or-create client.myapp \
  mon 'allow r' \
  osd 'allow rw pool=mypool'
```

### Session Keys

Session keys are ephemeral symmetric keys negotiated per connection. They expire after the session ends and are derived from the ticket exchange. This prevents replay attacks - even if traffic is captured, the session key cannot be reused.

## Managing Keys in Rook

Rook stores CephX keyrings as Kubernetes Secrets:

```bash
# List CephX secrets created by Rook
kubectl get secrets -n rook-ceph | grep keyring

# View the CSI keyring (used by RBD/CephFS CSI driver)
kubectl get secret rook-csi-rbd-provisioner -n rook-ceph -o yaml
```

The CSI driver uses these secrets to authenticate when provisioning volumes and when pods mount storage.

## Capability System

CephX keys include capability strings that define what the entity is allowed to do:

```bash
# View capabilities for CSI provisioner
ceph auth get client.csi-rbd-provisioner
```

Example output:

```text
[client.csi-rbd-provisioner]
  key = AQB...==
  caps mds = "allow rw"
  caps mon = "profile rbd"
  caps osd = "profile rbd"
```

Capabilities follow the principle of least privilege - only grant what each component needs.

## Creating a Read-Only Client

```bash
# Create a read-only client for monitoring
ceph auth get-or-create client.monitoring \
  mon 'allow r' \
  osd 'allow r'

# Export keyring to file
ceph auth get client.monitoring -o /etc/ceph/ceph.client.monitoring.keyring
```

## Rotating Keys

If a key is compromised, rotate it:

```bash
# Generate a new key for a client
ceph auth get-or-create-key client.myapp

# Delete and recreate with new key
ceph auth del client.myapp
ceph auth get-or-create client.myapp mon 'allow r' osd 'allow rw pool=mypool'
```

## Summary

CephX provides strong mutual authentication using shared secrets, time-limited session tickets, and ephemeral session keys. Clients never transmit raw passwords - instead, the monitor issues encrypted tickets that OSDs can verify using their own shared keys. In Rook-Ceph, CephX keys are managed automatically and stored as Kubernetes Secrets, but understanding the underlying mechanism helps you manage custom clients, audit access, rotate compromised keys, and apply least-privilege capability policies.
