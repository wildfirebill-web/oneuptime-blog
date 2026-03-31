# How to Troubleshoot Dapr Certificate Issuance Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Sentry, Certificate, Troubleshooting, mTLS

Description: Troubleshoot Dapr certificate issuance failures including CA initialization errors, sidecar-to-Sentry connectivity issues, and clock skew problems affecting mTLS.

---

## Symptoms of Certificate Issuance Failures

Certificate issuance issues manifest as:

- Sidecar fails to start with `failed to get workload cert` error
- Service-to-service calls fail with TLS handshake errors
- `certificate signed by unknown authority` errors in logs
- Services work initially but fail after 24 hours (certificate expiry)

## Checking Sidecar Logs for Cert Errors

```bash
kubectl logs <pod-name> -c daprd | grep -iE "cert|sentry|tls|x509" | tail -50
```

Common error messages and their causes:

```bash
# Error: failed to request workload cert
# Cause: Cannot reach Sentry service

# Error: certificate signed by unknown authority
# Cause: Trust bundle mismatch or CA rotation

# Error: certificate has expired or is not yet valid
# Cause: Clock skew between nodes
```

## Verifying Sentry Is Reachable

The sidecar connects to Sentry on port 50001:

```bash
# Test from within an app pod
kubectl exec -it <pod-name> -c daprd -- \
  sh -c "nc -zv dapr-sentry.dapr-system.svc.cluster.local 50001 && echo CONNECTED"
```

Check network policies that may block this connection:

```bash
kubectl describe networkpolicy -n dapr-system
kubectl describe networkpolicy -n <app-namespace>
```

## Diagnosing Trust Bundle Issues

If certificates are issued but not trusted, there may be a stale trust bundle:

```bash
# Check the current trust bundle
kubectl get configmap dapr-trust-bundle -n dapr-system -o yaml

# Check when Sentry last updated it
kubectl describe configmap dapr-trust-bundle -n dapr-system | grep "Last Applied"
```

Force a trust bundle refresh by restarting Sentry:

```bash
kubectl rollout restart deployment/dapr-sentry -n dapr-system
```

## Checking Clock Skew

Certificate validation fails if clocks differ by more than `allowedClockSkew`:

```bash
# Compare timestamps across nodes
kubectl get nodes -o custom-columns='NAME:.metadata.name,TIME:.status.conditions[-1].lastHeartbeatTime'

# Check time in sentry pod
kubectl exec -n dapr-system -l app=dapr-sentry -- date -u

# Check time in app pod
kubectl exec -n default <app-pod> -- date -u
```

If skew exceeds 15 minutes, increase the allowed clock skew or fix NTP on affected nodes.

## Recovering from Expired CA

If the Sentry CA certificate has expired, you must rotate it:

```bash
# Generate new CA
dapr mtls export -o ./certs

# Verify expiry
openssl x509 -in ./certs/ca.crt -enddate -noout

# Reissue if expired
openssl genrsa -out ca.key 4096
openssl req -new -x509 -days 3650 -key ca.key -out ca.crt \
  -subj "/CN=Dapr Root CA"

# Update the secret
kubectl create secret generic dapr-trust-bundle \
  --from-file=ca.crt=./ca.crt \
  -n dapr-system --dry-run=client -o yaml | kubectl apply -f -

kubectl rollout restart deployment/dapr-sentry -n dapr-system
```

## Summary

Troubleshoot Dapr certificate issuance by checking sidecar logs for specific error messages, verifying Sentry connectivity on port 50001, inspecting the trust bundle for staleness, and checking for clock skew between nodes. Most issuance failures are caused by network policies blocking port 50001 or expired CA certificates that need rotation.
