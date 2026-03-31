# How to Publish Services (ClusterIP, NodePort, LoadBalancer) in Portainer (2)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Service, Networking, ClusterIP, NodePort

Description: Learn how to expose Kubernetes applications using ClusterIP, NodePort, and LoadBalancer service types in Portainer.

## Kubernetes Service Types

| Service Type | Access | Use Case |
|-------------|--------|----------|
| **ClusterIP** | Internal only (within cluster) | Internal microservice communication |
| **NodePort** | External via node IP:port | On-prem, home lab, dev environments |
| **LoadBalancer** | External via cloud load balancer | Production cloud workloads |

## Publishing a Service in Portainer

When deploying or editing an application in Portainer:

1. Scroll to the **Publishing** or **Network ports** section.
2. Select the service type.
3. Map container ports to service ports.

## ClusterIP (Internal Service)

```yaml
# Service accessible only within the cluster

apiVersion: v1
kind: Service
metadata:
  name: my-api
  namespace: production
spec:
  type: ClusterIP      # Default type
  selector:
    app: my-api
  ports:
    - name: http
      port: 80         # Port other services call
      targetPort: 8080 # Container's listening port
```

Other pods in the cluster access it via: `http://my-api.production.svc.cluster.local`

## NodePort (External via Node IP)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: production
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080  # Must be in range 30000-32767
```

Access via: `http://<any-node-ip>:30080`

## LoadBalancer (Cloud Load Balancer)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: production
  annotations:
    # AWS: Use an internal ALB
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internal"
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
```

## Headless Service (for StatefulSets)

```yaml
# Headless service for stable pod DNS names (StatefulSets)
spec:
  type: ClusterIP
  clusterIP: None      # No cluster IP, DNS returns pod IPs directly
  selector:
    app: my-statefulset
```

## Managing Services via CLI

```bash
# List all services in a namespace
kubectl get services --namespace production

# Expose a deployment as a NodePort service
kubectl expose deployment my-app \
  --type=NodePort \
  --port=80 \
  --target-port=8080 \
  --namespace=production

# Get the NodePort assigned
kubectl get service my-app --namespace production \
  -o jsonpath='{.spec.ports[0].nodePort}'
```

## Using Ingress Instead of NodePort/LoadBalancer

For HTTP/HTTPS traffic, Ingress is more flexible than exposing multiple NodePort or LoadBalancer services:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app    # Must point to a ClusterIP service
                port:
                  number: 80
```

## Conclusion

Portainer's service publishing options cover all standard Kubernetes service types. Use ClusterIP for internal services, NodePort for quick external access in on-prem clusters, and LoadBalancer for production cloud deployments.
