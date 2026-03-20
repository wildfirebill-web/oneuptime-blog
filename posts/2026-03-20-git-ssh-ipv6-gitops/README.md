# How to Configure Git SSH over IPv6 for GitOps

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Git, SSH, GitOps, Authentication, OpenSSH

Description: Configure Git SSH connections over IPv6 for GitOps workflows, including SSH config for IPv6 hosts, host key verification, known_hosts formatting, and ArgoCD/Flux SSH key configuration.

## Introduction

SSH is the most common protocol for Git authentication in GitOps pipelines. Connecting to Git servers over IPv6 via SSH requires proper URL formatting, SSH config adjustments for IPv6 hosts, and correctly formatted `known_hosts` entries. Both ArgoCD and Flux CD use SSH keys stored in Kubernetes secrets with IPv6-formatted known_hosts to authenticate with Git servers.

## SSH URL Format for IPv6 Git Servers

```bash
# Standard SSH Git URL formats for IPv6:

# Format 1: SCP-like (does NOT support IPv6 literal addresses)

# git@[2001:db8::git]:org/repo.git  ← NOT valid
# Use hostname with AAAA record instead:
git clone git@gitserver.example.com:org/repo.git

# Format 2: Full SSH URL (supports IPv6 literal)
git clone "ssh://git@[2001:db8::git]:22/org/repo.git"

# Format 3: SSH URL without port (port 22)
git clone "ssh://git@[2001:db8::git]/org/repo.git"

# Test SSH connection to IPv6 Git server
ssh -6 -T git@2001:db8::git
# Expected: "Hi user! You've successfully authenticated..."

# Test with full URL
ssh -6 -v "ssh://git@[2001:db8::git]:22"
```

## SSH Config for IPv6 Git Server

```bash
# ~/.ssh/config

# Git server on IPv6
Host gitserver-ipv6
    HostName 2001:db8::git
    User git
    Port 22
    IdentityFile ~/.ssh/id_ed25519_gitops
    # Force IPv6
    AddressFamily inet6

# Gitea on IPv6
Host gitea.example.com
    User git
    IdentityFile ~/.ssh/id_ed25519
    # Hostname resolves to AAAA record - no AddressFamily needed
    # but forces IPv6:
    AddressFamily inet6

# Use the alias in Git commands
git clone "git@gitserver-ipv6:org/repo.git"
```

## known_hosts Format for IPv6

```bash
# Add IPv6 host key to known_hosts

# Standard format for IPv6 in known_hosts requires brackets:
# [hostname]:port key-type key-data
# or for port 22:
# [ipv6-address] key-type key-data

# Scan host keys from IPv6 Git server
ssh-keyscan -6 -H 2001:db8::git >> ~/.ssh/known_hosts
# Adds: |1|hash| ecdsa-sha2-nistp256 AAAAE2...

# Non-hashed format (easier to read/manage)
ssh-keyscan -6 2001:db8::git >> ~/.ssh/known_hosts
# Adds: 2001:db8::git ssh-rsa AAAAB3...
#   or: [2001:db8::git]:22 ecdsa-sha2-nistp256 AAAAE2...

# Verify the known_hosts entry works
ssh -6 -o "StrictHostKeyChecking=yes" git@2001:db8::git
```

## ArgoCD: SSH Repository with IPv6

```yaml
# argocd-ssh-ipv6-repo.yaml

apiVersion: v1
kind: Secret
metadata:
  name: gitserver-ssh
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  # SSH URL with IPv6 literal (must use ssh:// format)
  url: "ssh://git@[2001:db8::git]:22/org/myrepo.git"

  # SSH private key
  sshPrivateKey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQ...
    -----END OPENSSH PRIVATE KEY-----

  # known_hosts for the IPv6 Git server
  # Get with: ssh-keyscan -6 2001:db8::git
  knownHosts: |
    [2001:db8::git]:22 ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNo...
    [2001:db8::git]:22 ssh-rsa AAAAB3NzaC1yc2EAAAA...
    [2001:db8::git]:22 ssh-ed25519 AAAAC3NzaC1lZDI1NTE5...
```

## Flux CD: SSH Repository with IPv6

```yaml
# flux-ssh-ipv6.yaml

apiVersion: v1
kind: Secret
metadata:
  name: git-ssh-key
  namespace: flux-system
type: Opaque
stringData:
  identity: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAA...
    -----END OPENSSH PRIVATE KEY-----
  identity.pub: "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5... gitops-key"
  # known_hosts with IPv6 server entry
  known_hosts: |
    [2001:db8::git]:22 ssh-ed25519 AAAAC3NzaC1lZDI1NTE5...

---
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 5m
  url: "ssh://git@[2001:db8::git]:22/org/myapp.git"
  ref:
    branch: main
  secretRef:
    name: git-ssh-key
```

## Get Host Keys for IPv6 Git Servers

```bash
#!/bin/bash
# get-ipv6-hostkeys.sh - Collect SSH host keys from IPv6 Git server

GIT_SERVER_IPV6="2001:db8::git"
OUTPUT="known_hosts_ipv6"

echo "Scanning SSH host keys from [$GIT_SERVER_IPV6]..."

# Get all key types
ssh-keyscan -6 -t rsa,ecdsa,ed25519 "$GIT_SERVER_IPV6" 2>/dev/null | \
    sed "s/^/[$GIT_SERVER_IPV6]:22 /" | \
    sed "s/\[$GIT_SERVER_IPV6\]:22 $GIT_SERVER_IPV6 /[$GIT_SERVER_IPV6]:22 /" \
    > "$OUTPUT"

echo "Host keys written to $OUTPUT:"
cat "$OUTPUT"
echo ""
echo "Add to ArgoCD or Flux known_hosts secret field."
```

## Test SSH over IPv6 from GitOps Pod

```bash
# Test SSH connectivity from ArgoCD repo-server pod
kubectl exec -n argocd deployment/argocd-repo-server -- \
    ssh -6 -o StrictHostKeyChecking=no \
        -i /app/config/ssh/id_rsa \
        "git@[2001:db8::git]" 2>&1

# Test from Flux source-controller
kubectl exec -n flux-system deployment/source-controller -- \
    ssh -6 -T "git@[2001:db8::git]" 2>&1

# If SSH fails: check if IPv6 is available in the pod
kubectl exec -n flux-system deployment/source-controller -- \
    ip -6 addr show
```

## Conclusion

Git SSH over IPv6 requires using the `ssh://` URL scheme with bracket notation for IPv6 addresses (e.g., `ssh://git@[2001:db8::git]:22/repo.git`) since the SCP-like syntax doesn't support IPv6 literals. The `known_hosts` file must contain the IPv6 server's host keys in `[ipv6address]:port key-type data` format, collected with `ssh-keyscan -6`. ArgoCD stores known_hosts in the repository secret's `knownHosts` field; Flux stores it in the SSH secret's `known_hosts` field. Use `ssh-keyscan -6 -t rsa,ecdsa,ed25519` to collect all key types. Verify SSH over IPv6 is working by testing from inside the GitOps controller pod with `kubectl exec` before configuring the repository secret.
