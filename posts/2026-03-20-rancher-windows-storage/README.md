# How to Configure Windows Storage in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Windows, Storage, CSI, PersistentVolume

Description: Configure persistent storage for Windows containers in Rancher using local-path provisioner, iSCSI, and Windows-specific CSI drivers for stateful Windows workloads.

## Introduction

Storage for Windows containers in Kubernetes has specific requirements and limitations compared to Linux. Volume mounts use Windows paths, certain storage types are not supported, and CSI drivers must have Windows-compatible components. This guide covers configuring persistent storage for Windows workloads in Rancher.

## Prerequisites

- Rancher cluster with Windows worker nodes
- Storage infrastructure (local disk, NFS, iSCSI, or cloud storage)
- kubectl access with storage admin permissions

## Step 1: Understand Windows Volume Limitations

```text
Windows Container Storage Limitations:
- Volume mounts use Windows paths (C:\mountpath)
- No support for file permissions/ownership (chmod/chown)
- NFS has limited support on Windows Server
- Memory-backed volumes (emptyDir medium: Memory) not supported
- ReadWriteMany not supported for local storage
- Some StorageClass features differ from Linux

Supported volume types:
- emptyDir (disk-backed)
- hostPath (SMB/CIFS)
- Local path provisioner
- AWS EBS (via Windows CSI driver)
- Azure Disk (via Windows CSI driver)
- iSCSI (with Windows initiator)
```

## Step 2: Configure Local Path Provisioner for Windows

```bash
# Install local path provisioner with Windows support

kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.26/deploy/local-path-storage.yaml

# Create Windows-specific storage class
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path-windows
provisioner: rancher.io/local-path
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
EOF
```

```yaml
# Update local-path-config for Windows paths
apiVersion: v1
kind: ConfigMap
metadata:
  name: local-path-config
  namespace: local-path-storage
data:
  config.json: |-
    {
      "nodePathMap": [
        {
          "node": "DEFAULT_PATH_FOR_NON_LISTED_NODES",
          "paths": ["/opt/local-path-provisioner"]
        },
        {
          "node": "win-node-01",
          "paths": ["C:\\kubernetes\\storage"]
        },
        {
          "node": "win-node-02",
          "paths": ["C:\\kubernetes\\storage"]
        }
      ]
    }
```

## Step 3: Create PVC for Windows Pod

```yaml
# windows-pvc.yaml - PVC for Windows workload
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: windows-app-data
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path-windows
  resources:
    requests:
      storage: 20Gi
---
# windows-statefulset.yaml - Use PVC in Windows pod
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: sql-server-express
  namespace: production
spec:
  serviceName: sql-server
  replicas: 1
  selector:
    matchLabels:
      app: sql-server
  template:
    metadata:
      labels:
        app: sql-server
    spec:
      nodeSelector:
        kubernetes.io/os: windows
      tolerations:
        - key: os
          value: windows
          effect: NoSchedule
      containers:
        - name: sql-server
          image: mcr.microsoft.com/mssql/server:2022-latest
          env:
            - name: ACCEPT_EULA
              value: "Y"
            - name: MSSQL_SA_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: sql-server-secret
                  key: sa-password
            - name: MSSQL_DATA_DIR
              value: "C:\\sqldata"
          volumeMounts:
            - name: sqldata
              mountPath: C:\sqldata
          resources:
            requests:
              cpu: 2000m
              memory: 4Gi
            limits:
              cpu: 4000m
              memory: 8Gi
  volumeClaimTemplates:
    - metadata:
        name: sqldata
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: local-path-windows
        resources:
          requests:
            storage: 50Gi
```

## Step 4: Configure AWS EBS for Windows (Windows CSI Driver)

```bash
# Install Windows CSI driver for EBS
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.x"

# Verify Windows node support
kubectl get pods -n kube-system | grep ebs-csi
kubectl get daemonset ebs-csi-node-windows -n kube-system
```

```yaml
# windows-ebs-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3-windows
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  # Windows requires NTFS filesystem
  csi.storage.k8s.io/fstype: ntfs
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

## Step 5: Configure SMB/CIFS Volumes

```yaml
# SMB volume for shared Windows storage
# Requires csi-driver-smb installed

# Install SMB CSI driver
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/csi-driver-smb/master/deploy/install-driver.sh

# Create SMB credentials secret
kubectl create secret generic smb-credentials \
  --from-literal=username=smbuser \
  --from-literal=password=smbpassword \
  -n production

# Create PV for SMB share
apiVersion: v1
kind: PersistentVolume
metadata:
  name: smb-shared-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  csi:
    driver: smb.csi.k8s.io
    readOnly: false
    volumeHandle: unique-volumeid
    volumeAttributes:
      source: "//fileserver.example.com/shared"
    nodeStageSecretRef:
      name: smb-credentials
      namespace: production
```

## Step 6: hostPath Volume for Windows

```yaml
# Use hostPath for direct access to Windows host filesystem
apiVersion: apps/v1
kind: Deployment
metadata:
  name: windows-log-agent
  namespace: production
spec:
  template:
    spec:
      nodeSelector:
        kubernetes.io/os: windows
      containers:
        - name: log-agent
          image: registry.example.com/windows-log-agent:v1.0
          volumeMounts:
            - name: windows-logs
              mountPath: C:\logs
              readOnly: true
      volumes:
        - name: windows-logs
          hostPath:
            path: C:\Windows\System32\winevt\Logs
            type: Directory
```

## Conclusion

Windows storage in Kubernetes requires careful CSI driver selection and volume path configuration. For simple development workloads, the local path provisioner with Windows-specific paths works well. For production, AWS EBS with the Windows CSI driver provides reliable block storage with NTFS formatting. SMB shares via the SMB CSI driver enable shared storage across multiple Windows pods (ReadWriteMany), which is essential for applications that require shared file system access. Always specify Windows-compatible mount paths using Windows-style paths (`C:\data`) in volume mounts.
