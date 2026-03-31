# How to Configure Default and Global Quotas in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Quota, Administration, Object Storage, Governance, Default

Description: Configure default and global quotas in Ceph RGW to automatically apply storage limits to all new users and buckets without setting them individually.

---

Managing quotas individually for each user is impractical at scale. Ceph RGW supports default quotas (applied automatically to new users) and global quotas (applied cluster-wide) to enforce storage limits without per-user configuration.

## Default User Quotas

Default user quotas are applied to every new user created after the default is set. Existing users are not affected.

Set default user quotas at the zone level:

```bash
radosgw-admin zone modify \
  --rgw-zone default \
  --default-quota-max-size 53687091200 \
  --default-quota-max-objects 5000000

# Apply the period update
radosgw-admin period update --commit
```

Verify the default quota is set in zone configuration:

```bash
radosgw-admin zone get --rgw-zone default | jq '.default_user_quota'
```

Output:

```json
{
  "enabled": true,
  "max_size": 53687091200,
  "max_objects": 5000000
}
```

## Testing Default Quota Application

Create a new user and verify the quota was applied automatically:

```bash
radosgw-admin user create \
  --uid newuser \
  --display-name "New User"

radosgw-admin quota get \
  --uid newuser \
  --quota-scope user
```

The new user should have the default quota pre-configured and enabled.

## Default Bucket Quotas

Set a default bucket quota (applied to every new bucket):

```bash
radosgw-admin zone modify \
  --rgw-zone default \
  --default-bucket-quota-max-size 10737418240 \
  --default-bucket-quota-max-objects 1000000

radosgw-admin period update --commit
```

## Global Quotas via Configuration

For stricter cluster-wide enforcement, set global quota limits in RGW config:

```bash
ceph config set client.rgw rgw_user_quota_max_size 53687091200
ceph config set client.rgw rgw_bucket_quota_max_size 10737418240
ceph config set client.rgw rgw_quota_check_threads 2
```

These act as hard caps that cannot be exceeded even if individual quotas are not set.

## Applying Default Quotas to Existing Users

Default quotas only apply to new users. To apply them to existing users in bulk:

```bash
#!/bin/bash
MAX_SIZE=53687091200
MAX_OBJECTS=5000000

radosgw-admin user list | jq -r '.[]' | while read UID; do
  # Check if user already has a quota set
  CURRENT=$(radosgw-admin quota get --uid "$UID" --quota-scope user | jq -r '.max_size')
  if [ "$CURRENT" = "-1" ]; then
    echo "Setting default quota for $UID"
    radosgw-admin quota set --uid "$UID" --quota-scope user \
      --max-size $MAX_SIZE --max-objects $MAX_OBJECTS
    radosgw-admin quota enable --uid "$UID" --quota-scope user
  fi
done
```

## Rook: Configuring Default Quotas

In Rook deployments, set zone defaults via the Ceph toolbox:

```bash
kubectl exec -it -n rook-ceph deploy/rook-ceph-tools -- \
  radosgw-admin zone modify --rgw-zone default \
  --default-quota-max-size 53687091200 && \
  radosgw-admin period update --commit
```

## Summary

Default quotas in Ceph RGW automatically apply storage limits to new users and buckets without per-object configuration. Set them at the zone level with `zone modify` and commit the period. For existing users, use a bulk script to apply quotas retroactively. This approach scales quota management across thousands of users without manual intervention.
