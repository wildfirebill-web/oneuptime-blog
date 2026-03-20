# How to Set Up NeuVector Zero Trust Mode

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: NeuVector, Zero Trust, Container Security, Kubernetes, Network Security

Description: Implement a zero-trust security model with NeuVector by transitioning all container workloads to Protect mode with explicit deny-all defaults and whitelisted exceptions.

## Introduction

Zero trust security operates on the principle that no container, process, or network connection should be trusted by default. Every action must be explicitly authorized. NeuVector is purpose-built for zero-trust container security, providing the tools to implement "never trust, always verify" at the container level. This guide explains how to build a full zero-trust security posture.

## Zero Trust Principles in NeuVector

| Principle | NeuVector Implementation |
|---|---|
| Verify explicitly | Process profiles verify every process |
| Use least privilege | Network rules with default deny |
| Assume breach | Runtime monitoring with auto-quarantine |
| Micro-segmentation | Per-container group policies |
| Continuous validation | Ongoing behavioral monitoring |

## Prerequisites

- NeuVector installed with all components running
- All workloads have been in Discover mode for at least 48 hours
- Security policies have been reviewed and refined
- Change management approval for production workloads

## Step 1: Establish the Baseline

Before enforcing zero trust, ensure the baseline is accurate:

```bash
# Review discovered process profiles
curl -sk \
  "https://neuvector-manager:8443/v1/group?start=0&limit=100" \
  -H "X-Auth-Token: ${TOKEN}" | \
  jq '.groups[] | select(.policy_mode == "Discover") | .name'

# Review network rules for each group
curl -sk \
  "https://neuvector-manager:8443/v1/policy/rule?start=0&limit=500" \
  -H "X-Auth-Token: ${TOKEN}" | \
  jq '[.rules[] | select(.cfg_type == "learned")] | length'
```

## Step 2: Promote Learned Rules to User-Defined Rules

Convert auto-discovered rules into explicit policy:

```bash
# In the NeuVector UI:
# 1. Policy > Network Rules
# 2. Filter by Type: Learned
# 3. Select all relevant rules
# 4. Click "Promote" to convert to user-defined rules

# Via API - get learned rules and recreate as user rules
LEARNED_RULES=$(curl -sk \
  "https://neuvector-manager:8443/v1/policy/rule?start=0&limit=500" \
  -H "X-Auth-Token: ${TOKEN}" | \
  jq '[.rules[] | select(.cfg_type == "learned")]')

echo "Found $(echo ${LEARNED_RULES} | jq length) learned rules to review"
```

## Step 3: Add Default Deny Rules

Implement explicit default-deny at the network level:

```bash
# Add a deny-all rule at the lowest priority
curl -sk -X POST \
  "https://neuvector-manager:8443/v1/policy/rule" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "insert": {
      "after": 9999,
      "rules": [
        {
          "comment": "Zero Trust: Deny all unmatched traffic",
          "from": "any",
          "to": "any",
          "ports": "any",
          "action": "deny",
          "cfg_type": "user"
        }
      ]
    }
  }'
```

## Step 4: Configure Zero-Trust Process Profiles

Remove permissive process rules and define explicit allowlists:

```yaml
# zero-trust-process-policy.yaml
apiVersion: neuvector.com/v1
kind: NvSecurityRule
metadata:
  name: zero-trust-webapp
  namespace: production
spec:
  target:
    policymode: Protect
    selector:
      matchLabels:
        app: webapp
  process:
    # Explicitly allow only required processes
    - name: node
      path: /usr/local/bin/node
      action: allow
    - name: npm
      path: /usr/local/bin/npm
      action: allow
    # Explicitly deny attack vectors
    - name: sh
      action: deny
    - name: bash
      action: deny
    - name: sh
      path: /bin/sh
      action: deny
    - name: bash
      path: /bin/bash
      action: deny
    - name: curl
      action: deny
    - name: wget
      action: deny
    - name: nc
      action: deny
    - name: nmap
      action: deny
    - name: python3
      action: deny
    - name: python
      action: deny
    - name: perl
      action: deny
    - name: ruby
      action: deny
```

## Step 5: Implement Micro-Segmentation

Create explicit network rules for every allowed communication:

```yaml
# micro-segmentation-policy.yaml
apiVersion: neuvector.com/v1
kind: NvClusterSecurityRule
metadata:
  name: production-zero-trust
  namespace: neuvector
spec:
  target:
    policymode: Protect
    selector:
      matchLabels:
        env: production
  ingress:
    # Only allow ingress from the load balancer
    - action: allow
      name: allow-lb-to-web
      selector:
        matchLabels:
          app: ingress-nginx
      ports:
        - protocol: TCP
          port: 8080
  egress:
    # Allow web to API only
    - action: allow
      name: allow-web-to-api
      selector:
        matchLabels:
          tier: api
      ports:
        - protocol: TCP
          port: 3000
    # Allow API to database only
    - action: allow
      name: allow-api-to-db
      selector:
        matchLabels:
          tier: database
      ports:
        - protocol: TCP
          port: 5432
    # Allow DNS resolution
    - action: allow
      name: allow-dns
      selector:
        matchLabels:
          k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
    # Block all other egress (including internet)
    - action: deny
      name: deny-all-other-egress
      selector: {}
```

## Step 6: Enable Auto-Quarantine for Breaches

Configure automatic quarantine for containers that violate zero-trust policies:

```bash
# Create response rule for auto-quarantine
curl -sk -X POST \
  "https://neuvector-manager:8443/v1/response/rule" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "event": "security-event",
      "comment": "Zero Trust: Auto-quarantine on critical violations",
      "conditions": [
        {"type": "level", "value": "critical"},
        {"type": "name", "value": "process-violation"}
      ],
      "actions": ["quarantine", "webhook"],
      "webhooks": ["security-oncall"],
      "disable": false
    }
  }'
```

## Step 7: Transition to Protect Mode Gradually

Move namespaces to Protect mode incrementally:

```bash
#!/bin/bash
# gradual-protect-transition.sh

NAMESPACE="production"

# Get all groups in namespace
GROUPS=$(curl -sk \
  "https://neuvector-manager:8443/v1/group?start=0&limit=100" \
  -H "X-Auth-Token: ${TOKEN}" | \
  jq -r --arg ns "$NAMESPACE" \
  '.groups[] | select(.name | contains("." + $ns)) | .name')

# Move each group to Monitor first
for GROUP in $GROUPS; do
  echo "Setting ${GROUP} to Monitor mode..."
  curl -sk -X PATCH \
    "https://neuvector-manager:8443/v1/group/${GROUP}" \
    -H "Content-Type: application/json" \
    -H "X-Auth-Token: ${TOKEN}" \
    -d '{"config": {"mode": "Monitor"}}'
done

echo "All groups in ${NAMESPACE} now in Monitor mode. Review events before switching to Protect."
```

## Step 8: Monitor Zero-Trust Effectiveness

Track violations to measure the effectiveness of zero-trust:

```bash
# Daily violation summary
curl -sk \
  "https://neuvector-manager:8443/v1/event?type=security&start=0&limit=1000" \
  -H "X-Auth-Token: ${TOKEN}" | \
  jq '[.events[] | {
    type: .type,
    action: .action
  }] | group_by(.type) | map({
    type: .[0].type,
    count: length
  })'
```

## Conclusion

Implementing zero trust with NeuVector requires a systematic approach: start in Discover mode, review and promote learned rules, add default-deny policies, harden process profiles, and gradually transition to Protect mode. The result is a security posture where every container action is explicitly authorized — making breaches detectable and limiting their blast radius. Zero trust is a journey, not a destination; continuously refine your policies as your applications evolve.
