# How to Configure AWS CloudMap Name Resolution in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, AWS, CloudMap, Name Resolution, Service Discovery

Description: Learn how to configure Dapr to use AWS Cloud Map for service discovery and name resolution when running on AWS infrastructure.

---

## What Is AWS Cloud Map?

AWS Cloud Map is a managed service registry that allows you to register resources and query them by name. When running Dapr workloads on AWS (ECS, EKS, EC2), you can use Cloud Map as the name resolution backend so Dapr can discover and invoke services registered there.

The Dapr `nameresolution.aws.snssqs` does not handle name resolution directly, but Dapr supports AWS Cloud Map via the `nameresolution.consul` component configured to point to an AWS Cloud Map proxy, or via a custom name resolution component.

For AWS-native environments, the recommended approach uses AWS Cloud Map directly via the AWS SDK in a custom component, or uses the built-in approach with environment variable injection.

## Enabling AWS Cloud Map in Your Infrastructure

First, create a Cloud Map namespace and service:

```bash
aws servicediscovery create-private-dns-namespace \
  --name dapr.local \
  --vpc vpc-0123456789abcdef0 \
  --region us-east-1

aws servicediscovery create-service \
  --name order-service \
  --dns-config "NamespaceId=ns-xxxxxxxxxxxx,DnsRecords=[{Type=A,TTL=10}]" \
  --health-check-custom-config FailureThreshold=1
```

## Using Kubernetes DNS with Cloud Map on EKS

On EKS with AWS Cloud Map integration, the CoreDNS configuration can route queries to Cloud Map. Configure CoreDNS to forward `dapr.local` queries:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
    dapr.local:53 {
        errors
        forward . 169.254.20.10:53
    }
```

## Configuring Dapr for Cloud Map-Backed DNS

With Cloud Map integrated into your DNS, configure Dapr to use a `nameresolution.kubernetes`-style component pointing to your cluster domain:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: nameresolution
  namespace: default
spec:
  type: nameresolution.kubernetes
  version: v1
  metadata:
    - name: clusterDomain
      value: "dapr.local"
```

## Registering Instances with Cloud Map

When deploying a new service, register it with Cloud Map:

```bash
aws servicediscovery register-instance \
  --service-id srv-xxxxxxxxxxxx \
  --instance-id order-service-1 \
  --attributes AWS_INSTANCE_IPV4=10.0.1.42,AWS_INSTANCE_PORT=3500
```

For ECS tasks, use the `serviceRegistries` field in the task definition:

```json
{
  "serviceRegistries": [
    {
      "registryArn": "arn:aws:servicediscovery:us-east-1:123456789012:service/srv-xxxx",
      "port": 3500
    }
  ]
}
```

## IAM Permissions Required

Ensure the IAM role for your service has the following permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "servicediscovery:RegisterInstance",
        "servicediscovery:DeregisterInstance",
        "servicediscovery:DiscoverInstances"
      ],
      "Resource": "*"
    }
  ]
}
```

## Verifying Resolution

Test that Cloud Map is resolving your service correctly:

```bash
aws servicediscovery discover-instances \
  --namespace-name dapr.local \
  --service-name order-service \
  --region us-east-1
```

Then invoke via Dapr:

```bash
curl http://localhost:3500/v1.0/invoke/order-service/method/health
```

## Summary

AWS Cloud Map provides a managed service registry for Dapr workloads running on AWS. Integrate it with EKS via CoreDNS forwarding rules, or register instances directly for ECS workloads. Ensure proper IAM permissions for instance registration and discovery, and configure the Dapr name resolution component to match your cluster domain.
