# How to Configure ClusterIP Services in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, ClusterIP

Description: Learn how to configure ClusterIP services in Rancher for internal communication between Kubernetes workloads.

ClusterIP is the default Kubernetes service type that provides internal-only load balancing within your cluster. It assigns a virtual IP address accessible only from within the cluster, making it ideal for inter-service communication. This guide explains how to configure and manage ClusterIP services in Rancher.

## Prerequisites

- A running Rancher instance (v2.6 or later)
- A managed Kubernetes cluster
- kubectl access to your cluster
- At least one deployed application

## Understanding ClusterIP Services

A ClusterIP service receives a stable virtual IP address within the cluster's service CIDR range. Other pods can reach the service using this IP or the DNS name. The service load-balances traffic across all pods matching its selector.

## Step 1: Create a ClusterIP Service via kubectl

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: default
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
```

```bash
kubectl apply -f backend-service.yaml
```

Since ClusterIP is the default type, you can omit `type: ClusterIP` and it will be assigned automatically.

## Step 2: Create a ClusterIP Service via the Rancher UI

1. Navigate to your cluster in the Rancher dashboard.
2. Go to **Service Discovery** > **Services**.
3. Click **Create**.
4. Select **ClusterIP** as the service type.
5. Fill in the details:
   - **Name**: `backend-service`
   - **Namespace**: `default`
   - **Selectors**: Add `app = backend`
6. Configure port mappings:
   - **Service Port**: `80`
   - **Target Port**: `8080`
   - **Protocol**: `TCP`
7. Click **Create**.

## Step 3: Access the Service from Other Pods

ClusterIP services are accessible via DNS within the cluster. The DNS format is:

```xml
<service-name>.<namespace>.svc.cluster.local
```

Test connectivity from another pod:

```bash
kubectl run test-pod --image=busybox --rm -it -- sh

# Inside the pod
wget -qO- http://backend-service.default.svc.cluster.local
wget -qO- http://backend-service.default
wget -qO- http://backend-service
```

## Step 4: Configure a Headless Service

A headless service (ClusterIP set to None) does not perform load balancing. Instead, DNS returns the IP addresses of all individual pods:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: headless-backend
  namespace: default
spec:
  clusterIP: None
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080
```

This is useful for stateful applications where clients need to connect to specific pods, such as databases or message queues.

```bash
# DNS lookup returns all pod IPs
kubectl run test --image=busybox --rm -it -- nslookup headless-backend.default.svc.cluster.local
```

## Step 5: Configure Multiple Ports

Expose multiple ports through a single service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: multi-port-backend
  namespace: default
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: grpc
    port: 9090
    targetPort: 9090
  - name: metrics
    port: 9100
    targetPort: 9100
```

When using multiple ports, each port must have a unique name.

## Step 6: Use a Static ClusterIP

If you need a predictable IP address, specify it explicitly:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: static-ip-service
  namespace: default
spec:
  type: ClusterIP
  clusterIP: 10.43.100.50
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080
```

The IP must be within the cluster's service CIDR range and not already assigned. Check the range:

```bash
kubectl cluster-info dump | grep service-cluster-ip-range
```

## Step 7: Create an ExternalName Service

An ExternalName service maps a service to an external DNS name:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
  namespace: default
spec:
  type: ExternalName
  externalName: db.external-provider.com
```

Pods can now reach the external database using `external-db.default.svc.cluster.local`, which resolves to `db.external-provider.com`.

## Step 8: Configure Service with Endpoint Slices

For services that need to point to external IPs without selectors:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service
  namespace: default
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: external-service-1
  namespace: default
  labels:
    kubernetes.io/service-name: external-service
spec:
  addressType: IPv4
  ports:
    - port: 80
  endpoints:
    - addresses:
      - "192.168.1.100"
    - addresses:
      - "192.168.1.101"
```

## Step 9: Monitor Service Health

Verify the service and its endpoints:

```bash
kubectl get svc backend-service -n default
kubectl describe svc backend-service -n default
kubectl get endpoints backend-service -n default
kubectl get endpointslices -l kubernetes.io/service-name=backend-service -n default
```

Check if traffic is reaching pods:

```bash
kubectl logs -l app=backend --tail=20
```

## Troubleshooting

- **No endpoints**: Verify pod labels match the service selector: `kubectl get pods --show-labels`
- **DNS not resolving**: Check CoreDNS is running: `kubectl get pods -n kube-system -l k8s-app=kube-dns`
- **Connection refused**: Verify targetPort matches the container's listening port
- **Intermittent failures**: Check pod readiness probes and ensure pods are in Ready state
- **Wrong IP**: Delete and recreate the service if the ClusterIP needs to change

## Summary

ClusterIP services are the foundation of internal communication in Kubernetes clusters managed by Rancher. They provide stable DNS names and IP addresses for service discovery, built-in load balancing across pods, and support for advanced patterns like headless services and external name mappings. Understanding ClusterIP services is essential for building microservice architectures on Rancher.
