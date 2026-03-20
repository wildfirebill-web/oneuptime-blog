# How to Set Up Multi-Tenant SaaS Platform on Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: rancher, multi-tenant, saas, kubernetes, isolation, namespace

Description: A guide to building a multi-tenant SaaS platform on Rancher, covering tenant isolation, resource quotas, network policies, and self-service provisioning.

## Overview

Building a multi-tenant SaaS platform on Kubernetes requires robust isolation between tenants, self-service provisioning, resource governance, and scalability. Rancher's Projects, namespace-based isolation, RBAC, and resource quotas provide the foundation for multi-tenancy. This guide covers designing and implementing a production-grade multi-tenant SaaS platform on Rancher.

## Multi-Tenancy Architecture

```
Rancher Multi-Tenant SaaS Platform
├── Shared Infrastructure Cluster
│   ├── Ingress Controller (nginx)
│   ├── Cert Manager
│   └── Monitoring Stack

├── Per-Tier Application Clusters
│   ├── Starter Tier Cluster
│   ├── Professional Tier Cluster
│   └── Enterprise Tier Cluster

└── Per-Tenant Isolation
    ├── Tenant A: Namespace isolation
    ├── Tenant B: Namespace isolation
    └── Tenant C: Dedicated cluster (enterprise tier)
```

## Tenant Isolation Models

### Model 1: Namespace Isolation (Starter/Pro)

```yaml
# Create tenant namespace with labels
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-acme-corp
  labels:
    tenant: acme-corp
    tier: professional
    billing-plan: pro-2026
---
# Resource quota per tenant
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-quota
  namespace: tenant-acme-corp
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    persistentvolumeclaims: "10"
    services.loadbalancers: "1"
    pods: "50"
---
# LimitRange for default limits
apiVersion: v1
kind: LimitRange
metadata:
  name: tenant-limits
  namespace: tenant-acme-corp
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "4"
        memory: "8Gi"
```

### Network Isolation Between Tenants

```yaml
# Default deny all - tenants cannot communicate with each other
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: tenant-isolation
  namespace: tenant-acme-corp
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Only accept traffic from the ingress controller
    - from:
        - namespaceSelector:
            matchLabels:
              app.kubernetes.io/name: ingress-nginx
    # Internal communication within namespace
    - from:
        - podSelector: {}
  egress:
    # Allow DNS
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - port: 53
          protocol: UDP
    # Allow access to shared services (monitoring, logging)
    - to:
        - namespaceSelector:
            matchLabels:
              zone: shared-services
    # Allow internal namespace communication
    - to:
        - podSelector: {}
```

## Self-Service Tenant Provisioning API

```python
#!/usr/bin/env python3
# tenant-provisioner.py - Called by your SaaS onboarding flow

import subprocess
import json
import yaml
from typing import Dict

class TenantProvisioner:
    def __init__(self, rancher_client, cluster_id: str):
        self.rancher = rancher_client
        self.cluster_id = cluster_id

    def provision_tenant(self, tenant_config: Dict) -> Dict:
        """Provision a new SaaS tenant"""
        tenant_id = tenant_config['id']
        plan = tenant_config['plan']   # starter, professional, enterprise
        namespace = f"tenant-{tenant_id}"

        print(f"Provisioning tenant: {tenant_id} on plan: {plan}")

        # Create namespace
        self._create_namespace(namespace, tenant_config)

        # Apply resource quota based on plan
        self._apply_quota(namespace, plan)

        # Apply network policies
        self._apply_network_policies(namespace)

        # Create tenant service account
        sa_token = self._create_service_account(namespace, tenant_id)

        # Create ingress for tenant
        self._create_ingress(namespace, tenant_config)

        return {
            'tenant_id': tenant_id,
            'namespace': namespace,
            'kubeconfig': self._generate_kubeconfig(sa_token, tenant_id)
        }

    def _create_namespace(self, namespace: str, tenant_config: Dict):
        manifest = {
            'apiVersion': 'v1',
            'kind': 'Namespace',
            'metadata': {
                'name': namespace,
                'labels': {
                    'tenant': tenant_config['id'],
                    'tier': tenant_config['plan'],
                    'billing-account': tenant_config.get('billing_id', '')
                }
            }
        }
        subprocess.run(
            ['kubectl', 'apply', '-f', '-'],
            input=yaml.dump(manifest), text=True, check=True
        )

    def _apply_quota(self, namespace: str, plan: str):
        quotas = {
            'starter': {'cpu': '1', 'memory': '2Gi', 'pods': '10'},
            'professional': {'cpu': '4', 'memory': '8Gi', 'pods': '50'},
            'enterprise': {'cpu': '16', 'memory': '32Gi', 'pods': '200'}
        }
        quota = quotas.get(plan, quotas['starter'])

        manifest = {
            'apiVersion': 'v1',
            'kind': 'ResourceQuota',
            'metadata': {'name': 'tenant-quota', 'namespace': namespace},
            'spec': {
                'hard': {
                    'requests.cpu': quota['cpu'],
                    'requests.memory': quota['memory'],
                    'pods': quota['pods']
                }
            }
        }
        subprocess.run(
            ['kubectl', 'apply', '-f', '-'],
            input=yaml.dump(manifest), text=True, check=True
        )

    def _create_ingress(self, namespace: str, tenant_config: Dict):
        """Create tenant subdomain ingress"""
        tenant_id = tenant_config['id']
        manifest = {
            'apiVersion': 'networking.k8s.io/v1',
            'kind': 'Ingress',
            'metadata': {
                'name': 'tenant-ingress',
                'namespace': namespace,
                'annotations': {
                    'nginx.ingress.kubernetes.io/proxy-body-size': '50m',
                    'cert-manager.io/cluster-issuer': 'letsencrypt-prod'
                }
            },
            'spec': {
                'tls': [{
                    'hosts': [f"{tenant_id}.app.saas.example.com"],
                    'secretName': f"{namespace}-tls"
                }],
                'rules': [{
                    'host': f"{tenant_id}.app.saas.example.com",
                    'http': {
                        'paths': [{
                            'path': '/',
                            'pathType': 'Prefix',
                            'backend': {
                                'service': {
                                    'name': 'webapp',
                                    'port': {'number': 8080}
                                }
                            }
                        }]
                    }
                }]
            }
        }
        subprocess.run(
            ['kubectl', 'apply', '-f', '-'],
            input=yaml.dump(manifest), text=True, check=True
        )
```

## Rancher Projects for Tenant Grouping

```bash
# Create Rancher Project for tenant tier grouping
curl -s -k \
  -X POST \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  -H "Content-Type: application/json" \
  "${RANCHER_URL}/v3/projects" \
  -d "{
    \"clusterId\": \"${CLUSTER_ID}\",
    \"name\": \"Professional Tenants\",
    \"resourceQuota\": {
      \"limit\": {
        \"limitsCpu\": \"100\",
        \"limitsMemory\": \"200Gi\"
      }
    },
    \"containerDefaultResourceLimit\": {
      \"limitsCpu\": \"500m\",
      \"limitsMemory\": \"512Mi\"
    }
  }"
```

## Billing Integration

```yaml
# Label namespaces with billing metadata for cost tracking
# These labels are read by your cost management tool (Kubecost/OpenCost)
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-acme-corp
  labels:
    billing-tenant-id: "t-12345"
    billing-plan: "professional"
    billing-cycle: "monthly"
    cost-center: "saas-platform"
```

## Conclusion

Building a multi-tenant SaaS platform on Rancher requires careful design of namespace isolation, network policies, resource quotas, and self-service provisioning workflows. Rancher's Projects provide organizational grouping, namespace-level ResourceQuotas enforce tenant limits, and NetworkPolicies prevent cross-tenant traffic. Automating tenant provisioning via API ensures consistent configuration and eliminates manual errors. As your SaaS platform grows, consider moving enterprise tier tenants to dedicated clusters for stronger isolation and higher resource guarantees.
