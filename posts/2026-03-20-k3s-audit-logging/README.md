# How to Configure K3s Audit Logging

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: k3s, Kubernetes, Security, Audit Logging, Compliance, DevOps

Description: Learn how to enable and configure Kubernetes audit logging in K3s to track API server activity for security compliance and incident investigation.

## Introduction

Kubernetes audit logging records all API server requests - who did what, to which resource, and when. This is essential for security investigations, compliance requirements (SOC 2, PCI-DSS, HIPAA), and understanding cluster activity. K3s supports the full Kubernetes audit logging framework through kube-apiserver flags. This guide covers enabling and configuring comprehensive audit logging.

## Understanding Audit Log Levels

Kubernetes audit logging has four verbosity levels:
- **None**: Don't log this request
- **Metadata**: Log request metadata (user, resource, verb) but not the body
- **Request**: Log metadata and the request body
- **RequestResponse**: Log metadata, request body, and response body

## Prerequisites

- Running K3s server with root access
- Sufficient disk space for audit logs
- Understanding of your compliance requirements

## Step 1: Create an Audit Policy File

The audit policy defines which requests to log and at what level:

```yaml
# /etc/rancher/k3s/audit-policy.yaml

apiVersion: audit.k8s.io/v1
kind: Policy
# Don't log requests to certain non-sensitive endpoints
omitStages:
  - "RequestReceived"

rules:
  # Log secret access at RequestResponse level (security-sensitive)
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["secrets"]

  # Log pod exec and port-forward at RequestResponse level
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["pods/exec", "pods/portforward", "pods/attach"]

  # Log all authentication requests
  - level: Metadata
    nonResourceURLs:
      - "/api*"
      - "/version"

  # Log RBAC changes at RequestResponse level
  - level: RequestResponse
    resources:
      - group: "rbac.authorization.k8s.io"
        resources:
          - "clusterroles"
          - "clusterrolebindings"
          - "roles"
          - "rolebindings"

  # Log all namespace-scoped resource modifications
  - level: Request
    verbs: ["create", "update", "patch", "delete"]
    resources:
      - group: ""
        resources:
          - "configmaps"
          - "pods"
          - "services"
          - "persistentvolumeclaims"

  # Log metadata for read operations
  - level: Metadata
    verbs: ["get", "list", "watch"]

  # Skip logging for known system/health check paths
  - level: None
    nonResourceURLs:
      - "/healthz*"
      - "/readyz*"
      - "/livez*"

  # Skip noisy system components
  - level: None
    users:
      - "system:kube-scheduler"
      - "system:kube-proxy"
    verbs: ["get", "list", "watch"]

  # Default: log everything else at Metadata level
  - level: Metadata
```

## Step 2: Configure K3s to Enable Audit Logging

```yaml
# /etc/rancher/k3s/config.yaml
kube-apiserver-arg:
  # Path for audit log file
  - "audit-log-path=/var/log/k3s/audit.log"
  # Keep 30 days of logs
  - "audit-log-maxage=30"
  # Keep max 10 rotated log files
  - "audit-log-maxbackup=10"
  # Rotate when log exceeds 100MB
  - "audit-log-maxsize=100"
  # Path to audit policy
  - "audit-policy-file=/etc/rancher/k3s/audit-policy.yaml"
```

Create the log directory:

```bash
mkdir -p /var/log/k3s
chmod 700 /var/log/k3s
```

Restart K3s to apply:

```bash
systemctl restart k3s
```

## Step 3: Verify Audit Logging is Working

```bash
# Check the audit log is being created
ls -la /var/log/k3s/audit.log

# Make a test API call
kubectl get pods -A

# View the audit log entries (JSON format)
tail -f /var/log/k3s/audit.log | python3 -m json.tool

# Filter for specific user
cat /var/log/k3s/audit.log | \
  python3 -c "import sys,json; [print(json.dumps(json.loads(l), indent=2))
  for l in sys.stdin if 'admin' in l]"
```

## Step 4: Parse Audit Log Entries

Each audit log entry is a JSON object:

```json
{
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1",
  "level": "RequestResponse",
  "auditID": "abc-123-def",
  "stage": "ResponseComplete",
  "requestURI": "/api/v1/namespaces/default/pods",
  "verb": "create",
  "user": {
    "username": "admin",
    "groups": ["system:masters", "system:authenticated"]
  },
  "sourceIPs": ["192.168.1.100"],
  "userAgent": "kubectl/v1.29.3",
  "objectRef": {
    "resource": "pods",
    "namespace": "default",
    "name": "my-pod",
    "apiVersion": "v1"
  },
  "responseStatus": {
    "code": 201
  },
  "requestReceivedTimestamp": "2024-03-15T10:30:00Z",
  "stageTimestamp": "2024-03-15T10:30:00.150Z"
}
```

## Step 5: Parse and Query Audit Logs with jq

```bash
# Install jq for JSON processing
apt-get install -y jq

# View all Secret access events
cat /var/log/k3s/audit.log | \
  jq 'select(.objectRef.resource == "secrets")'

# Find all pod exec events (potential security events)
cat /var/log/k3s/audit.log | \
  jq 'select(.requestURI | contains("/exec"))'

# Find events by specific user
cat /var/log/k3s/audit.log | \
  jq 'select(.user.username == "admin") | {time: .requestReceivedTimestamp, verb: .verb, resource: .objectRef.resource, name: .objectRef.name}'

# Count events by verb
cat /var/log/k3s/audit.log | \
  jq -r '.verb' | sort | uniq -c | sort -rn

# Find failed requests (non-2xx responses)
cat /var/log/k3s/audit.log | \
  jq 'select(.responseStatus.code >= 400) | {user: .user.username, verb: .verb, resource: .objectRef.resource, code: .responseStatus.code}'
```

## Step 6: Ship Audit Logs to a SIEM

Forward audit logs to a central logging system:

```yaml
# /var/lib/rancher/k3s/server/manifests/fluentd-audit.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-audit
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: fluentd-audit
  template:
    metadata:
      labels:
        app: fluentd-audit
    spec:
      # Only run on server nodes
      nodeSelector:
        node-role.kubernetes.io/control-plane: "true"
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          effect: NoSchedule
      containers:
        - name: fluentd
          image: fluent/fluentd-kubernetes-daemonset:v1-debian-elasticsearch
          env:
            - name: FLUENT_ELASTICSEARCH_HOST
              value: "elasticsearch.monitoring.svc.cluster.local"
            - name: FLUENT_ELASTICSEARCH_PORT
              value: "9200"
          volumeMounts:
            - name: audit-logs
              mountPath: /var/log/k3s
              readOnly: true
      volumes:
        - name: audit-logs
          hostPath:
            path: /var/log/k3s
            type: DirectoryOrCreate
```

## Step 7: Set Up Log Rotation

```bash
# Configure logrotate for K3s audit logs
cat > /etc/logrotate.d/k3s-audit << 'EOF'
/var/log/k3s/audit.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    create 0600 root root
    postrotate
        systemctl reload k3s 2>/dev/null || true
    endscript
}
EOF
```

## Conclusion

Audit logging in K3s provides a complete record of all API server activity, which is essential for security investigations and compliance. Start with a policy that captures security-sensitive operations (secret access, RBAC changes, pod exec) at high verbosity and other operations at lower levels to balance completeness with storage costs. For production environments, always ship audit logs to a central SIEM for long-term retention, alerting, and forensic analysis.
