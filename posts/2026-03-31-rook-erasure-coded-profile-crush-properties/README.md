# How to Configure Erasure Coded Profile Properties in CRUSH

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Erasure Coding, CRUSH, Storage

Description: Configure Ceph erasure coding profile CRUSH properties including failure domain, device class, and chunk placement to achieve the desired resilience and topology.

---

## Erasure Code Profiles and CRUSH

An erasure code profile in Ceph bundles two concerns: the erasure coding algorithm settings (k, m, and plugin) and the CRUSH placement properties (failure domain and device class). The CRUSH properties in a profile tell Ceph how to spread the erasure coded chunks across the topology.

When you create a pool with an erasure profile, Ceph generates a matching CRUSH rule that targets the specified failure domain and device class.

## Key CRUSH Properties in an Erasure Profile

- `crush-root` - the CRUSH bucket to start placement from (default: `default`)
- `crush-failure-domain` - the bucket type across which chunks are spread (default: `host`)
- `crush-device-class` - target only OSDs of this class (optional)
- `crush-osds-per-failure-domain` - number of OSDs to use per failure domain instance (default: 1)
- `crush-num-failure-domains` - total number of failure domains to use (k+m by default)

## Listing and Inspecting Profiles

```bash
# List all erasure code profiles
ceph osd erasure-code-profile ls

# Get details on the default profile
ceph osd erasure-code-profile get default

# Get a custom profile
ceph osd erasure-code-profile get my-profile
```

Example profile output:

```text
crush-device-class=
crush-failure-domain=host
crush-osds-per-failure-domain=1
crush-root=default
jerasure-per-chunk-alignment=false
k=2
m=1
plugin=jerasure
technique=reed_sol_van
w=8
```

## Creating Profiles with Custom CRUSH Properties

```bash
# Profile spreading chunks across racks (not just hosts)
ceph osd erasure-code-profile set ec-rack-profile \
  k=4 \
  m=2 \
  crush-failure-domain=rack \
  crush-root=default \
  plugin=jerasure \
  technique=reed_sol_van

# Profile using only SSD OSDs
ceph osd erasure-code-profile set ec-ssd-profile \
  k=4 \
  m=2 \
  crush-failure-domain=host \
  crush-device-class=ssd

# Profile with datacenter-level failure domain
ceph osd erasure-code-profile set ec-dc-profile \
  k=3 \
  m=2 \
  crush-failure-domain=datacenter

# Verify the profile
ceph osd erasure-code-profile get ec-rack-profile
```

## Creating a Pool with the Profile

```bash
# Create a pool using the custom erasure profile
ceph osd pool create ec-data 128 128 erasure ec-rack-profile

# Verify the pool and its auto-generated CRUSH rule
ceph osd pool get ec-data all | grep crush
ceph osd crush rule dump | grep -A15 "ec-data"
```

## Modifying Profiles

Profiles cannot be modified after pools are created using them. Create a new profile instead:

```bash
# Create a corrected profile
ceph osd erasure-code-profile set ec-rack-v2-profile \
  k=4 \
  m=2 \
  crush-failure-domain=rack \
  crush-device-class=hdd

# Migrate the pool to the new profile (requires new pool creation and data migration)
# 1. Create new pool with correct profile
ceph osd pool create ec-data-v2 128 128 erasure ec-rack-v2-profile

# 2. Copy data using rados or application-level migration
# 3. Delete old pool after verification
```

## Deleting Profiles

```bash
# Delete an unused erasure code profile
ceph osd erasure-code-profile rm old-profile

# Note: profiles in use by pools cannot be deleted
ceph osd pool get ec-data erasure_code_profile
```

## Summary

Erasure code profile CRUSH properties control how chunks are distributed across the storage topology. Set `crush-failure-domain` to match your desired fault boundary (host, rack, or datacenter), use `crush-device-class` to restrict placement to a specific drive tier, and choose `crush-root` if you have multiple CRUSH roots. These properties are baked in at profile creation time and generate the associated CRUSH rule automatically when a pool is created.
