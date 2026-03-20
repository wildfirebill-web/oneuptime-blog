# How to Set Up Rancher for Media and Entertainment

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Media, Entertainment, Video Processing, CDN, Kubernetes, Streaming

Description: Configure Rancher for media and entertainment workloads including video transcoding, content delivery, live streaming infrastructure, and high-throughput storage systems for broadcast and OTT platforms.

## Introduction

Media and entertainment Kubernetes workloads are characterized by high throughput, massive storage requirements, GPU-accelerated transcoding, and traffic spikes during live events. OTT streaming platforms, broadcast systems, and digital asset management all run effectively on Rancher-managed Kubernetes, leveraging burst scaling and object storage integration.

## Media Platform Architecture

```
┌──────────────────────────────────────────────────┐
│  Rancher Management                               │
└──────────────────────────┬───────────────────────┘
          ┌─────────────────┼─────────────────┐
          │                 │                 │
  ┌───────▼────────┐  ┌─────▼──────┐  ┌──────▼──────────┐
  │ Ingest/         │  │ Transcode  │  │ Distribution    │
  │ Processing      │  │ Cluster    │  │ / CDN Origin    │
  │ Cluster         │  │ (GPU)      │  │ Cluster         │
  └─────────────────┘  └────────────┘  └─────────────────┘
  Live feed ingest      GPU transcode   HLS/DASH serving
  DAM, metadata         HLS packaging   Cache warming
```

## Step 1: GPU Transcoding Cluster

```yaml
# NVIDIA GPU operator for transcoding nodes
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm install gpu-operator nvidia/gpu-operator \
  --namespace gpu-operator \
  --create-namespace

# Video transcoding job using FFmpeg with GPU acceleration
apiVersion: batch/v1
kind: Job
metadata:
  name: transcode-4k-to-1080p
  namespace: media-processing
spec:
  parallelism: 4       # Parallel transcoding
  template:
    spec:
      containers:
        - name: ffmpeg
          image: linuxserver/ffmpeg:latest
          command:
            - ffmpeg
            - "-hwaccel"
            - "cuda"
            - "-i"
            - "/input/source-4k.mov"
            - "-vf"
            - "scale=1920:1080"
            - "-c:v"
            - "h264_nvenc"
            - "-preset"
            - "fast"
            - "/output/1080p.mp4"
          resources:
            limits:
              nvidia.com/gpu: "1"
              memory: "8Gi"
          volumeMounts:
            - name: media-input
              mountPath: /input
            - name: media-output
              mountPath: /output
      volumes:
        - name: media-input
          persistentVolumeClaim:
            claimName: raw-media-pvc
        - name: media-output
          persistentVolumeClaim:
            claimName: processed-media-pvc
```

## Step 2: HLS Packaging and Streaming

```yaml
# HLS packager deployment for adaptive bitrate streaming
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hls-packager
  namespace: streaming
spec:
  replicas: 4
  template:
    spec:
      containers:
        - name: packager
          image: shaka/packager:release-v2.6.1
          command:
            - packager
            - "in=rtmp://ingest:1935/live/$STREAM_KEY,stream=video,init_segment=/out/init.mp4,segment_template=/out/$Number$.m4s"
            - "--hls_master_playlist_output=/out/master.m3u8"
            - "--segment_duration=2"
            - "--hls_segment_duration=2"
          resources:
            requests:
              cpu: "2"
              memory: "2Gi"
            limits:
              cpu: "4"
              memory: "4Gi"
```

## Step 3: Object Storage for Media Assets

```yaml
# MinIO for media asset storage
helm install minio minio/minio \
  --namespace media-storage \
  --set mode=distributed \
  --set replicas=4 \
  --set drivesPerNode=4 \
  --set persistence.size=10Ti \
  --set persistence.storageClass=local-nvme

# Lifecycle policy: move content to archival after 90 days
mc ilm rule add \
  --transition-days "90" \
  --storage-class "GLACIER" \
  myminio/media-assets
```

## Step 4: KEDA-Based Autoscaling for Live Events

```yaml
# Scale transcoding workers based on queue depth
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: transcode-worker-scaler
  namespace: media-processing
spec:
  scaleTargetRef:
    name: transcode-workers
  minReplicaCount: 2
  maxReplicaCount: 50    # Scale to 50 during live events
  triggers:
    - type: rabbitmq
      metadata:
        queueName: transcoding-jobs
        queueLength: "5"    # 1 worker per 5 jobs in queue
        hostFromEnv: RABBITMQ_URI
```

## Step 5: CDN Origin Serving

```yaml
# NGINX-based CDN origin with caching
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-cdn-config
  namespace: cdn-origin
data:
  nginx.conf: |
    proxy_cache_path /cache/media levels=1:2
      keys_zone=media_cache:100m
      max_size=500g
      inactive=7d
      use_temp_path=off;

    server {
      listen 80;
      location ~* \.(m3u8|m4s|mp4|ts)$ {
        proxy_cache media_cache;
        proxy_cache_valid 200 1d;
        proxy_cache_use_stale error timeout updating;
        proxy_pass http://minio.media-storage.svc:9000;
        add_header Cache-Control "public, max-age=86400";
        add_header X-Cache-Status $upstream_cache_status;
      }
    }
```

## Step 6: Event-Driven Live Streaming

```yaml
# RTMP ingest with automatic HLS packaging
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rtmp-ingest
  namespace: streaming
spec:
  template:
    spec:
      containers:
        - name: nginx-rtmp
          image: tiangolo/nginx-rtmp
          ports:
            - containerPort: 1935    # RTMP
            - containerPort: 8080    # HLS output
          resources:
            requests:
              cpu: "4"
              memory: "4Gi"
---
# LoadBalancer for RTMP ingest
apiVersion: v1
kind: Service
metadata:
  name: rtmp-ingest-lb
spec:
  type: LoadBalancer
  ports:
    - port: 1935
      targetPort: 1935
      name: rtmp
```

## Conclusion

Rancher manages media and entertainment Kubernetes clusters that demand GPU transcoding, high-throughput storage, and massive autoscaling during live events. KEDA-based autoscaling handles traffic spikes from thousands to millions of concurrent viewers. MinIO provides on-premises S3-compatible media storage, while NGINX handles CDN origin serving with aggressive caching. The combination of GPU operator, HLS packaging, and object storage makes Rancher a complete platform for broadcast, OTT, and digital asset management workloads.
