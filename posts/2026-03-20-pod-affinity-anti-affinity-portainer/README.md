# Pod Affinity and Anti-Affinity in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Pod Affinity, Scheduling, Reliability

Description: Learn how to configure Kubernetes pod affinity and anti-affinity rules through Portainer to control pod placement for high availability and performance.

## What are Pod Affinity and Anti-Affinity?

Kubernetes scheduler uses affinity and anti-affinity rules to influence where pods are placed relative to other pods:

- **Pod Affinity** - Schedule a pod on the same node (or zone) as another pod with a matching label
- **Pod Anti-Affinity** - Schedule a pod away from nodes already running pods with a matching label

These rules are critical for:
- **High availability**: Spreading replicas across failure domains
- **Performance**: Co-locating tightly coupled services
- **Resource isolation**: Keeping resource-intensive workloads apart

## Configuring via Portainer

In Portainer, navigate to your Kubernetes cluster → **Applications** → create or edit a deployment. In the **Advanced settings** section, you can paste a full YAML spec including affinity rules.

## Pod Anti-Affinity for High Availability

Spread replicas of the same application across different nodes:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - web-app
              topologyKey: kubernetes.io/hostname
      containers:
        - name: web
          image: myapp/web:v2.0.0
```

This ensures no two `web-app` pods land on the same node.

## Preferred Anti-Affinity (Soft Rule)

Use preferred rules when hard constraints would prevent scheduling:

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: web-app
          topologyKey: topology.kubernetes.io/zone
```

Weight ranges from 1–100; higher weights are preferred more strongly.

## Pod Affinity for Co-Location

Co-locate a caching layer with the application that uses it:

```yaml
affinity:
  podAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: my-app
          topologyKey: kubernetes.io/hostname
```

This places the cache pod on the same node as `my-app` pods when possible.

## Topology Keys

Common `topologyKey` values:

| Key | Scope |
|---|---|
| `kubernetes.io/hostname` | Same node |
| `topology.kubernetes.io/zone` | Same availability zone |
| `topology.kubernetes.io/region` | Same region |

## Spreading Across Zones

For zone-level HA, combine anti-affinity with topology spread constraints:

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: web-app
```

## Applying in Portainer

1. Navigate to **Kubernetes → Applications**
2. Click **Add application** or edit an existing one
3. Switch to **Advanced** mode
4. Paste the full YAML with the `affinity` section
5. Click **Deploy** or **Update**

Alternatively, use Portainer's **Manifests** (GitOps) feature to manage YAML from a Git repository.

## Troubleshooting

**Pods stuck in Pending:**
```bash
kubectl describe pod <pod-name> | grep -A10 Events
```
Look for `FailedScheduling` events indicating affinity constraints cannot be satisfied.

**Check node labels:**
```bash
kubectl get nodes --show-labels
```

**Simulate scheduling:**
```bash
kubectl describe pod <pending-pod>
```

## Best Practices

1. **Use `required` anti-affinity** for critical HA requirements; `preferred` for best-effort spreading
2. **Match topology keys** to your actual infrastructure topology (zones, nodes)
3. **Combine with topology spread constraints** for more granular spreading control
4. **Test with fewer replicas than nodes** first to verify constraints are satisfiable
5. **Monitor pod distribution** to ensure rules are being honored

## Conclusion

Pod affinity and anti-affinity rules give you precise control over Kubernetes scheduling behavior. Applied through Portainer's YAML editor or manifest feature, they help you build highly available and well-distributed workloads without complex manual node management.
