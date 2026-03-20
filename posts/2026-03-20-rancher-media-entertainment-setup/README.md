# How to Set Up Rancher for Media and Entertainment

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: rancher, media, entertainment, gpu, rendering, kubernetes, streaming

Description: A guide to configuring Rancher for media and entertainment workloads, covering GPU-accelerated rendering, video transcoding, content delivery, and burst scaling.

## Overview

Media and entertainment organizations use Kubernetes for video transcoding, real-time rendering, content delivery optimization, and live streaming pipelines. These workloads have unique requirements: GPU acceleration, high storage throughput, burst scaling for live events, and low-latency content delivery. Rancher provides the multi-cluster management capabilities to handle these demanding workloads. This guide covers key configurations.

## Architecture

```
Production Cluster (RKE2 + GPU nodes)
├── Video Transcoding Farm
├── Rendering Pipeline
└── Content Processing

Delivery Cluster (CDN integration)
├── Origin Server
├── Video Packaging
└── DRM Management

Streaming Cluster (Low-latency)
├── Live Ingest
├── Adaptive Bitrate
└── Player API
```

## Step 1: GPU Node Configuration

Configure worker nodes for GPU-accelerated media workloads:

```bash
# Install NVIDIA GPU Operator via Helm
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm install gpu-operator nvidia/gpu-operator \
  --namespace gpu-operator \
  --create-namespace \
  --set driver.enabled=true \
  --set toolkit.enabled=true

# Verify GPU nodes are labeled
kubectl get nodes -l accelerator=nvidia-gpu
```

```yaml
# Node label GPU worker nodes
apiVersion: v1
kind: Node
metadata:
  name: gpu-worker-01
  labels:
    accelerator: nvidia-gpu
    gpu-type: a100           # GPU model label for scheduling
    gpu-count: "8"
```

## Step 2: Video Transcoding Pipeline

```yaml
# Video transcoding job with GPU acceleration
apiVersion: batch/v1
kind: Job
metadata:
  name: transcode-movie-4k
  namespace: media-processing
spec:
  parallelism: 4    # Parallel transcoding jobs
  completions: 16   # Total segments to transcode
  template:
    spec:
      nodeSelector:
        accelerator: nvidia-gpu
      containers:
        - name: ffmpeg-transcoder
          image: registry.studio.internal/ffmpeg-gpu:v6.0
          env:
            - name: INPUT_URL
              value: "s3://raw-footage/movie-4k-raw.mov"
            - name: OUTPUT_BUCKET
              value: "s3://transcoded/movie-4k/"
            - name: PRESET
              value: "h264_nvenc_4k_hls"   # NVENC hardware encoder preset
          resources:
            limits:
              nvidia.com/gpu: 1
              memory: "16Gi"
          command:
            - ffmpeg
            - -hwaccel
            - cuda
            - -i
            - $(INPUT_URL)
            - -c:v
            - h264_nvenc
            - -preset
            - slow
            - -crf
            - "18"
            - $(OUTPUT_BUCKET)segment_%03d.ts
      restartPolicy: OnFailure
```

## Step 3: Burst Scaling for Live Events

Configure Horizontal Pod Autoscaler for live streaming spikes:

```yaml
# HPA for live streaming ingest (scales during live events)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: live-ingest-hpa
  namespace: streaming
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: live-ingest-server
  minReplicas: 3
  maxReplicas: 100   # Scale up to 100 for major live events
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
    - type: Pods
      pods:
        metric:
          name: active_streams
        target:
          type: AverageValue
          averageValue: "500"   # Max 500 streams per pod
```

## Step 4: High-Throughput Storage for Media

Configure Longhorn with optimal settings for large video files:

```yaml
# StorageClass optimized for large sequential video reads/writes
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-media
provisioner: driver.longhorn.io
parameters:
  numberOfReplicas: "2"      # 2 replicas for media (balance cost/HA)
  staleReplicaTimeout: "2880"
  diskSelector: "ssd"        # Use SSD-labeled disks for video I/O
  nodeSelector: "media-storage"
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

## Step 5: Rendering Farm Job Queue

```yaml
# Kubernetes Job for 3D rendering with blender
apiVersion: batch/v1
kind: Job
metadata:
  name: render-scene-001
  namespace: rendering-farm
spec:
  completions: 250    # 250 frames to render
  parallelism: 25     # 25 concurrent render nodes
  template:
    spec:
      nodeSelector:
        workload: rendering
        accelerator: nvidia-gpu
      containers:
        - name: blender-render
          image: registry.studio.internal/blender:v3.6
          env:
            - name: SCENE_FILE
              value: "/render/projects/movie-scene-001.blend"
            - name: FRAME_START
              value: "1"
            - name: FRAME_END
              value: "250"
          resources:
            requests:
              memory: "32Gi"
              nvidia.com/gpu: 2
            limits:
              memory: "64Gi"
              nvidia.com/gpu: 2
          volumeMounts:
            - name: render-projects
              mountPath: /render/projects
            - name: render-output
              mountPath: /render/output
      volumes:
        - name: render-projects
          persistentVolumeClaim:
            claimName: render-projects-pvc
        - name: render-output
          persistentVolumeClaim:
            claimName: render-output-pvc
      restartPolicy: OnFailure
```

## Step 6: Content Protection with DRM

```yaml
# DRM key server deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: drm-key-server
  namespace: content-delivery
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: widevine-keyserver
          image: registry.studio.internal/drm-server:v2.1.0
          env:
            - name: WIDEVINE_PROVIDER_ID
              valueFrom:
                secretKeyRef:
                  name: drm-credentials
                  key: provider-id
          resources:
            requests:
              cpu: "1"
              memory: "2Gi"
```

## Step 7: Monitoring Media Pipeline KPIs

```yaml
# PrometheusRule for media pipeline monitoring
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: media-pipeline-alerts
  namespace: cattle-monitoring-system
spec:
  groups:
    - name: transcoding
      rules:
        - alert: TranscodingQueueDepth
          expr: media_transcode_queue_depth > 100
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Transcoding queue depth above 100 jobs"
        - alert: LiveStreamBuffering
          expr: live_stream_buffer_seconds < 2
          for: 30s
          labels:
            severity: critical
          annotations:
            summary: "Live stream buffer critically low"
```

## Step 8: Multi-CDN Management

```yaml
# ConfigMap for CDN routing based on geography
apiVersion: v1
kind: ConfigMap
metadata:
  name: cdn-routing
  namespace: content-delivery
data:
  routing.json: |
    {
      "regions": {
        "us-east": "cdn-akamai.streaming.com",
        "us-west": "cdn-cloudfront.streaming.com",
        "europe": "cdn-fastly.streaming.com",
        "asia-pacific": "cdn-cloudflare.streaming.com"
      }
    }
```

## Conclusion

Rancher provides a powerful platform for media and entertainment workloads with its GPU scheduling, burst autoscaling, and multi-cluster management. The ability to run GPU-accelerated transcoding and rendering jobs alongside streaming infrastructure on the same Rancher-managed platform simplifies operations. For live events requiring massive scale-out, Kubernetes HPA and cluster autoscaler integrate seamlessly with Rancher to handle spikes that traditional infrastructure cannot address cost-effectively.
