# How to Create Dapr Component Templates for Your Organization

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Component, Template, Organization, Governance, Helm

Description: Build reusable Dapr component templates for your organization using Helm, Kustomize, and Git to standardize component configuration across all teams and environments.

---

## Why Component Templates Matter

Without templates, teams copy-paste component YAML files and inevitably diverge. One team uses different TTL settings, another skips scopes, a third hardcodes passwords. Templates enforce standards at the source and simplify multi-environment management.

## Template Repository Structure

Create a dedicated Git repository for Dapr component templates:

```bash
dapr-components/
├── base/
│   ├── configuration.yaml      # Tracing and metrics config
│   └── resiliency.yaml         # Standard resiliency policy
├── state/
│   ├── redis/
│   │   ├── base.yaml
│   │   └── kustomization.yaml
│   └── postgresql/
│       ├── base.yaml
│       └── kustomization.yaml
├── pubsub/
│   ├── kafka/
│   │   ├── base.yaml
│   │   └── kustomization.yaml
│   └── redis/
│       └── base.yaml
├── secrets/
│   └── kubernetes/
│       └── base.yaml
└── overlays/
    ├── dev/
    ├── staging/
    └── production/
```

## Base Redis State Store Template

```yaml
# state/redis/base.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore         # Teams override this name
  namespace: default       # Teams override this namespace
spec:
  type: state.redis
  version: v1
  initTimeout: 10s
  ignoreErrors: false
  metadata:
    - name: redisHost
      value: REDIS_HOST_PLACEHOLDER
    - name: redisPassword
      secretKeyRef:
        name: redis-secret
        key: password
    - name: ttlInSeconds
      value: "86400"
    - name: enableTLS
      value: "false"
auth:
  secretStore: kubernetes
```

## Using Kustomize for Environment Overlays

Override template values per environment:

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../state/redis/base.yaml
patches:
  - target:
      kind: Component
      name: statestore
    patch: |
      - op: replace
        path: /spec/metadata/0/value
        value: redis-master.production.svc:6379
      - op: add
        path: /spec/scopes
        value:
          - order-service
          - inventory-service
      - op: replace
        path: /spec/metadata/3/value
        value: "true"   # Enable TLS in production
```

Apply with Kustomize:

```bash
kubectl apply -k overlays/production/
```

## Helm-Based Templates for Multi-Team Use

Create a Helm chart for teams to include as a dependency:

```yaml
# charts/dapr-components/templates/statestore.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: {{ .Values.statestore.name | default "statestore" }}
  namespace: {{ .Release.Namespace }}
spec:
  type: state.{{ .Values.statestore.backend | default "redis" }}
  version: v1
  initTimeout: {{ .Values.statestore.initTimeout | default "10s" }}
  metadata:
    - name: redisHost
      value: {{ .Values.statestore.host }}
    - name: redisPassword
      secretKeyRef:
        name: {{ .Values.statestore.secretName }}
        key: password
scopes: {{ .Values.statestore.scopes | toYaml | nindent 4 }}
```

Teams configure with values files:

```yaml
# my-service/values.yaml
statestore:
  name: redis-statestore
  backend: redis
  host: redis-master:6379
  secretName: redis-secret
  initTimeout: 5s
  scopes:
    - my-service
```

## CI/CD Integration

Validate templates in CI before merging:

```bash
#!/bin/bash
# validate-components.sh

for file in $(find . -name "*.yaml" -path "*/components/*"); do
  echo "Validating $file..."
  kubectl apply --dry-run=client -f "$file"
  if [ $? -ne 0 ]; then
    echo "FAIL: $file is invalid"
    exit 1
  fi
done
echo "All component templates are valid"
```

## Summary

Dapr component templates in a dedicated Git repository using Kustomize overlays or Helm charts provide a scalable way to standardize component configuration across multiple teams and environments. Base templates enforce required fields like `initTimeout`, `auth.secretStore`, and `scopes`, while overlays allow environment-specific customization. CI validation of all templates before merging prevents configuration errors from reaching production.
