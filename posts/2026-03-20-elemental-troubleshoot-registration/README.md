# How to Troubleshoot Elemental Registration Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Elemental, Troubleshooting, Kubernetes, Edge, Debugging

Description: Debug and resolve common Elemental machine registration failures including connectivity, certificate, and configuration issues.

## Introduction

Machine registration failures are one of the most common issues when deploying Elemental. Machines may fail to contact the registration endpoint, fail TLS validation, or encounter configuration errors during installation. This guide covers systematic troubleshooting approaches for the most common registration problems.

## Common Registration Failure Symptoms

- Machine boots but does not appear in MachineInventory
- Machine appears in inventory with error conditions
- OS installation fails or machine reboots repeatedly
- Certificate validation errors in machine logs

## Step 1: Check Operator Logs

```bash
# View recent operator logs

kubectl logs -n elemental-system \
  -l app=elemental-operator \
  --since=30m

# Stream logs while machine is registering
kubectl logs -n elemental-system \
  -l app=elemental-operator \
  --follow

# Look for specific error patterns
kubectl logs -n elemental-system \
  -l app=elemental-operator \
  --since=1h | grep -i "error\|fail\|warn"
```

## Step 2: Verify MachineRegistration Status

```bash
# Check registration conditions
kubectl describe machineregistration -n fleet-default my-nodes

# Get status as JSON for detailed inspection
kubectl get machineregistration -n fleet-default my-nodes \
  -o jsonpath='{.status}' | jq .

# Verify registration URL is populated
kubectl get machineregistration -n fleet-default my-nodes \
  -o jsonpath='{.status.registrationURL}'
```

## Step 3: Test Registration Endpoint Connectivity

```bash
# From the machine (via serial console or SSH during live boot):

# Test network connectivity
ping -c 3 rancher.example.com

# Test HTTPS connectivity
curl -v https://rancher.example.com

# Test with CA certificate
curl -v --cacert /etc/elemental/registration/ca.crt \
  https://rancher.example.com/v1/elemental/registration/TOKEN

# Test DNS resolution
nslookup rancher.example.com
dig rancher.example.com
```

## Step 4: Verify Certificate Configuration

```bash
# On the management cluster - check CA cert
kubectl get secret tls-rancher-internal-ca \
  -n cattle-system \
  -o jsonpath='{.data.cacerts\.pem}' | base64 -d | \
  openssl x509 -text -noout | grep -A 2 "Subject:\|Issuer:\|Not After"

# Verify the cert in your registration config matches
openssl x509 -in /tmp/registration-ca.pem -text -noout | \
  grep -A 2 "Subject:"

# Test TLS handshake
openssl s_client -connect rancher.example.com:443 \
  -CAfile /tmp/registration-ca.pem \
  -servername rancher.example.com
```

## Step 5: Check Machine Boot Logs

```bash
# Access machine console during boot (if accessible)
# Look for elemental registration logs

# Or check after failed boot using rescue mode
journalctl -u elemental-register --no-pager
journalctl -u elemental-install --no-pager

# Check system logs
journalctl -b -p err --no-pager
```

## Step 6: Debug Cloud-Config Issues

```bash
# On the machine, check if cloud-config was applied
cat /run/elemental/cloud-init.log

# Check cloud-init status
cloud-init status --long

# Validate cloud-config syntax
cloud-init schema --config-file /oem/registration/config.yaml
```

## Common Issues and Solutions

### Issue: Certificate Authority Not Trusted

```bash
# Solution: Regenerate seed image with correct CA cert
kubectl get secret tls-rancher-internal-ca \
  -n cattle-system \
  -o jsonpath='{.data.cacerts\.pem}' | base64 -d > correct-ca.pem

# Verify the cert is valid
openssl x509 -in correct-ca.pem -text -noout
```

### Issue: Registration URL Unreachable

```bash
# Check Rancher ingress
kubectl get ingress -n cattle-system

# Verify LoadBalancer IP
kubectl get svc -n cattle-system rancher \
  -o jsonpath='{.status.loadBalancer.ingress}'

# Test from within cluster
kubectl run test-curl --rm -it --image=curlimages/curl -- \
  curl -k https://rancher.example.com/v1/elemental
```

### Issue: Machine Appears with Error

```bash
# Check MachineInventory conditions
kubectl get machineinventory -n fleet-default \
  -o json | jq '.items[] | {name: .metadata.name, conditions: .status.conditions}'
```

## Conclusion

Systematic troubleshooting of Elemental registration issues starts with operator logs and works outward to network connectivity and certificate validation. Most registration failures fall into one of a few categories: incorrect CA certificates, network unreachability, or misconfigured registration URLs. By checking each layer methodically, you can quickly identify and resolve the root cause.
