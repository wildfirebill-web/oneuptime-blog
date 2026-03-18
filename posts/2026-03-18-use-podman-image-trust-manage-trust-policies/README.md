# How to Use podman image trust to Manage Trust Policies

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Security, Trust Policies, CLI

Description: Learn how to use the podman image trust command to view, set, and manage container image trust policies directly from the command line.

---

> The podman image trust command puts trust policy management at your fingertips without needing to edit JSON files by hand.

Managing trust policies through JSON configuration files can be error-prone and tedious. Podman provides the `podman image trust` command to view and modify trust policies directly from the command line. This guide covers all aspects of using this command to manage your image trust configuration.

---

## Understanding podman image trust

The `podman image trust` command provides two subcommands: `show` to display current trust settings, and `set` to configure trust policies for specific registries. These commands modify the same `policy.json` file used by Podman during image pulls.

## Viewing Current Trust Policies

```bash
# Display the current trust configuration
podman image trust show
```

```bash
# Show trust policies in a table format
podman image trust show 2>/dev/null

# The output shows columns:
# TRANSPORT  NAME                       TYPE        ID         STORE
# docker     registry.example.com       signedBy   GPGKey     https://...
# docker     docker.io/library          accept
# docker     default                    reject
```

```bash
# Show raw JSON output (if supported by your Podman version)
podman image trust show --json 2>/dev/null || \
  echo "JSON output may not be available in all versions"

# Alternatively, view the policy file directly
cat /etc/containers/policy.json | python3 -m json.tool
```

## Setting Trust to Accept All Images

For development environments, you may want to accept all images from a registry.

```bash
# Accept all images from a specific registry
sudo podman image trust set -t accept registry.example.com

# Verify the change
podman image trust show
```

```bash
# Accept all images from Docker Hub official library
sudo podman image trust set -t accept docker.io/library
```

## Setting Trust to Reject Images

Block images from untrusted registries.

```bash
# Reject all images from an untrusted registry
sudo podman image trust set -t reject untrusted-registry.com

# Set the default policy to reject
sudo podman image trust set -t reject default

# Verify
podman image trust show
```

## Requiring Signed Images

Configure a registry to require GPG-signed images.

```bash
# First, ensure you have the public key available
sudo mkdir -p /etc/pki/containers

# Set signedBy trust for a registry
sudo podman image trust set -t signedBy \
  --pubkeysfile /etc/pki/containers/signer.gpg \
  registry.example.com

# Verify the configuration
podman image trust show
```

```bash
# Set signedBy trust with a signature store URL
sudo podman image trust set -t signedBy \
  --pubkeysfile /etc/pki/containers/signer.gpg \
  registry.secure.example.com
```

## Managing Multiple Registries

Configure trust policies for several registries at once.

```bash
#!/bin/bash
# setup-trust.sh - Configure trust policies for multiple registries

# Set default policy to reject
sudo podman image trust set -t reject default

# Allow official Docker Hub images
sudo podman image trust set -t accept docker.io/library

# Allow Red Hat images with signature verification
sudo podman image trust set -t signedBy \
  --pubkeysfile /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release \
  registry.access.redhat.com

# Allow internal registry with signature verification
sudo podman image trust set -t signedBy \
  --pubkeysfile /etc/pki/containers/internal-signer.gpg \
  registry.internal.example.com

# Allow development images without signatures
sudo podman image trust set -t accept dev-registry.example.com

# Show the final configuration
echo "=== Final Trust Configuration ==="
podman image trust show
```

```bash
chmod +x setup-trust.sh
sudo ./setup-trust.sh
```

## Comparing CLI Changes with policy.json

```bash
# View the policy file before changes
echo "=== Before ==="
cat /etc/containers/policy.json | python3 -m json.tool

# Make a trust change via CLI
sudo podman image trust set -t accept quay.io/myorg

# View the policy file after changes
echo "=== After ==="
cat /etc/containers/policy.json | python3 -m json.tool
```

## Removing Trust Settings

```bash
# To remove a trust setting for a specific registry,
# set it back to the default behavior
# or edit policy.json directly

# Reset a registry to follow the default policy
# (remove its specific entry from policy.json)
sudo python3 -c "
import json
with open('/etc/containers/policy.json', 'r') as f:
    policy = json.load(f)
docker_transport = policy.get('transports', {}).get('docker', {})
if 'quay.io/myorg' in docker_transport:
    del docker_transport['quay.io/myorg']
with open('/etc/containers/policy.json', 'w') as f:
    json.dump(policy, f, indent=2)
print('Registry trust entry removed')
"
```

## Auditing Trust Configuration

```bash
#!/bin/bash
# audit-trust.sh - Audit the current trust configuration

echo "========================================"
echo "Podman Image Trust Audit"
echo "Date: $(date)"
echo "========================================"

# Show current trust settings
echo ""
echo "--- Current Trust Policies ---"
podman image trust show 2>/dev/null

# Check for overly permissive defaults
echo ""
echo "--- Default Policy Check ---"
default_type=$(python3 -c "
import json
with open('/etc/containers/policy.json') as f:
    p = json.load(f)
print(p.get('default', [{}])[0].get('type', 'unknown'))
")

if [ "$default_type" = "insecureAcceptAnything" ]; then
  echo "[WARN] Default policy accepts all images (consider setting to reject)"
elif [ "$default_type" = "reject" ]; then
  echo "[PASS] Default policy rejects unsigned images"
else
  echo "[INFO] Default policy type: $default_type"
fi

# Check for key files
echo ""
echo "--- Key File Verification ---"
grep -o '"keyPath": "[^"]*"' /etc/containers/policy.json 2>/dev/null | while read line; do
  keyfile=$(echo "$line" | cut -d'"' -f4)
  if [ -f "$keyfile" ]; then
    echo "[PASS] Key exists: $keyfile"
  else
    echo "[FAIL] Key missing: $keyfile"
  fi
done

echo ""
echo "Audit complete."
```

```bash
chmod +x audit-trust.sh
./audit-trust.sh
```

## Cleanup

```bash
rm -f setup-trust.sh audit-trust.sh
```

## Summary

The `podman image trust` command provides a convenient CLI interface for managing image trust policies without manually editing JSON files. Use `podman image trust show` to review your current configuration and `podman image trust set` to add or modify policies for specific registries. Combine this with scripted setups for consistent policy deployment across your infrastructure, and regularly audit your trust configuration to ensure it matches your security requirements.
