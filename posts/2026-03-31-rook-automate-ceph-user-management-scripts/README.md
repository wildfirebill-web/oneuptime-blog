# How to Automate Ceph User Management with Scripts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Scripting, User Management, Automation, RGW

Description: Automate Ceph RGW user lifecycle management including creation, quota assignment, key rotation, and deprovisioning using Bash and Python scripts.

---

Managing dozens or hundreds of Ceph RGW users manually is error-prone and time-consuming. Automating user lifecycle operations ensures consistent configurations, enforces quotas, and speeds up onboarding and offboarding workflows.

## Bulk User Creation from CSV

Create a script to provision users from a CSV file:

```bash
#!/bin/bash
# create-ceph-users.sh
# CSV format: uid,display_name,email,max_buckets,quota_max_size_gb

NAMESPACE="${CEPH_NAMESPACE:-rook-ceph}"
TOOLS="deploy/rook-ceph-tools"
INPUT_FILE="${1:-users.csv}"

rgw_admin() {
  kubectl -n "$NAMESPACE" exec -it "$TOOLS" -- radosgw-admin "$@"
}

while IFS=',' read -r uid display_name email max_buckets quota_gb; do
  [[ "$uid" == "uid" ]] && continue  # skip header

  echo "Creating user: $uid"
  rgw_admin user create \
    --uid="$uid" \
    --display-name="$display_name" \
    --email="$email" \
    --max-buckets="$max_buckets"

  # Set quota
  rgw_admin quota set \
    --uid="$uid" \
    --quota-scope=user \
    --max-size="${quota_gb}GiB" \
    --max-objects=1000000

  rgw_admin quota enable --uid="$uid" --quota-scope=user

  echo "User $uid created with ${quota_gb}GiB quota"
done < "$INPUT_FILE"
```

## Rotating Access Keys

Automate key rotation for security compliance:

```python
#!/usr/bin/env python3
"""Rotate access keys for all Ceph RGW users older than 90 days."""

import json
import subprocess
from datetime import datetime, timedelta

NAMESPACE = "rook-ceph"
ROTATION_DAYS = 90


def rgw_admin(*args):
    cmd = [
        "kubectl", "-n", NAMESPACE, "exec", "deploy/rook-ceph-tools",
        "--", "radosgw-admin", *args, "--format", "json"
    ]
    result = subprocess.run(cmd, capture_output=True, text=True)
    return json.loads(result.stdout) if result.stdout else {}


def rotate_keys():
    users = rgw_admin("user", "list")
    for uid in users:
        user_info = rgw_admin("user", "info", f"--uid={uid}")
        for key in user_info.get("keys", []):
            access_key = key["access_key"]
            # Rotate by removing old key and creating new one
            print(f"Rotating key for user {uid}")
            rgw_admin("key", "rm", f"--uid={uid}",
                      f"--access-key={access_key}")
            new_key = rgw_admin("key", "create", f"--uid={uid}")
            print(f"  New access key: {new_key.get('keys', [{}])[0].get('access_key', 'N/A')}")


if __name__ == "__main__":
    rotate_keys()
```

## Deprovisioning Users Safely

```bash
#!/bin/bash
# deprovision-user.sh

NAMESPACE="${CEPH_NAMESPACE:-rook-ceph}"
TOOLS="deploy/rook-ceph-tools"
UID="$1"

if [[ -z "$UID" ]]; then
  echo "Usage: $0 <uid>"
  exit 1
fi

rgw_admin() {
  kubectl -n "$NAMESPACE" exec -it "$TOOLS" -- radosgw-admin "$@"
}

# List buckets owned by user before removal
echo "Buckets owned by $UID:"
rgw_admin bucket list --uid="$UID"

# Suspend user first (soft delete)
echo "Suspending user $UID..."
rgw_admin user suspend --uid="$UID"

# Optionally remove buckets and data
read -rp "Remove all buckets and data for $UID? [y/N] " confirm
if [[ "$confirm" == "y" ]]; then
  for bucket in $(rgw_admin bucket list --uid="$UID" | python3 -c "import sys,json; [print(b) for b in json.load(sys.stdin)]"); do
    echo "Removing bucket: $bucket"
    rgw_admin bucket rm --bucket="$bucket" --purge-objects
  done
  rgw_admin user rm --uid="$UID"
  echo "User $UID removed"
fi
```

## Quota Audit Report

```bash
#!/bin/bash
# quota-report.sh

NAMESPACE="${CEPH_NAMESPACE:-rook-ceph}"
TOOLS="deploy/rook-ceph-tools"

echo "User Quota Report - $(date)"
echo "=============================="

users=$(kubectl -n "$NAMESPACE" exec -it "$TOOLS" -- \
  radosgw-admin user list --format json | python3 -c "import sys,json; [print(u) for u in json.load(sys.stdin)]")

for uid in $users; do
  info=$(kubectl -n "$NAMESPACE" exec -it "$TOOLS" -- \
    radosgw-admin user info --uid="$uid" --format json)
  quota=$(echo "$info" | python3 -c "
import sys, json
d = json.load(sys.stdin)
q = d.get('user_quota', {})
print(f\"{d['user_id']}: enabled={q.get('enabled',False)} max_size={q.get('max_size_kb',0)}KB\")
")
  echo "$quota"
done
```

## Summary

Automating Ceph user management through scripts reduces manual errors and enforces consistent policies across all users. By combining Bash for bulk operations and Python for complex workflows like key rotation, you can manage hundreds of RGW users efficiently while maintaining audit trails and security compliance.
