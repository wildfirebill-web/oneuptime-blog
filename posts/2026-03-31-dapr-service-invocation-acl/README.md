# How to Use Dapr Service Invocation with Access Control Lists

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Access Control, Security, Authorization, Service Invocation

Description: Learn how to configure Dapr access control lists to restrict which services can invoke which methods, enforcing least-privilege communication between microservices.

---

## What Are Dapr Access Control Lists?

Dapr Access Control Lists (ACLs) let you define which source applications are allowed to call which methods on a target application. Rules are enforced at the sidecar level using mutual TLS identity, so they cannot be bypassed by manipulating headers.

## Enabling mTLS (Required for ACLs)

ACLs require mTLS to identify callers. mTLS is enabled by default in Kubernetes. Verify:

```bash
kubectl get configuration daprsystem -n dapr-system -o jsonpath='{.spec.mtls.enabled}'
# Should output: true
```

## Defining an Access Control Policy

Create a Dapr Configuration for the `order-service` that restricts who can call it:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: order-service-config
  namespace: default
spec:
  accessControl:
    defaultAction: deny
    trustDomain: "cluster.local"
    policies:
      - appId: payment-service
        defaultAction: allow
        trustDomain: "cluster.local"
        namespace: "default"
        operations:
          - name: /v1/charge
            httpVerb: ["POST"]
            action: allow
          - name: /v1/refund
            httpVerb: ["POST"]
            action: allow
      - appId: admin-service
        defaultAction: allow
        trustDomain: "cluster.local"
        namespace: "default"
        operations:
          - name: "*"
            action: allow
```

This configuration denies all traffic by default, allows `payment-service` to call only `/v1/charge` and `/v1/refund` via POST, and allows `admin-service` to call any method.

## Applying the Configuration to the Target App

Reference the configuration in the deployment annotation:

```yaml
annotations:
  dapr.io/enabled: "true"
  dapr.io/app-id: "order-service"
  dapr.io/config: "order-service-config"
```

## Testing the ACL

```bash
# From payment-service sidecar - this should succeed
curl http://localhost:3500/v1.0/invoke/order-service/method/v1/charge \
  -X POST -d '{"amount": 100}'

# From inventory-service sidecar - this should return 403
curl http://localhost:3500/v1.0/invoke/order-service/method/v1/charge \
  -X POST -d '{"amount": 100}'
```

## Wildcard Patterns in ACLs

```yaml
operations:
  - name: /v1/*
    httpVerb: ["GET"]
    action: allow
  - name: /admin/*
    action: deny
```

## Summary

Dapr ACLs enforce least-privilege communication between microservices by specifying which app IDs can call which methods. Set `defaultAction: deny` in a Configuration resource, then whitelist allowed callers and operations. ACLs rely on mTLS identity and cannot be spoofed with header manipulation.
