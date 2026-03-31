# How to Configure Kafka TLS for Dapr Pub/Sub

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Kafka, TLS, Security, Pub/Sub, Certificate

Description: Configure mutual TLS (mTLS) and one-way TLS for Dapr Kafka pub/sub connections, including certificate management with Kubernetes secrets and cert-manager.

---

## Overview

TLS encrypts data in transit between Dapr sidecars and Kafka brokers. Kafka supports one-way TLS (server authentication only) and mutual TLS (mTLS, where both sides present certificates). mTLS is preferred for production as it prevents unauthorized clients from connecting even with network access.

## One-Way TLS Configuration

Server certificate authentication only:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: kafka-pubsub
  namespace: default
spec:
  type: pubsub.kafka
  version: v1
  metadata:
    - name: brokers
      value: "kafka-broker:9093"
    - name: authType
      value: "none"
    - name: tlsEnabled
      value: "true"
    - name: caCert
      secretKeyRef:
        name: kafka-tls-secret
        key: ca.crt
```

## Mutual TLS Configuration

Both client and server authenticate with certificates:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: kafka-pubsub
  namespace: default
spec:
  type: pubsub.kafka
  version: v1
  metadata:
    - name: brokers
      value: "kafka-broker:9093"
    - name: authType
      value: "mtls"
    - name: caCert
      secretKeyRef:
        name: kafka-tls-secret
        key: ca.crt
    - name: clientCert
      secretKeyRef:
        name: kafka-tls-secret
        key: tls.crt
    - name: clientKey
      secretKeyRef:
        name: kafka-tls-secret
        key: tls.key
```

## Generating Certificates with cert-manager

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: kafka-client-cert
  namespace: default
spec:
  secretName: kafka-tls-secret
  duration: 8760h
  renewBefore: 720h
  issuerRef:
    name: kafka-ca-issuer
    kind: ClusterIssuer
  commonName: dapr-kafka-client
  dnsNames:
    - dapr-kafka-client
  usages:
    - client auth
    - digital signature
    - key encipherment
```

## Manual Certificate Generation

For testing without cert-manager:

```bash
# Generate CA
openssl genrsa -out ca.key 4096
openssl req -new -x509 -key ca.key -out ca.crt -days 3650 \
  -subj "/CN=Kafka CA"

# Generate client key and cert
openssl genrsa -out client.key 2048
openssl req -new -key client.key -out client.csr \
  -subj "/CN=dapr-kafka-client"
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out client.crt -days 365

# Create Kubernetes secret
kubectl create secret generic kafka-tls-secret \
  --from-file=ca.crt=ca.crt \
  --from-file=tls.crt=client.crt \
  --from-file=tls.key=client.key
```

## Kafka Broker mTLS Configuration

```properties
# server.properties
ssl.client.auth=required
ssl.truststore.location=/etc/kafka/ssl/kafka.server.truststore.jks
ssl.truststore.password=truststorepassword
ssl.keystore.location=/etc/kafka/ssl/kafka.server.keystore.jks
ssl.keystore.password=keystorepassword
listeners=SSL://0.0.0.0:9093
```

Import the client CA into the broker truststore:

```bash
keytool -import -alias dapr-client-ca \
  -file ca.crt \
  -keystore kafka.server.truststore.jks \
  -storepass truststorepassword -noprompt
```

## Combining TLS with SASL

```yaml
metadata:
  - name: authType
    value: "password"
  - name: saslUsername
    secretKeyRef:
      name: kafka-secret
      key: username
  - name: saslPassword
    secretKeyRef:
      name: kafka-secret
      key: password
  - name: saslMechanism
    value: "SCRAM-SHA-512"
  - name: tlsEnabled
    value: "true"
  - name: caCert
    secretKeyRef:
      name: kafka-tls-secret
      key: ca.crt
```

## Summary

Configuring TLS for Dapr Kafka pub/sub requires setting `tlsEnabled: "true"` and providing the CA certificate. For mTLS, set `authType: "mtls"` and additionally provide `clientCert` and `clientKey` from a Kubernetes secret. Use cert-manager to automate certificate rotation so that client certificates renew before expiry without manual intervention. Combine mTLS with SASL/SCRAM for defense-in-depth authentication.
