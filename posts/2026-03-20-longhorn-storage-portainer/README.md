# How to Deploy Longhorn Storage and Manage via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, Portainer, Kubernetes, Storage, Persistent Volume, DevOps

Description: Learn how to deploy Longhorn distributed block storage on Kubernetes and manage persistent volumes through Portainer's Kubernetes interface.

---

Longhorn is a cloud-native distributed block storage system for Kubernetes that runs entirely inside your cluster. It provides replicated persistent volumes with snapshots, backups, and a built-in UI for storage management. This guide covers deploying Longhorn via Portainer and using it as the default storage class for your workloads.

---

## What Longhorn Provides

- Replicated persistent volumes across nodes
- Automatic recovery when nodes fail
- Volume snapshots and scheduled backups
- Built-in Longhorn UI for storage management
- Compatible with any standard Kubernetes PersistentVolumeClaim

---

## Step 1: Verify Prerequisites on All Nodes

Longhorn requires `open-iscsi` on every node.

```bash
# Run on every Kubernetes node

sudo apt update && sudo apt install -y open-iscsi

# Enable and start iscsid
sudo systemctl enable --now iscsid

# Verify
sudo systemctl status iscsid

# Check Longhorn readiness (run on any node with kubectl)
curl -sSfL https://raw.githubusercontent.com/longhorn/longhorn/v1.6.0/scripts/environment_check.sh | bash
```

---

## Step 2: Deploy Longhorn via Portainer

In Portainer, navigate to your Kubernetes environment and deploy Longhorn as a stack.

```yaml
# longhorn-namespace.yaml - create the namespace first
apiVersion: v1
kind: Namespace
metadata:
  name: longhorn-system
```

Then deploy Longhorn using the official manifest:

```bash
# Apply the Longhorn manifest from within Portainer or via kubectl
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.6.0/deploy/longhorn.yaml

# Watch the deployment progress
kubectl get pods -n longhorn-system -w
```

Or in Portainer:
1. Go to **Stacks > Add Stack**
2. Select **URL** as the source
3. Enter: `https://raw.githubusercontent.com/longhorn/longhorn/v1.6.0/deploy/longhorn.yaml`
4. Click **Deploy the stack**

---

## Step 3: Expose the Longhorn UI via Portainer

Create an ingress or port-forward to access the Longhorn UI.

```yaml
# longhorn-ingress.yaml - expose Longhorn UI
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: longhorn-ingress
  namespace: longhorn-system
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
spec:
  ingressClassName: nginx
  rules:
    - host: longhorn.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: longhorn-frontend
                port:
                  number: 80
```

---

## Step 4: Set Longhorn as Default Storage Class

```bash
# Make Longhorn the default storage class
kubectl patch storageclass longhorn \
  -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "true"}}}'

# Remove the default annotation from any other storage class
kubectl patch storageclass local-path \
  -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "false"}}}'

# Verify
kubectl get storageclass
```

---

## Step 5: Deploy an Application with a Longhorn PVC in Portainer

Now deploy an application through Portainer that uses Longhorn for persistent storage.

```yaml
# app-with-longhorn-pvc.yaml - deploy via Portainer stack
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myapp-data
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 10Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myapp:latest
          volumeMounts:
            - name: data
              mountPath: /app/data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: myapp-data
```

---

## Managing Longhorn via Portainer

In Portainer's Kubernetes interface:
- **Volumes** → shows all Longhorn PVCs
- **Cluster > Storage** → shows storage classes and PVs
- The Longhorn UI provides deeper operations: snapshots, backups, replica management

---

## Summary

Longhorn provides production-grade distributed storage for Kubernetes with minimal setup. Deploy it via Portainer's stack interface, set it as the default storage class, and all PVCs automatically use replicated Longhorn volumes. The Longhorn UI offers snapshot and backup management beyond what Portainer exposes natively.
