# How to Troubleshoot Kubernetes DNS Issues from Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, DNS, CoreDNS, Troubleshooting

Description: Debug Kubernetes DNS resolution failures using Portainer's terminal access to run nslookup, dig, and CoreDNS log inspection.

---

Kubernetes DNS issues prevent pods from resolving service names, external hostnames, or cluster-internal names. CoreDNS handles all DNS in most clusters, and Portainer's terminal gives you access to run the diagnostic commands needed.

## Step 1: Verify CoreDNS is Running

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

If CoreDNS pods are not running, all DNS resolution in the cluster fails.

## Step 2: Test DNS from Inside a Pod

Use Portainer's terminal to exec into a running pod:

```bash
## Test service DNS resolution
nslookup kubernetes.default.svc.cluster.local
nslookup my-service.production.svc.cluster.local

## Test external DNS
nslookup google.com
```

Kubernetes service DNS format: \`<service>.<namespace>.svc.cluster.local\`

## Step 3: Check CoreDNS Logs

```bash
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50
```

Look for:
- \`NXDOMAIN\` - name does not exist (wrong service name)
- \`SERVFAIL\` - CoreDNS upstream failure
- \`i/o timeout\` - CoreDNS cannot reach upstream resolvers

## Step 4: Inspect CoreDNS ConfigMap

```bash
kubectl get configmap coredns -n kube-system -o yaml
```

The Corefile controls forwarding behavior. Check that the \`forward\` directive points to valid upstream resolvers.

## Step 5: Run a DNS Debug Pod

```bash
## Deploy a temporary debug pod with DNS tools
kubectl run dns-debug   --image=infoblox/dnstools:latest   --restart=Never --rm -it   -n production   -- /bin/bash

## Inside the pod
host my-service
dig my-service.production.svc.cluster.local
```

## Step 6: Check ndots and Search Domains

Short names like \`my-service\` go through the search domain list. Check the pod's \`/etc/resolv.conf\`:

```bash
cat /etc/resolv.conf
search production.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
## options ndots:5
```

With \`ndots:5\`, names with fewer than 5 dots are tried with search suffixes first.

## Summary

Kubernetes DNS troubleshooting starts with confirming CoreDNS is running, then testing resolution from inside a pod. NXDOMAIN errors mean the service name or namespace is wrong; timeout errors mean CoreDNS cannot reach upstream resolvers. Portainer's terminal provides the exec access needed for all these diagnostic commands.
