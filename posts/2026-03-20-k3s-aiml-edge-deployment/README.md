# How to Deploy AI/ML Models at the Edge with K3s

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: k3s, AI/ML, Edge Computing, Kubernetes, NVIDIA, TensorFlow, Model Serving

Description: Learn how to deploy AI and machine learning model inference workloads at the edge using K3s, including GPU configuration, model serving with Triton or TorchServe, and edge-optimized deployment...

---

Running AI inference at the edge reduces latency, preserves privacy, and works offline. K3s's lightweight footprint makes it ideal for edge AI deployments on NVIDIA Jetson, edge servers, and industrial hardware.

---

## Step 1: Install K3s with GPU Support

```bash
# Install NVIDIA drivers and container toolkit on the edge node

distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt-get update && sudo apt-get install -y nvidia-container-toolkit

# Configure containerd for NVIDIA
sudo nvidia-ctk runtime configure --runtime=containerd
sudo systemctl restart containerd

# Install K3s
curl -sfL https://get.k3s.io | sh -
```

---

## Step 2: Install the NVIDIA Device Plugin

```bash
# Deploy NVIDIA device plugin as a DaemonSet
kubectl apply -f \
  https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.14.5/nvidia-device-plugin.yml

# Verify GPU is visible to Kubernetes
kubectl get nodes -o json | jq '.items[].status.capacity | select(."nvidia.com/gpu")'
```

---

## Step 3: Deploy an AI Inference Server

Deploy NVIDIA Triton Inference Server to serve ML models:

```yaml
# triton-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: triton-server
  namespace: ai-inference
spec:
  replicas: 1
  selector:
    matchLabels:
      app: triton-server
  template:
    metadata:
      labels:
        app: triton-server
    spec:
      containers:
        - name: triton
          image: nvcr.io/nvidia/tritonserver:24.01-py3
          args:
            # Path to the model repository (mounted from a PVC or hostPath)
            - --model-repository=/models
            - --strict-model-config=false
          ports:
            - containerPort: 8000  # HTTP
            - containerPort: 8001  # gRPC
            - containerPort: 8002  # metrics
          resources:
            limits:
              # Request 1 GPU for inference
              nvidia.com/gpu: "1"
              memory: 8Gi
            requests:
              nvidia.com/gpu: "1"
              memory: 4Gi
          volumeMounts:
            - name: model-store
              mountPath: /models
      volumes:
        - name: model-store
          hostPath:
            path: /data/models
            type: Directory
```

---

## Step 4: Load a Model

Copy your ONNX, TensorFlow, or TorchScript model to the model repository:

```bash
# Model repository structure
/data/models/
  my-model/
    1/                    # model version
      model.onnx          # ONNX model file
    config.pbtxt          # Triton model configuration

# config.pbtxt example
cat > /data/models/my-model/config.pbtxt <<EOF
name: "my-model"
platform: "onnxruntime_onnx"
max_batch_size: 8
input [{ name: "input", data_type: TYPE_FP32, dims: [3, 224, 224] }]
output [{ name: "output", data_type: TYPE_FP32, dims: [1000] }]
EOF
```

---

## Step 5: Run Inference

```bash
# Port-forward for local testing
kubectl port-forward svc/triton-server 8000:8000 -n ai-inference

# Send a test inference request
curl -s -X POST http://localhost:8000/v2/models/my-model/infer \
  -H "Content-Type: application/json" \
  -d '{"inputs":[{"name":"input","shape":[1,3,224,224],"datatype":"FP32","data":[...]}]}'
```

---

## Best Practices

- Cache models in a local PVC backed by Longhorn - avoid pulling large models over a slow edge connection on every restart.
- Use **model versioning** in Triton to roll out new model versions without service disruption.
- Set **CPU fallback** for non-GPU edge nodes using TensorRT Lite or ONNX Runtime CPU providers.
- Monitor GPU utilization and temperature using the DCGM exporter and alert when utilization drops unexpectedly.
