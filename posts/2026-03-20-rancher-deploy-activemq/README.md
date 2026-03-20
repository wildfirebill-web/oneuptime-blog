# How to Deploy ActiveMQ on Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, ActiveMQ, Message Queue, JMS

Description: Deploy Apache ActiveMQ Artemis on Rancher for JMS-compliant messaging with support for AMQP, STOMP, and MQTT protocols.

## Introduction

Apache ActiveMQ Artemis is the next-generation ActiveMQ broker that supports multiple messaging protocols including JMS, AMQP, STOMP, and MQTT. It's widely used in enterprise Java applications. This guide covers deploying ActiveMQ Artemis on Rancher using the ArtemisCloud operator for Kubernetes-native management.

## Prerequisites

- Rancher-managed Kubernetes cluster
- Helm 3.x installed
- kubectl access
- A StorageClass for persistent volumes

## Step 1: Install the ArtemisCloud Operator

```bash
# Install ArtemisCloud Operator
helm repo add artemiscloud https://artemiscloud.io/helm-charts
helm repo update

helm install activemq-operator artemiscloud/activemq-artemis-operator \
  --namespace activemq-operator \
  --create-namespace \
  --wait

# Verify operator
kubectl get pods -n activemq-operator
```

## Step 2: Deploy an ActiveMQ Artemis Broker

```yaml
# activemq-cluster.yaml - ActiveMQ Artemis cluster
apiVersion: broker.amq.io/v1beta1
kind: ActiveMQArtemis
metadata:
  name: activemq-prod
  namespace: messaging
spec:
  deploymentPlan:
    # Number of broker instances
    size: 2

    # Enable high availability
    requireLogin: true
    persistenceEnabled: true

    # Storage configuration
    storage:
      storageClassName: standard
      size: 20Gi

    # Resource limits
    resources:
      limits:
        cpu: "2"
        memory: 2Gi
      requests:
        cpu: 500m
        memory: 1Gi

  # Acceptors (protocols)
  acceptors:
    - name: amqp
      port: 5672
      protocols: amqp
      sslEnabled: false
    - name: core
      port: 61616
      protocols: core
    - name: stomp
      port: 61613
      protocols: stomp
    - name: mqtt
      port: 1883
      protocols: mqtt

  # Address settings for queues
  addressSettings:
    applyRule: merge_all
    addressSetting:
      - match: "#"
        # Dead letter queue
        deadLetterAddress: DLQ
        autoCreateDeadLetterResources: true
        # Expiry queue
        expiryAddress: ExpiryQueue
        autoCreateExpiryResources: true
        maxDeliveryAttempts: 5
        redeliveryDelay: 5000
        maxRedeliveryDelay: 60000
        redeliveryMultiplier: 2.0
        messageCounterHistoryDayLimit: 10

  # Admin user
  adminUser: admin
  adminPassword: AdminP@ss
```

```bash
# Create namespace and apply
kubectl create namespace messaging
kubectl apply -f activemq-cluster.yaml

# Check status
kubectl get activemqartemis -n messaging
kubectl get pods -n messaging -l ActiveMQArtemis=activemq-prod
```

## Step 3: Configure Security

```yaml
# activemq-security.yaml - ActiveMQ security configuration
apiVersion: broker.amq.io/v1beta1
kind: ActiveMQArtemisSecurity
metadata:
  name: activemq-security
  namespace: messaging
spec:
  loginModules:
    propertiesLoginModules:
      - name: prop-module
        users:
          - name: admin
            password: AdminP@ss
            roles:
              - admin
          - name: appuser
            password: AppUserP@ss
            roles:
              - app-role

  securityDomains:
    brokerDomain:
      name: activemq
      loginModules:
        - name: prop-module
          flag: sufficient

  securitySettings:
    broker:
      - match: "#"
        permissions:
          - operationType: consume
            roles:
              - admin
              - app-role
          - operationType: send
            roles:
              - admin
              - app-role
          - operationType: browse
            roles:
              - admin
              - app-role
          - operationType: createAddress
            roles:
              - admin
          - operationType: createNonDurableQueue
            roles:
              - admin
              - app-role
          - operationType: createDurableQueue
            roles:
              - admin
```

## Step 4: Create Queues and Topics

```yaml
# activemq-addresses.yaml - Queue/topic definitions
apiVersion: broker.amq.io/v1beta1
kind: ActiveMQArtemisAddress
metadata:
  name: orders-queue
  namespace: messaging
spec:
  addressName: orders
  queueName: orders.processor
  routingType: anycast
  removeFromBrokerOnDelete: false
---
apiVersion: broker.amq.io/v1beta1
kind: ActiveMQArtemisAddress
metadata:
  name: events-topic
  namespace: messaging
spec:
  addressName: events
  queueName: events.subscriber
  routingType: multicast
  removeFromBrokerOnDelete: false
```

## Step 5: Configure Application Connection

```yaml
# app-config.yaml - Application connecting to ActiveMQ Artemis
apiVersion: v1
kind: ConfigMap
metadata:
  name: activemq-config
  namespace: production
data:
  # AMQP connection string
  ACTIVEMQ_BROKER_URL: "amqp://appuser:AppUserP@ss@activemq-prod-hdls-svc.messaging.svc.cluster.local:5672"
  # JMS core connection
  ACTIVEMQ_CORE_URL: "tcp://activemq-prod-hdls-svc.messaging.svc.cluster.local:61616?user=appuser&password=AppUserP@ss"
  ACTIVEMQ_QUEUE: "orders"
```

## Step 6: Access Management Console

```bash
# Port forward to ActiveMQ management console
kubectl port-forward -n messaging svc/activemq-prod-wconsj-svc 8161:8161

# Access at: http://localhost:8161/console
# Default credentials: admin/AdminP@ss
```

## Step 7: Monitor ActiveMQ

```yaml
# activemq-servicemonitor.yaml - Prometheus scraping
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: activemq-monitor
  namespace: cattle-monitoring-system
  labels:
    release: rancher-monitoring
spec:
  namespaceSelector:
    matchNames:
      - messaging
  selector:
    matchLabels:
      app: activemq-prod
  endpoints:
    - port: wconsj
      path: /metrics/
      interval: 30s
```

## Troubleshooting

```bash
# Check broker status via Artemis CLI
kubectl exec -n messaging activemq-prod-ss-0 -- \
  ./bin/artemis queue stat \
  --url tcp://activemq-prod-hdls-svc.messaging.svc.cluster.local:61616 \
  --user admin \
  --password AdminP@ss

# Check DLQ
kubectl exec -n messaging activemq-prod-ss-0 -- \
  ./bin/artemis browse \
  --url tcp://activemq-prod-hdls-svc.messaging.svc.cluster.local:61616 \
  --user admin \
  --password AdminP@ss \
  --destination DLQ

# View logs
kubectl logs -n messaging activemq-prod-ss-0 --tail=100

# Check journal files
kubectl exec -n messaging activemq-prod-ss-0 -- \
  ls /opt/activemq-artemis/data/
```

## Conclusion

ActiveMQ Artemis on Rancher provides enterprise-grade messaging with JMS compatibility and multi-protocol support. The ArtemisCloud operator enables declarative Kubernetes management of broker configuration, security, and scaling. For organizations with existing JMS applications or requirements for protocol flexibility (AMQP, STOMP, MQTT), Artemis is an excellent choice that provides a clear migration path from legacy ActiveMQ while offering modern high-availability features.
