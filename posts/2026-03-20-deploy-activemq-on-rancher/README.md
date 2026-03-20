# How to Deploy ActiveMQ on Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, ActiveMQ, Kubernetes, JMS, Message Broker, Helm

Description: Deploy Apache ActiveMQ Artemis on Rancher with persistent storage, management console access, and JMS queue configuration.

## Introduction

Apache ActiveMQ Artemis is the next-generation ActiveMQ broker supporting JMS, AMQP, STOMP, and MQTT. It's commonly used in Java enterprise environments. This guide deploys Artemis on Rancher using the official Helm chart.

## Prerequisites

- Rancher cluster with `kubectl` and `helm`
- StorageClass for message persistence

## Step 1: Add Repository

```bash
helm repo add activemq-artemis https://arkmq-org.github.io/activemq-artemis-operator/
helm repo update
```

## Step 2: Deploy with Custom Values

For a straightforward deployment, use the Bitnami Artemis chart:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami

# Create values file
cat > artemis-values.yaml << 'EOF'
auth:
  enabled: true
  user: admin
  password: "securepassword"

persistence:
  enabled: true
  storageClass: "longhorn"
  size: 20Gi

resources:
  requests:
    memory: "512Mi"
    cpu: "250m"
  limits:
    memory: "2Gi"
    cpu: "1"

replicaCount: 2   # Active-passive HA pair
EOF
```

## Step 3: Deploy ActiveMQ Artemis

```bash
kubectl create namespace messaging

helm install artemis bitnami/activemq \
  --namespace messaging \
  --values artemis-values.yaml
```

## Step 4: Verify Deployment

```bash
kubectl get pods -n messaging
kubectl logs -n messaging deployment/artemis -f
```

## Step 5: Access Management Console

```bash
# Port-forward to the web console
kubectl port-forward svc/artemis -n messaging 8161:8161

# Open http://localhost:8161/console
# Login with admin/securepassword
```

## Step 6: Create Queues via Broker XML

Configure queues programmatically via a ConfigMap:

```yaml
# artemis-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: artemis-config
  namespace: messaging
data:
  broker.xml: |
    <configuration>
      <core>
        <addresses>
          <address name="order.queue">
            <anycast>
              <queue name="order.queue">
                <durable>true</durable>
              </queue>
            </anycast>
          </address>
          <address name="notification.topic">
            <multicast/>   <!-- Pub/Sub topic -->
          </address>
        </addresses>
      </core>
    </configuration>
```

## Step 7: Connect Java Applications

```java
// Java JMS connection example
ConnectionFactory factory = new ActiveMQConnectionFactory(
    "tcp://artemis.messaging.svc.cluster.local:61616"
);
Connection connection = factory.createConnection("admin", "securepassword");
connection.start();

Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
Queue queue = session.createQueue("order.queue");
MessageProducer producer = session.createProducer(queue);
```

## Conclusion

ActiveMQ Artemis is running on Rancher with persistent storage and a management console. The multi-protocol support (JMS, AMQP, STOMP, MQTT) makes it versatile for polyglot microservice environments.
