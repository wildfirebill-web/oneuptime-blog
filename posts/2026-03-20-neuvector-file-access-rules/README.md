# How to Configure NeuVector File Access Rules

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: NeuVector, File Security, Container Security, Kubernetes, Runtime Security

Description: Configure NeuVector file access rules to monitor and restrict which files and directories containers can read, write, or execute.

## Introduction

File access rules in NeuVector provide runtime file integrity monitoring and access control for containers. They allow you to detect and prevent unauthorized file operations such as writing to system directories, reading sensitive configuration files, or modifying application binaries. This guide walks through setting up comprehensive file access controls.

## Why File Access Rules Are Critical

Attackers who gain container access often:
- Modify application binaries to persist malware
- Read secrets and credentials from the filesystem
- Write web shells to web-accessible directories
- Modify `/etc/passwd` or crontabs for persistence

NeuVector file access rules detect and block these operations in real time.

## Prerequisites

- NeuVector with Enforcer running on all nodes
- Workloads in Discover mode for at least 24 hours
- NeuVector Manager access

## Step 1: Review Auto-Discovered File Access Patterns

After the learning period, review what files your containers access:

```bash
# Get file access profile for a group

curl -sk \
  "https://neuvector-manager:8443/v1/file/profile/group/nv.webapp.default" \
  -H "X-Auth-Token: ${TOKEN}" | jq '.process_profile.process_list'
```

In the NeuVector UI:
1. Go to **Policy** > **Groups**
2. Select a group
3. Click the **File Access** tab

## Step 2: Create File Access Rules via UI

1. Navigate to **Policy** > **Groups** > select your group
2. Click the **File Access** tab
3. Click **Add Rule**
4. Configure the rule:

```text
Filter: /etc/nginx/nginx.conf
Recursive: No
Applications: nginx
Access: Read
Action: Allow
```

## Step 3: Configure File Access Rules via API

```bash
# Add a file access rule to allow nginx to read its config
curl -sk -X POST \
  "https://neuvector-manager:8443/v1/file/profile/group/nv.nginx.default" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "process_profile": {
      "process_list": [
        {
          "filter": "/etc/nginx/",
          "recursive": true,
          "behavior": "monitor change",
          "applications": ["nginx"]
        },
        {
          "filter": "/var/log/nginx/",
          "recursive": true,
          "behavior": "monitor change",
          "applications": ["nginx"]
        },
        {
          "filter": "/usr/share/nginx/html/",
          "recursive": true,
          "behavior": "block access",
          "applications": ["sh", "bash", "curl", "wget"]
        }
      ]
    }
  }'
```

## Step 4: Define File Access Rules as CRDs

```yaml
# file-access-policy.yaml
apiVersion: neuvector.com/v1
kind: NvSecurityRule
metadata:
  name: webapp-file-access
  namespace: default
spec:
  target:
    policymode: Protect
    selector:
      matchLabels:
        app: webapp
  file:
    # Monitor changes to critical config files
    - filter: /etc/passwd
      recursive: false
      behavior: monitor change
    - filter: /etc/shadow
      recursive: false
      behavior: block access
    # Block writes to binary directories
    - filter: /usr/bin/
      recursive: true
      behavior: block access
      applications:
        - deny-write
    # Monitor application config changes
    - filter: /app/config/
      recursive: true
      behavior: monitor change
    # Block shell access to web root
    - filter: /var/www/html/
      recursive: true
      behavior: block access
      applications:
        - sh
        - bash
        - python3
```

```bash
kubectl apply -f file-access-policy.yaml
```

## Step 5: Protect Sensitive Directories

Configure strict access controls for common sensitive paths:

```bash
# Deny access to credential files
curl -sk -X POST \
  "https://neuvector-manager:8443/v1/file/profile/group/nv.myapp.production" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "process_profile": {
      "process_list": [
        {
          "filter": "/etc/ssl/private/",
          "recursive": true,
          "behavior": "block access"
        },
        {
          "filter": "/.ssh/",
          "recursive": false,
          "behavior": "block access"
        },
        {
          "filter": "/proc/",
          "recursive": true,
          "behavior": "monitor change"
        },
        {
          "filter": "/sys/",
          "recursive": true,
          "behavior": "monitor change"
        }
      ]
    }
  }'
```

## Step 6: Monitor File Integrity

File access rules double as a file integrity monitoring (FIM) system:

```bash
# View file access violation events
curl -sk \
  "https://neuvector-manager:8443/v1/event?type=file&start=0&limit=50" \
  -H "X-Auth-Token: ${TOKEN}" | jq '.events[] | {
    container: .workload_name,
    file: .file_name,
    process: .proc_name,
    action: .action,
    timestamp: .at
  }'
```

In the UI:
1. Go to **Notifications** > **Security Events**
2. Filter by type: **File Access**

## Step 7: Configure Rules for Multi-Stage Application Startup

Applications that have initialization phases may need temporary write access:

```yaml
# Allow write access during startup only for specific processes
apiVersion: neuvector.com/v1
kind: NvSecurityRule
metadata:
  name: app-startup-file-rules
  namespace: default
spec:
  target:
    policymode: Monitor  # Use Monitor during initial deployment
    selector:
      matchLabels:
        app: myapp
  file:
    - filter: /app/data/
      recursive: true
      behavior: monitor change
      applications:
        - myapp
    - filter: /tmp/
      recursive: true
      behavior: monitor change
```

## Step 8: Audit File Access Patterns

Export file access data for compliance and auditing:

```bash
# Export file access events for the last 24 hours
curl -sk \
  "https://neuvector-manager:8443/v1/event?type=file&start=0&limit=1000" \
  -H "X-Auth-Token: ${TOKEN}" | \
  jq '[.events[] | {
    timestamp: .at,
    namespace: .namespace,
    container: .workload_name,
    file: .file_name,
    process: .proc_name,
    action: .action
  }]' > file-access-audit.json
```

## Conclusion

NeuVector file access rules provide both preventive control and detective capability for container filesystem activity. By monitoring changes to critical files and blocking unauthorized access to sensitive directories, you can enforce file integrity and prevent common attack patterns like web shell installation and credential theft. Combine file access rules with process profiles and network rules for a comprehensive zero-trust runtime security posture.
