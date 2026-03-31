# How to Understand Ceph Squid Release Features

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Squid, Release, Features, SMB

Description: Explore the key features in Ceph Squid (v19), including improved SMB support, RGW enhancements, NVMe-oF graduation, and performance improvements.

---

Ceph Squid (v19) is the release that followed Reef (v18), continuing the tradition of naming Ceph releases after ocean creatures. Squid focuses on graduating technology previews from Reef and expanding enterprise capabilities.

## Squid Release Overview

Ceph Squid (v19.x) highlights include:
- SMB gateway graduated from technology preview
- NVMe-oF gateway improvements
- RGW S3 Select improvements
- Enhanced RADOS namespace support
- Improved Rook operator integration
- BlueStore performance tuning improvements

## SMB Gateway Graduation

The SMB/CIFS gateway for CephFS graduated from technology preview to supported in Squid:

```yaml
# Rook now supports SMB gateway as a CRD
apiVersion: ceph.rook.io/v1
kind: CephSMB
metadata:
  name: my-smb
  namespace: rook-ceph
spec:
  gateway:
    instances: 2
    port: 445
  security:
    kerberos:
      enabled: false
  shares:
    - name: data
      cephfs:
        filesystem: myfs
        path: /smb-share
      readOnly: false
```

```bash
# Verify SMB service is running
kubectl -n rook-ceph get pods -l app=rook-ceph-smb

# Test SMB connectivity
smbclient //ceph-smb.example.com/data -U ceph-user
```

## NVMe-oF Gateway Improvements

Squid graduated the NVMe-oF gateway with improved Rook integration:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephNVMEofGateway
metadata:
  name: nvmeof-gw
  namespace: rook-ceph
spec:
  gatewayGroup:
    name: my-gateway-group
  placement:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: role
                operator: In
                values: [storage-nvme]
```

```bash
# List NVMe-oF subsystems
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  nvme list-subsys

# Check gateway status
kubectl -n rook-ceph get CephNVMEofGateway nvmeof-gw -o yaml
```

## Enhanced RADOS Namespace Support

Squid improved namespace isolation for multi-tenant deployments:

```bash
# Create namespaces in a pool
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c "
rados -p mypool --namespace tenant-a put myobj /tmp/testfile
rados -p mypool --namespace tenant-b put myobj /tmp/testfile

# List objects per namespace
rados -p mypool --namespace tenant-a ls
rados -p mypool --namespace tenant-b ls

# Namespace-aware quota
ceph osd pool set-quota mypool max_bytes 10737418240 --namespace tenant-a
"
```

## RGW S3 Select Improvements

```bash
# S3 Select allows querying CSV/JSON data in place
aws s3api select-object-content \
  --bucket my-bucket \
  --key data.csv \
  --expression "SELECT name, age FROM S3Object WHERE age > 30" \
  --expression-type SQL \
  --input-serialization '{"CSV": {"FileHeaderInfo": "USE"}}' \
  --output-serialization '{"CSV": {}}' \
  /dev/stdout \
  --endpoint-url http://rgw.example.com
```

## BlueStore Improvements in Squid

```bash
# New BlueStore allocation statistics
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c "
# Check allocation efficiency
ceph osd df style=terse --format json | python3 -c \"
import sys, json
data = json.load(sys.stdin)
for node in data['nodes']:
    if node['type'] == 'osd':
        print(f'OSD {node[\\\"id\\\"]}: {node.get(\\\"utilization\\\", 0):.1f}% used')
\"
"

# Enable new BlueStore fragmentation handling
ceph config set osd bluestore_fragmentation_check_interval 3600
```

## Upgrading to Squid via Rook

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
spec:
  cephVersion:
    image: quay.io/ceph/ceph:v19.2.0
    allowUnsupported: false
```

```bash
# Pre-upgrade check
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph health detail

# Monitor upgrade
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph versions

# After upgrade, verify all features
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph features
```

## Summary

Ceph Squid (v19) graduates SMB and NVMe-oF gateways from technology preview to supported features, expanding Ceph's protocol coverage beyond S3 and CephFS. The improvements to RADOS namespaces, S3 Select, and BlueStore performance tuning make Squid a compelling upgrade for organizations looking to expand their Ceph cluster's capabilities and improve multi-tenant isolation.
