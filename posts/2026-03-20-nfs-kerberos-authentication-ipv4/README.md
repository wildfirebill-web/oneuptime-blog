# How to Secure NFS Exports with Kerberos Authentication on IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: NFS, Kerberos, IPv4, Security, Authentication, Linux, krb5

Description: Learn how to configure NFS exports secured with Kerberos authentication on IPv4 to prevent unauthorized access to shared filesystems.

---

NFS without Kerberos relies only on IP-based access control and trusts client UIDs. Kerberos adds cryptographic authentication, ensuring only properly authenticated hosts and users can mount NFS shares.

## Prerequisites

- A Kerberos KDC (e.g., MIT krb5 or FreeIPA) reachable over IPv4.
- `nfs-utils` and `krb5-libs` installed on both server and client.
- Host keytabs for both the NFS server and client.

## Server-Side Setup

### Install NFS and Kerberos Support

```bash
dnf install nfs-utils krb5-workstation -y    # RHEL/Rocky
# or
apt install nfs-kernel-server krb5-user -y   # Debian/Ubuntu
```

### Get a Host Keytab from the KDC

```bash
# On the KDC: create a service principal for the NFS server
kadmin.local -q "addprinc -randkey nfs/nfs-server.example.com@EXAMPLE.COM"
kadmin.local -q "ktadd -k /etc/krb5.keytab nfs/nfs-server.example.com@EXAMPLE.COM"
```

### Configure NFS Exports

```bash
# /etc/exports

# Export /data with krb5p (integrity + encryption) to the IPv4 subnet
/data  192.168.1.0/24(rw,sync,sec=krb5p,fsid=0)

# sec options:
# krb5   = identity only (authentication)
# krb5i  = integrity (authentication + checksums)
# krb5p  = privacy (authentication + encryption)
```

```bash
# Export and start NFS services
exportfs -arv
systemctl enable --now nfs-server rpcbind
```

### Enabling GSS (Kerberos) Daemon

```bash
# Enable and start the RPC GSS daemon for Kerberos NFS
systemctl enable --now rpc-gssd
```

## Client-Side Setup

### Get the Client Keytab

```bash
# On the KDC: create and export the client host principal
kadmin.local -q "addprinc -randkey host/nfs-client.example.com@EXAMPLE.COM"
kadmin.local -q "ktadd -k /etc/krb5.keytab host/nfs-client.example.com@EXAMPLE.COM"
```

### Mount with Kerberos

```bash
# Mount using krb5p security flavor
mount -t nfs4 -o sec=krb5p 192.168.1.10:/ /mnt/nfs

# Persistent mount in /etc/fstab
echo "192.168.1.10:/ /mnt/nfs nfs4 sec=krb5p,rw,sync 0 0" >> /etc/fstab
mount -a
```

```bash
# Start rpc-gssd on the client too
systemctl enable --now rpc-gssd
```

## Verifying Kerberos NFS

```bash
# Get a Kerberos ticket (as a regular user)
kinit user@EXAMPLE.COM

# Access the NFS share — should work with the ticket
ls /mnt/nfs

# Without a valid ticket, access should be denied
kdestroy
ls /mnt/nfs  # Should return permission denied
```

## Key Takeaways

- `sec=krb5p` on exports requires clients to authenticate with Kerberos and encrypts data in transit.
- Both the NFS server and client need host principals and keytabs from the KDC.
- `rpc-gssd` must run on both server and client to handle GSS-API (Kerberos) authentication.
- Use `sec=krb5i` if encryption is not needed but data integrity (checksums) is desired.
