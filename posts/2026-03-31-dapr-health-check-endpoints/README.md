# How to Set Up Dapr Health Check Endpoints

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Health, Monitoring, Kubernetes, Sidecar

Description: Set up and use Dapr health check endpoints to verify sidecar readiness, check outbound component availability, and implement startup sequencing for Dapr-enabled services.

---

## Dapr Health Check Endpoints

The Dapr sidecar exposes several health check endpoints that you can use for Kubernetes probes, startup sequencing, and operational monitoring:

| Endpoint | Purpose |
|----------|---------|
| `/v1.0/healthz` | Sidecar is running and healthy |
| `/v1.0/healthz/outbound` | Sidecar can reach all components |
| `/v1.0/metadata` | Full sidecar metadata including component status |

## Configuring Kubernetes Liveness Probe

Add a liveness probe that checks the Dapr sidecar:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      containers:
        - name: app
          livenessProbe:
            httpGet:
              path: /v1.0/healthz
              port: 3500
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3
```

Note: Port 3500 is the default Dapr HTTP port on the sidecar.

## Configuring Readiness Probe with Outbound Check

Use the outbound health endpoint for readiness - it verifies that all Dapr components are reachable:

```yaml
readinessProbe:
  httpGet:
    path: /v1.0/healthz/outbound
    port: 3500
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
```

The outbound endpoint returns HTTP 200 only when all configured components (state stores, pub/sub, etc.) are accessible.

## Waiting for Dapr to Be Ready at App Startup

Implement startup sequencing in your application to wait for the Dapr sidecar:

```python
import time
import requests
import logging

logger = logging.getLogger(__name__)

def wait_for_dapr(max_retries: int = 30, delay: float = 1.0):
    """Wait for Dapr sidecar to be ready before starting the app."""
    for attempt in range(max_retries):
        try:
            response = requests.get(
                "http://localhost:3500/v1.0/healthz/outbound",
                timeout=2
            )
            if response.status_code == 204:
                logger.info("Dapr sidecar is ready")
                return True
        except requests.exceptions.RequestException:
            pass

        logger.info(f"Waiting for Dapr... attempt {attempt + 1}/{max_retries}")
        time.sleep(delay)

    raise RuntimeError("Dapr sidecar did not become ready in time")

# Call at application startup
wait_for_dapr()
# Now safe to use Dapr APIs
```

## Checking Component Health via Metadata API

Get detailed component health status:

```bash
curl http://localhost:3500/v1.0/metadata | python3 -m json.tool | grep -A 5 "components"
```

Response includes each component's status:

```json
{
  "components": [
    {
      "name": "statestore",
      "type": "state.redis",
      "version": "v1",
      "capabilities": ["ETAG", "TRANSACTIONAL"],
      "status": "OK"
    }
  ]
}
```

## Implementing Health Check in Your App

Expose your own health endpoint that checks Dapr component availability:

```javascript
app.get('/health', async (req, res) => {
  try {
    const daprHealth = await fetch('http://localhost:3500/v1.0/healthz/outbound');
    if (daprHealth.status !== 204) {
      return res.status(503).json({ status: 'degraded', reason: 'dapr-not-ready' });
    }
    res.status(200).json({ status: 'healthy' });
  } catch (err) {
    res.status(503).json({ status: 'unhealthy', reason: err.message });
  }
});
```

## Summary

Dapr provides `/v1.0/healthz` for basic sidecar health and `/v1.0/healthz/outbound` for component connectivity checks. Use the outbound endpoint for Kubernetes readiness probes to prevent traffic from reaching pods before Dapr components are reachable. Implement startup sequencing in your application to wait for the sidecar before making Dapr API calls, preventing race conditions during pod initialization.
