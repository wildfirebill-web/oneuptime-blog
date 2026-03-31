# How to Understand CephX Authentication Flow

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephX, Authentication, Security, Kubernetes

Description: A practical guide to understanding how CephX authentication works in Ceph clusters, including key exchange, capability enforcement, and troubleshooting auth failures.

---

CephX is the authentication protocol used by Ceph to secure communication between clients and cluster daemons. Understanding how CephX works is essential for administering secure Ceph deployments and diagnosing authentication failures.

## What is CephX?

CephX is a shared-secret authentication system inspired by Kerberos. It uses symmetric cryptography and session tickets to authenticate entities (clients, OSDs, MONs, MDSs) without transmitting secrets over the network.

The key participants in CephX are:
- The Ceph Monitor, which acts as the authentication server
- Clients requesting access (applications, RBD, CephFS users)
- Daemons (OSDs, MDSs) that validate client credentials

## The Authentication Flow

The CephX handshake follows these steps:

1. **Client sends auth request** - The client contacts a Monitor with its entity name (e.g., `client.admin`)
2. **Monitor issues a challenge** - The Monitor sends a random challenge encrypted with the client's shared secret
3. **Client proves identity** - The client decrypts the challenge and responds, proving it holds the secret
4. **Monitor issues a session ticket** - A time-limited ticket is returned, granting access to specific services
5. **Client presents ticket to OSD/MDS** - The service validates the ticket against the Monitor's shared secret

```bash
# View all existing auth keys in the cluster
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph auth list

# Get details for a specific entity
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph auth get client.admin
```

## Keyring Files and Their Role

Each entity in the cluster uses a keyring file to store its shared secret:

```bash
# Export a specific keyring
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph auth get client.admin -o /tmp/ceph.client.admin.keyring

# Print keyring content
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph auth print-key client.admin
```

In Rook, keyrings are stored as Kubernetes Secrets:

```bash
kubectl -n rook-ceph get secret rook-ceph-admin-keyring -o yaml
```

## Capabilities and What They Control

Every key has capability strings that define what it can do:

```bash
# Create a restricted client key
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph auth get-or-create client.myapp \
    mon 'allow r' \
    osd 'allow rw pool=mypool'
```

Common capability values:
- `allow r` - read-only access
- `allow rw` - read/write access
- `allow *` - full access (use with caution)
- `allow rw pool=<name>` - scoped to a specific pool

## Debugging Authentication Failures

If a client fails to authenticate, check the Monitor logs:

```bash
kubectl -n rook-ceph logs deploy/rook-ceph-mon-a | grep -i "auth\|EACCES\|no key"
```

Test authentication directly:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph --id myapp --keyring /etc/ceph/keyring health
```

Common errors and fixes:
- `EACCES` - wrong key or capability mismatch
- `auth: error reading file` - keyring file missing or wrong permissions
- `no key: client.X` - entity does not exist in auth database

## Summary

CephX authentication uses a challenge-response mechanism where Monitors act as an authentication authority, issuing time-limited session tickets to verified clients. Keyrings store shared secrets, and capabilities define fine-grained access permissions. In Rook-managed clusters, keyrings are stored as Kubernetes Secrets and can be inspected and rotated via standard Ceph auth commands or the Rook operator.
