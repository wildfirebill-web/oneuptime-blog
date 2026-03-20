# How to Set Up K3s on IoT Gateways

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: K3s, Kubernetes, IoT, Edge Computing, Raspberry Pi, Gateway, DevOps

Description: Learn how to deploy K3s on IoT gateway devices to orchestrate containerized edge workloads close to IoT sensors and devices.

## Introduction

IoT gateways act as the bridge between IoT devices (sensors, actuators, cameras) and cloud infrastructure. Running K3s on these gateways enables containerized orchestration of data processing, protocol translation, and local AI/ML inference close to where data is generated. This guide covers setting up K3s on IoT gateways, deploying typical IoT workloads, and managing connectivity to IoT devices.

## Typical IoT Gateway Hardware

- **Raspberry Pi 4** (4GB RAM): Excellent for most IoT gateway use cases
- **NVIDIA Jetson Nano**: GPU-accelerated inference at the edge
- **BeagleBone Black**: Industrial-grade connectivity
- **Intel NUC**: Higher performance x86 edge computing
- **Industrial PCs**: Fanless, DIN-rail mountable options

## Step 1: Prepare the IoT Gateway

```bash
# On Raspberry Pi 4 running Ubuntu 22.04 LTS (ARM64)

# Update and install dependencies
apt-get update && apt-get upgrade -y
apt-get install -y \
  curl \
  git \
  python3 \
  mosquitto-clients  # MQTT client for testing

# Enable memory cgroups (required for K3s on Raspberry Pi)
# Add to /boot/firmware/cmdline.txt (not /boot/cmdline.txt on Pi4)
sed -i '$ s/$/ cgroup_memory=1 cgroup_enable=memory cgroup_enable=cpuset/' \
  /boot/firmware/cmdline.txt

# Reboot to apply cgroup changes
reboot
```

## Step 2: Install K3s on the Gateway

```yaml
# /etc/rancher/k3s/config.yaml
# IoT Gateway K3s configuration

# Minimal components for IoT gateway
disable:
  - traefik       # IoT devices rarely need web routing
  - metrics-server # Optional - disable if RAM is tight

# Resource allocation for IoT workloads
kubelet-arg:
  - "max-pods=20"  # IoT gateways run fewer, smaller workloads
  - "system-reserved=cpu=200m,memory=256Mi"
  - "kube-reserved=cpu=100m,memory=128Mi"
  - "eviction-hard=memory.available<100Mi"

# Node labels for IoT topology
node-label:
  - "role=iot-gateway"
  - "location=warehouse-floor-1"
  - "hardware=raspberrypi-4"
```

Install K3s:

```bash
curl -sfL https://get.k3s.io | sh -

# Verify K3s is running
systemctl status k3s
kubectl get nodes
```

## Step 3: Deploy MQTT Broker for IoT Messaging

MQTT is the standard protocol for IoT device communication:

```yaml
# mqtt-broker.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mosquitto-broker
  namespace: iot
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mosquitto
  template:
    metadata:
      labels:
        app: mosquitto
    spec:
      nodeSelector:
        role: iot-gateway
      containers:
        - name: mosquitto
          image: eclipse-mosquitto:2.0
          ports:
            - containerPort: 1883   # MQTT
            - containerPort: 8883   # MQTT over TLS
            - containerPort: 9001   # WebSocket
          volumeMounts:
            - name: config
              mountPath: /mosquitto/config
            - name: data
              mountPath: /mosquitto/data
      volumes:
        - name: config
          configMap:
            name: mosquitto-config
        - name: data
          hostPath:
            path: /data/mosquitto
            type: DirectoryOrCreate
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: mosquitto-config
  namespace: iot
data:
  mosquitto.conf: |
    listener 1883
    allow_anonymous true
    persistence true
    persistence_location /mosquitto/data/
    log_type error
    log_type warning
    log_type notice
    log_type information
---
apiVersion: v1
kind: Service
metadata:
  name: mosquitto-svc
  namespace: iot
spec:
  selector:
    app: mosquitto
  ports:
    - name: mqtt
      port: 1883
      targetPort: 1883
      nodePort: 31883
  type: NodePort
```

## Step 4: Deploy Node-RED for IoT Data Processing

Node-RED is a popular flow-based programming tool for IoT:

```yaml
# node-red.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-red
  namespace: iot
spec:
  replicas: 1
  selector:
    matchLabels:
      app: node-red
  template:
    metadata:
      labels:
        app: node-red
    spec:
      nodeSelector:
        role: iot-gateway
      containers:
        - name: node-red
          image: nodered/node-red:3.1
          ports:
            - containerPort: 1880
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          volumeMounts:
            - name: node-red-data
              mountPath: /data
          env:
            - name: TZ
              value: "America/New_York"
      volumes:
        - name: node-red-data
          hostPath:
            path: /data/node-red
            type: DirectoryOrCreate
---
apiVersion: v1
kind: Service
metadata:
  name: node-red-svc
  namespace: iot
spec:
  selector:
    app: node-red
  ports:
    - port: 1880
      targetPort: 1880
      nodePort: 31880
  type: NodePort
```

## Step 5: Access Host GPIO and Serial Devices

IoT gateways often need access to hardware interfaces:

```yaml
# gpio-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpio-sensor-reader
  namespace: iot
spec:
  nodeSelector:
    role: iot-gateway
  hostNetwork: true  # Access host network for direct device communication
  containers:
    - name: sensor-reader
      image: myregistry/sensor-reader:v1
      securityContext:
        privileged: true  # Required for GPIO access
      resources:
        requests:
          memory: 64Mi
        limits:
          memory: 128Mi
      volumeMounts:
        # Mount GPIO device filesystem
        - name: gpio
          mountPath: /sys/class/gpio
        # Mount serial device
        - name: serial
          mountPath: /dev/ttyUSB0
        # Mount I2C
        - name: i2c
          mountPath: /dev/i2c-1
  volumes:
    - name: gpio
      hostPath:
        path: /sys/class/gpio
    - name: serial
      hostPath:
        path: /dev/ttyUSB0
    - name: i2c
      hostPath:
        path: /dev/i2c-1
```

## Step 6: Deploy InfluxDB for Time-Series Data

IoT sensor data is naturally time-series:

```yaml
# influxdb.yaml
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: influxdb
  namespace: iot
spec:
  replicas: 1
  selector:
    matchLabels:
      app: influxdb
  template:
    metadata:
      labels:
        app: influxdb
    spec:
      nodeSelector:
        role: iot-gateway
      containers:
        - name: influxdb
          image: influxdb:2.7-alpine
          ports:
            - containerPort: 8086
          resources:
            requests:
              memory: 128Mi
            limits:
              memory: 512Mi
          env:
            - name: DOCKER_INFLUXDB_INIT_MODE
              value: setup
            - name: DOCKER_INFLUXDB_INIT_USERNAME
              value: admin
            - name: DOCKER_INFLUXDB_INIT_PASSWORD
              value: securepassword
            - name: DOCKER_INFLUXDB_INIT_ORG
              value: iot-org
            - name: DOCKER_INFLUXDB_INIT_BUCKET
              value: sensors
          volumeMounts:
            - name: influxdb-data
              mountPath: /var/lib/influxdb2
      volumes:
        - name: influxdb-data
          hostPath:
            path: /data/influxdb
            type: DirectoryOrCreate
```

## Step 7: OTA (Over-the-Air) Updates via K3s

Use K3s's System Upgrade Controller for OTA updates:

```yaml
# iot-app-update-plan.yaml
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: iot-app-update
  namespace: system-upgrade
spec:
  # Monitor for new versions from the local registry
  channel: http://iot-update-server/api/latest-version
  serviceAccountName: system-upgrade
  concurrency: 1
  nodeSelector:
    matchLabels:
      role: iot-gateway
  upgrade:
    image: myregistry/iot-updater
```

## Conclusion

K3s on IoT gateways enables powerful containerized data processing at the edge, close to sensors and devices. The key components for IoT gateway deployments are an MQTT broker for device communication, a flow engine like Node-RED for data processing, and time-series storage for sensor data. Hardware access requires privileged pods or device plugins for GPIO, serial, and I2C interfaces. K3s's light resource footprint makes it an excellent choice for resource-constrained gateway hardware like Raspberry Pi.
