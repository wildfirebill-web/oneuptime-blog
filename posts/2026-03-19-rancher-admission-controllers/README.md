# How to Configure Admission Controllers in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Security

Description: Learn how to configure Kubernetes admission controllers in Rancher-managed clusters to enforce policies and validate resource configurations.

Admission controllers intercept requests to the Kubernetes API server before resources are persisted. They can validate, mutate, or reject requests based on policies. Rancher-managed clusters support both built-in and custom admission controllers. This guide covers configuring them for security enforcement.

## Prerequisites

- Rancher v2.5 or later
- RKE or RKE2 managed clusters
- Admin access to Rancher
- kubectl access with cluster admin privileges

## Step 1: Understand Admission Controller Types

Kubernetes has two types of admission controllers:

- **Validating**: Check requests and accept or reject them. They cannot modify the request.
- **Mutating**: Modify requests before they are persisted. For example, adding default labels or injecting sidecar containers.

Controllers run in order: mutating first, then validating.

## Step 2: View Enabled Admission Controllers

Check which admission controllers are currently enabled:

```bash
kubectl -n kube-system describe pod kube-apiserver-NODE | grep enable-admission
```

For RKE2:

```bash
ps aux | grep kube-apiserver | tr ' ' '\n' | grep admission
```

Default enabled controllers in RKE2 include: NamespaceLifecycle, LimitRanger, ServiceAccount, DefaultStorageClass, DefaultTolerationSeconds, MutatingAdmissionWebhook, ValidatingAdmissionWebhook, ResourceQuota, PodSecurityAdmission, and NodeRestriction.

## Step 3: Enable Additional Built-in Controllers

For RKE2 clusters, add admission controllers via the config:

```yaml
# /etc/rancher/rke2/config.yaml

kube-apiserver-arg:
  - "enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota,PodSecurityAdmission,NodeRestriction,AlwaysPullImages,DenyServiceExternalIPs"
```

Restart the server:

```bash
systemctl restart rke2-server
```

For RKE clusters via Rancher:

```yaml
rancher_kubernetes_engine_config:
  services:
    kube_api:
      extra_args:
        enable-admission-plugins: "NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota,AlwaysPullImages"
```

## Step 4: Configure AlwaysPullImages

The AlwaysPullImages controller forces every pod to pull its image from the registry on every start. This prevents using cached images that might have been pulled with different credentials:

This is enabled by adding `AlwaysPullImages` to the admission plugins list as shown in Step 3. No additional configuration is needed.

## Step 5: Create a Validating Webhook

Deploy a custom validating webhook to enforce your own policies. Here is an example that rejects pods without resource limits:

First, create the webhook service:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-limit-validator
  namespace: kube-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app: resource-limit-validator
  template:
    metadata:
      labels:
        app: resource-limit-validator
    spec:
      containers:
      - name: validator
        image: your-registry/resource-limit-validator:latest
        ports:
        - containerPort: 8443
        volumeMounts:
        - name: tls
          mountPath: /etc/webhook/certs
          readOnly: true
      volumes:
      - name: tls
        secret:
          secretName: resource-limit-validator-tls
---
apiVersion: v1
kind: Service
metadata:
  name: resource-limit-validator
  namespace: kube-system
spec:
  selector:
    app: resource-limit-validator
  ports:
  - port: 443
    targetPort: 8443
```

Register the webhook:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: resource-limit-validator
webhooks:
- name: resource-limits.example.com
  clientConfig:
    service:
      name: resource-limit-validator
      namespace: kube-system
      path: /validate
    caBundle: BASE64_CA_BUNDLE
  rules:
  - operations: ["CREATE", "UPDATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
  admissionReviewVersions: ["v1"]
  sideEffects: None
  failurePolicy: Fail
  namespaceSelector:
    matchExpressions:
    - key: kubernetes.io/metadata.name
      operator: NotIn
      values:
      - kube-system
      - cattle-system
```

## Step 6: Create a Mutating Webhook

Mutating webhooks can automatically modify resources. A common use case is injecting default labels:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: default-labels-injector
webhooks:
- name: labels.example.com
  clientConfig:
    service:
      name: label-injector
      namespace: kube-system
      path: /mutate
    caBundle: BASE64_CA_BUNDLE
  rules:
  - operations: ["CREATE"]
    apiGroups: ["apps"]
    apiVersions: ["v1"]
    resources: ["deployments"]
  admissionReviewVersions: ["v1"]
  sideEffects: None
  failurePolicy: Ignore
  namespaceSelector:
    matchExpressions:
    - key: kubernetes.io/metadata.name
      operator: NotIn
      values:
      - kube-system
```

## Step 7: Configure ResourceQuota Admission Controller

Enforce resource quotas per namespace:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
    services: "20"
    secrets: "100"
    configmaps: "100"
```

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
  - default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    type: Container
```

Apply both:

```bash
kubectl apply -f resource-quota.yaml
kubectl apply -f limit-range.yaml
```

## Step 8: Manage Webhooks via Rancher

View and manage admission webhooks through Rancher:

1. Navigate to the cluster.
2. Go to **More Resources** > **Admission**.
3. View MutatingWebhookConfigurations and ValidatingWebhookConfigurations.

You can also edit or delete webhooks from this interface.

## Step 9: Test Admission Controllers

Test that your admission controllers are working:

```bash
# Test ResourceQuota enforcement
kubectl run test-pod --image=nginx -n production --limits="cpu=100,memory=100Gi"
# Should be rejected if it exceeds quota

# Test AlwaysPullImages
kubectl run test-pull --image=nginx -n production --dry-run=server -o yaml | grep imagePullPolicy
# Should show "Always"
```

## Step 10: Monitor Admission Controller Performance

Admission controllers add latency to API requests. Monitor their impact:

```bash
kubectl get --raw /metrics | grep apiserver_admission
```

Key metrics to watch:

- `apiserver_admission_controller_admission_duration_seconds`: Time spent in admission controllers.
- `apiserver_admission_webhook_rejection_count`: Number of rejections by webhooks.
- `apiserver_admission_webhook_fail_open_count`: Failures where the webhook was skipped.

## Troubleshooting

### Webhook Timeout Errors

If webhooks time out, increase the timeout or check the webhook service health:

```bash
kubectl get pods -n kube-system -l app=resource-limit-validator
kubectl logs -n kube-system -l app=resource-limit-validator
```

### Webhook Blocking All Requests

If a webhook blocks all API requests (including system operations), fix the namespaceSelector to exclude system namespaces or temporarily delete the webhook:

```bash
kubectl delete validatingwebhookconfigurations resource-limit-validator
```

### Cannot Create Resources After Enabling Controllers

Check which controller is rejecting the request:

```bash
kubectl apply -f resource.yaml 2>&1
```

The error message will indicate which admission controller rejected the request and why.

## Conclusion

Admission controllers are a powerful mechanism for enforcing security and operational policies in Rancher-managed clusters. By combining built-in controllers like ResourceQuota and AlwaysPullImages with custom validating and mutating webhooks, you can automate policy enforcement and prevent non-compliant resources from being deployed. Always exclude system namespaces from custom webhooks to avoid breaking cluster operations.
