# How to Troubleshoot CrashLoopBackOff Errors in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, CrashLoopBackOff, Troubleshooting, Operation

Description: Diagnose and resolve Kubernetes CrashLoopBackOff errors for containers managed through Portainer's web interface.

## Introduction

CrashLoopBackOff means a container is crashing repeatedly. Kubernetes backs off exponentially between restarts. Portainer surfaces these failures in its Applications view, and this guide provides a systematic approach to diagnosing the root cause.

## Identifying CrashLoopBackOff in Portainer

1. **Kubernetes > Applications**: Pod status shows "CrashLoopBackOff"
2. **Pod > Events**: Shows restart count and container exit codes
3. **Pod > Logs**: Shows why the container crashed (previous instance)

## Step 1: Get the Exit Code

```bash
# Exit code tells you WHY the container crashed

kubectl get pod crashing-pod -n production -o json \
  | python3 -c "
import sys, json
pod = json.load(sys.stdin)
for cs in pod['status'].get('containerStatuses', []):
    name = cs['name']
    restart_count = cs.get('restartCount', 0)
    last_state = cs.get('lastState', {}).get('terminated', {})
    exit_code = last_state.get('exitCode', 'N/A')
    reason = last_state.get('reason', 'N/A')
    print(f'{name}: restarts={restart_count}, exitCode={exit_code}, reason={reason}')
"
```

## Common Exit Codes

| Exit Code | Meaning | Common Cause |
|-----------|---------|--------------|
| 0 | Success | Wrong init command (loops) |
| 1 | General error | Application error |
| 2 | Misuse of command | Missing argument |
| 137 | SIGKILL (OOMKilled) | Out of memory |
| 139 | Segfault | Application bug |
| 143 | SIGTERM | Graceful shutdown (unexpected) |

## Step 2: Check Previous Container Logs

```bash
# Get logs from the PREVIOUS (crashed) container instance
kubectl logs crashing-pod -n production --previous

# In Portainer: Pod > Logs > Toggle "Previous"
```

## Diagnosing Specific Cases

### Case 1: Application Startup Error

```bash
# View startup logs
kubectl logs crashing-pod -n production --previous | head -50

# Common startup errors:
# - Missing environment variables
# - Database connection refused
# - Configuration file not found
# - Port already in use
```

### Case 2: OOMKilled (Exit Code 137)

```bash
# Check if OOMKilled
kubectl describe pod oom-pod -n production | grep -i "OOMKilled"

# Fix: Increase memory limit
kubectl patch deployment myapp -n production -p '{
  "spec": {"template": {"spec": {"containers": [
    {"name": "app", "resources": {"limits": {"memory": "512Mi"}}}
  ]}}}
}'
```

### Case 3: Missing Configuration

```yaml
# Check if ConfigMap/Secret exists
kubectl get configmap app-config -n production
kubectl get secret app-secrets -n production

# Fix: Create missing resources
kubectl create configmap app-config \
  --from-literal=key=value \
  -n production
```

### Case 4: Container Command Error

```yaml
# Wrong command in deployment
containers:
- name: app
  image: myapp:latest
  command: ["python"]           # Missing args!
  # Should be:
  command: ["python", "app.py"]
```

## Debugging with an Overridden Command

```yaml
# Temporarily override command to keep container running for debugging
# Deploy via Portainer YAML editor
containers:
- name: app
  image: myapp:latest
  command: ["sleep", "infinity"]  # Override crashing command temporarily
  # This keeps the container running so you can exec into it
```

```bash
# Then exec into it for investigation
kubectl exec -it crashing-pod -n production -- sh

# Check if dependencies are available
which python3
ls /app
cat /app/config.yaml
env | grep DATABASE
```

## Automatic Fix Script

```python
#!/usr/bin/env python3
# analyze_crashloop.py

import requests
import json

PORTAINER_URL = "https://portainer.example.com"
API_KEY = "your-api-key"
ENDPOINT_ID = 1

def find_crashlooping_pods(namespace="default"):
    resp = requests.get(
        f"{PORTAINER_URL}/api/endpoints/{ENDPOINT_ID}/kubernetes/api/v1/namespaces/{namespace}/pods",
        headers={"X-API-Key": API_KEY}
    )
    pods = resp.json().get('items', [])
    
    crashlooping = []
    for pod in pods:
        for cs in pod['status'].get('containerStatuses', []):
            state = cs.get('state', {})
            waiting = state.get('waiting', {})
            if waiting.get('reason') == 'CrashLoopBackOff':
                crashlooping.append({
                    'pod': pod['metadata']['name'],
                    'container': cs['name'],
                    'restarts': cs.get('restartCount', 0),
                    'exit_code': cs.get('lastState', {}).get('terminated', {}).get('exitCode')
                })
    
    return crashlooping

pods = find_crashlooping_pods("production")
for p in pods:
    print(f"CrashLoop: {p['pod']}/{p['container']} "
          f"(restarts: {p['restarts']}, last exit: {p['exit_code']})")
```

## Conclusion

CrashLoopBackOff diagnosis follows a systematic pattern: get the exit code, check previous container logs, identify the root cause (OOM, app error, config missing), and apply the fix. Portainer's log viewer and previous-instance log access are your primary tools. For complex issues, temporarily overriding the container command to keep it running enables interactive debugging.
