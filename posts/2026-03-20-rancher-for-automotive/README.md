# How to Set Up Rancher for Automotive

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Automotive, ADAS, Edge, Connected Vehicle, Kubernetes, Safety

Description: Configure Rancher for automotive workloads including connected vehicle data processing, ADAS (Advanced Driver Assistance Systems) testing, OTA update infrastructure, and edge computing for in-vehicle systems.

## Introduction

The automotive industry's digital transformation requires Kubernetes for connected vehicle data pipelines, ADAS algorithm testing and validation, over-the-air software update infrastructure, and in-vehicle edge computing. Rancher manages the Kubernetes clusters across factory production systems, test track compute, cloud analytics, and edge vehicle gateways.

## Automotive Platform Architecture

```
                        Cloud (AWS/Azure)
┌─────────────────────────────────────────────────────┐
│  Rancher Management                                  │
│  Connected Vehicle Data Platform                     │
│  ADAS ML Training (GPU clusters)                    │
│  OTA Update Distribution                            │
└─────────────────────┬───────────────────────────────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
┌───────▼──────┐ ┌────▼──────┐ ┌───▼──────────────┐
│ Test Track   │ │ Production │ │ Vehicle Edge     │
│ Compute      │ │ Factory    │ │ Gateway (K3s)    │
│ (ADAS sim)   │ │ Cluster    │ │                  │
└──────────────┘ └───────────┘ └──────────────────┘
```

## Step 1: ADAS Algorithm Testing Infrastructure

```yaml
# GPU cluster for autonomous driving simulations
# Using CARLA or similar simulation platform

apiVersion: apps/v1
kind: Deployment
metadata:
  name: carla-simulator
  namespace: adas-testing
spec:
  replicas: 4    # Parallel simulation runs
  template:
    spec:
      containers:
        - name: carla
          image: carlasimulator/carla:0.9.15
          args:
            - "/bin/bash"
            - CarlaUE4.sh
            - -RenderOffScreen
            - -carla-server
            - -nosound
          resources:
            limits:
              nvidia.com/gpu: "1"
              memory: "16Gi"
              cpu: "8"
          ports:
            - containerPort: 2000    # CARLA server port
---
# ADAS testing job
apiVersion: batch/v1
kind: Job
metadata:
  name: adas-scenario-test
  namespace: adas-testing
spec:
  parallelism: 10    # Run 10 scenarios in parallel
  template:
    spec:
      containers:
        - name: scenario-runner
          image: myregistry/adas-test-suite:v2.1.0
          env:
            - name: CARLA_HOST
              value: carla-simulator
            - name: SCENARIO_FILE
              value: /scenarios/highway-merge.xml
          resources:
            limits:
              memory: "4Gi"
              cpu: "2"
```

## Step 2: Connected Vehicle Data Pipeline

```yaml
# Kafka for high-throughput vehicle telemetry ingestion
helm install kafka bitnami/kafka \
  --namespace vehicle-data \
  --set replicaCount=3 \
  --set controller.replicaCount=3 \
  --set persistence.size=500Gi

# Vehicle data processor (Flink/Spark)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vehicle-telemetry-processor
  namespace: vehicle-data
spec:
  replicas: 6
  template:
    spec:
      containers:
        - name: processor
          image: myregistry/vehicle-processor:latest
          env:
            - name: KAFKA_BOOTSTRAP
              value: kafka.vehicle-data.svc:9092
            - name: KAFKA_TOPIC
              value: vehicle-telemetry
            - name: TIMESCALEDB_URL
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: url
          resources:
            requests:
              cpu: "1"
              memory: "2Gi"
```

## Step 3: OTA Update Management

```yaml
# Eclipse Hawkbit for OTA update management
helm install hawkbit myregistry/hawkbit \
  --namespace ota-updates \
  --set persistence.enabled=true \
  --set rabbitmq.enabled=true

# OTA update delivery job
apiVersion: batch/v1
kind: Job
metadata:
  name: deliver-sw-update-v3-2-1
  namespace: ota-updates
spec:
  template:
    spec:
      containers:
        - name: update-distributor
          image: myregistry/ota-distributor:latest
          env:
            - name: HAWKBIT_URL
              value: "http://hawkbit.ota-updates.svc:8080"
            - name: CAMPAIGN_ID
              value: "sw-v3.2.1-fleet-A"
            - name: ROLLOUT_PERCENTAGE
              value: "5"    # 5% canary rollout first
```

## Step 4: Vehicle Edge Computing with K3s

```bash
# K3s on vehicle gateway computer (ARM64 hardware)
# e.g., NVIDIA Jetson AGX Orin

curl -sfL https://get.k3s.io | \
  INSTALL_K3S_CHANNEL=stable \
  INSTALL_K3S_EXEC="
    --disable traefik
    --node-label vehicle.vin=${VIN}
    --node-label vehicle.model=${MODEL}
    --private-registry=/etc/rancher/k3s/registries.yaml
  " sh -

# Vehicle edge workloads:
# - Sensor fusion preprocessing
# - Edge inference (object detection)
# - CAN bus data collection
# - V2X communication
```

## Step 5: Functional Safety Considerations

```yaml
# Kubernetes is not certified for safety-critical functions
# Safety-critical: handled by dedicated ECUs
# Kubernetes role: non-safety infotainment, data collection, OTA

# Use pod priority for critical vehicle services
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: vehicle-critical
value: 1000000
preemptionPolicy: PreemptLowerPriority
---
# Watchdog for vehicle edge process monitoring
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: vehicle-watchdog
  namespace: vehicle-edge
spec:
  template:
    spec:
      containers:
        - name: watchdog
          image: myregistry/vehicle-watchdog:1.0.0
          securityContext:
            privileged: true    # Required for hardware watchdog access
          volumeMounts:
            - name: watchdog-dev
              mountPath: /dev/watchdog
      volumes:
        - name: watchdog-dev
          hostPath:
            path: /dev/watchdog
            type: CharDevice
```

## Step 6: Factory Production Cluster

```yaml
# Quality inspection ML inference at factory
apiVersion: apps/v1
kind: Deployment
metadata:
  name: quality-inspection
  namespace: factory-ai
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: vision-model
          image: myregistry/qc-vision:v1.5.0
          resources:
            limits:
              nvidia.com/gpu: "1"
              memory: "8Gi"
          env:
            - name: CAMERA_STREAM
              value: "rtsp://camera-station-12:8554/stream"
            - name: DEFECT_THRESHOLD
              value: "0.92"
            - name: ALERTING_WEBHOOK
              value: "http://mes-system.factory.internal/defects"
```

## Conclusion

Rancher orchestrates automotive Kubernetes infrastructure from cloud data centers down to vehicle-embedded K3s gateways. ADAS teams run parallel simulation workloads on GPU clusters, OTA updates reach millions of vehicles via Eclipse Hawkbit, and factory quality inspection runs ML inference on assembly line cameras. The critical safety systems remain on dedicated certified ECUs, while Rancher manages the surrounding digital infrastructure for connected, software-defined vehicles.
