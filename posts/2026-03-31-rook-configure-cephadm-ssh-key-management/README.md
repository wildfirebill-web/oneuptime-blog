# How to Configure cephadm SSH Key Management

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, cephadm, SSH, Security, Key Management, Admin

Description: Manage cephadm's SSH keys used for host connectivity, including key rotation, custom key pairs, and troubleshooting SSH authentication failures.

---

## How cephadm Uses SSH

cephadm uses a dedicated SSH key pair stored in the Ceph monitor key-value store to connect to all managed hosts. This is separate from your personal SSH key. The private key stays on the admin node; the public key must be present in `authorized_keys` on every managed host.

## Viewing the Current Public Key

```bash
ceph cephadm get-pub-key
```

Or directly:

```bash
cat /etc/ceph/ceph.pub
```

## Distributing the Key to a New Host

```bash
# Method 1: ssh-copy-id
ssh-copy-id -f -i /etc/ceph/ceph.pub root@10.0.0.25

# Method 2: Manual
ssh root@10.0.0.25 \
  "mkdir -p ~/.ssh && chmod 700 ~/.ssh && \
   echo '$(cat /etc/ceph/ceph.pub)' >> ~/.ssh/authorized_keys && \
   chmod 600 ~/.ssh/authorized_keys"
```

## Testing SSH Connectivity

Before adding a host, verify cephadm can reach it:

```bash
ceph cephadm check-host --addr 10.0.0.25
```

This tests SSH, Python, and container runtime availability.

## Rotating the SSH Key

To rotate the SSH key (e.g., after a security incident):

```bash
# Generate a new key pair
ceph cephadm generate-key

# View the new public key
ceph cephadm get-pub-key

# Re-distribute to all managed hosts
ceph orch host ls --format json | \
  python3 -c "import json,sys; [print(h['hostname']) for h in json.load(sys.stdin)]" | \
  while read host; do
    ssh-copy-id -f -i /etc/ceph/ceph.pub root@$host
  done
```

## Using a Custom SSH Key

To use your own SSH key pair instead of the auto-generated one:

```bash
# Generate your own key
ssh-keygen -t ed25519 -f /etc/ceph/ceph_custom -N ""

# Import it into cephadm
ceph cephadm set-priv-key -i /etc/ceph/ceph_custom
ceph cephadm set-pub-key -i /etc/ceph/ceph_custom.pub
```

## Configuring a Non-Root SSH User

If you cannot use root:

```bash
ceph config set mgr mgr/cephadm/ssh_user myuser
```

Ensure `myuser` has passwordless sudo and the public key in `~myuser/.ssh/authorized_keys`.

## Troubleshooting SSH Failures

```bash
# Test SSH manually
ssh -i /etc/ceph/ceph -o StrictHostKeyChecking=no root@10.0.0.25 hostname

# Check cephadm SSH debug output
ceph cephadm check-host --addr 10.0.0.25 2>&1
```

Common causes:
- Public key not in `authorized_keys` on the target host
- Wrong file permissions on `~/.ssh` (must be 700) or `authorized_keys` (must be 600)
- `PermitRootLogin` disabled in `sshd_config`
- Firewall blocking port 22

## Summary

cephadm uses a dedicated SSH key stored in the Ceph mon key-value store. Distribute the public key to all hosts using `ssh-copy-id`, test connectivity with `ceph cephadm check-host`, and rotate keys after security incidents with `ceph cephadm generate-key`. Use `ceph config set mgr mgr/cephadm/ssh_user` to configure a non-root user for SSH connections.
