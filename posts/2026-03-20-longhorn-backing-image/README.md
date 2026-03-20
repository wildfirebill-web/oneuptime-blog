# How to Configure Longhorn Backing Image - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, Backing Image, Kubernetes, VM Image, Storage, Template Volumes, SUSE Rancher

Description: Learn how to configure Longhorn backing images to pre-populate volumes with base images such as OS images for virtual machines or base data for stateful applications.

---

Longhorn backing images allow you to create volumes pre-populated with a base image, such as a Linux OS image for a virtual machine (Harvester VMs) or a database seed image. This eliminates the need to manually copy data into new volumes.

---

## What Is a Backing Image?

A Longhorn backing image is an immutable base image stored as a set of replicas in the cluster. When a volume is created with a backing image, it uses copy-on-write semantics - the base image is shared, and only changes are stored per-volume.

---

## Step 1: Create a Backing Image from a URL

```yaml
# backing-image-ubuntu.yaml

apiVersion: longhorn.io/v1beta2
kind: BackingImage
metadata:
  name: ubuntu-22-04
  namespace: longhorn-system
spec:
  sourceType: download
  sourceParameters:
    url: "https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img"
  checksum: "sha512:abc123..."  # Optional but recommended
  minNumberOfCopies: 2
  diskSelector: []
  nodeSelector: []
```

```bash
kubectl apply -f backing-image-ubuntu.yaml

# Monitor download progress
kubectl get lhbackingimage ubuntu-22-04 -n longhorn-system -w
```

---

## Step 2: Create a Backing Image from a Local File

Upload an image directly from a node:

```bash
# Via the Longhorn UI: Backing Image > Create > Upload
# The UI supports uploading .img, .qcow2, .iso files

# Or via API
LONGHORN_URL=http://longhorn-frontend.longhorn-system.svc.cluster.local
curl -X POST \
  ${LONGHORN_URL}/v1/backingimages \
  -H "Content-Type: application/json" \
  -d '{
    "name": "my-db-seed",
    "expectedChecksum": "",
    "sourceType": "upload"
  }'
```

---

## Step 3: Create a Volume Using a Backing Image

```yaml
# StorageClass with backing image reference
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-ubuntu-vm
provisioner: driver.longhorn.io
parameters:
  numberOfReplicas: "2"
  backingImage: ubuntu-22-04
  backingImageURL: ""  # Leave empty when using name reference
  backingImageChecksum: ""
```

```yaml
# PVC that will be pre-populated with the Ubuntu image
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ubuntu-vm-disk
  namespace: vms
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn-ubuntu-vm
  resources:
    requests:
      storage: 20Gi
```

---

## Step 4: Manage Backing Image Replicas

```bash
# Check how many copies are ready
kubectl get lhbackingimage ubuntu-22-04 -n longhorn-system \
  -o jsonpath='{.status.diskFileStatusMap}'

# Delete a backing image (only if no volumes are using it)
kubectl delete lhbackingimage ubuntu-22-04 -n longhorn-system
```

---

## Step 5: Backing Image in Harvester

Harvester (Rancher's HCI platform) uses Longhorn backing images for VM disk templates. Import images through the Harvester UI under **Images > Import**. Harvester handles the BackingImage creation automatically.

---

## Best Practices

- Set `minNumberOfCopies: 2` for production backing images to ensure availability during node failures.
- Use checksums to verify backing image integrity after download.
- Reuse backing images across multiple volumes to reduce storage consumption - the base data is shared via copy-on-write.
- Regularly clean up unused backing images to free storage capacity.
