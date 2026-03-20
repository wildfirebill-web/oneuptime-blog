# How to Expose Portainer on Kubernetes via NodePort

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, NodePort, Networking, Installation

Description: Learn how to expose Portainer on Kubernetes using NodePort so you can access it from outside the cluster.

## What Is NodePort?

A Kubernetes NodePort service exposes an application on a static port on every node in the cluster. Any traffic to `<NodeIP>:<NodePort>` is forwarded to the service. NodePort values are in the range `30000–32767`.

This is the simplest way to expose Portainer without a cloud load balancer.

## Deploying Portainer with NodePort via Helm

```bash
# Install Portainer with NodePort service type
helm install portainer portainer/portainer \
  --namespace portainer \
  --create-namespace \
  --set service.type=NodePort \
  --set service.nodePort=30777 \
  --set service.httpsNodePort=30779
```

## Deploying Portainer with a Manifest (NodePort)

If you prefer to use a raw Kubernetes manifest:

```yaml
# portainer-nodeport.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: portainer
  namespace: portainer
spec:
  replicas: 1
  selector:
    matchLabels:
      app: portainer
  template:
    metadata:
      labels:
        app: portainer
    spec:
      containers:
        - name: portainer
          image: portainer/portainer-ce:latest
          ports:
            - containerPort: 9000
            - containerPort: 9443
            - containerPort: 8000
          volumeMounts:
            - name: portainer-data
              mountPath: /data
      volumes:
        - name: portainer-data
          persistentVolumeClaim:
            claimName: portainer-data-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: portainer
  namespace: portainer
spec:
  type: NodePort
  selector:
    app: portainer
  ports:
    - name: http
      port: 9000
      targetPort: 9000
      nodePort: 30777  # Fixed NodePort for predictable access
    - name: https
      port: 9443
      targetPort: 9443
      nodePort: 30779
    - name: edge
      port: 8000
      targetPort: 8000
      nodePort: 30778
```

```bash
# Apply the manifest
kubectl apply -f portainer-nodeport.yaml
```

## Finding the Node IP

```bash
# Get node IPs
kubectl get nodes -o wide

# Access Portainer at:
# http://<any-node-ip>:30777
```

## Changing the NodePort on an Existing Installation

```bash
# Patch the service to change the NodePort
kubectl patch service portainer \
  --namespace portainer \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/ports/0/nodePort", "value": 30800}]'
```

## Security Consideration

NodePort exposes the service on all nodes. Restrict access with firewall rules:

```bash
# Example: Allow only specific IPs to reach NodePort 30777
# Using iptables (adjust for your firewall tool)
iptables -A INPUT -p tcp --dport 30777 -s 203.0.113.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 30777 -j DROP
```

## Conclusion

NodePort is the quickest way to expose Portainer on Kubernetes without requiring a cloud load balancer. It's ideal for on-premises clusters, home labs, and development environments. For production, consider using a LoadBalancer or Ingress instead.
