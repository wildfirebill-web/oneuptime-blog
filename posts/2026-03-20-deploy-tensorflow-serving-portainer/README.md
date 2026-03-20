# How to Deploy TensorFlow Serving via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TensorFlow, TensorFlow Serving, Portainer, Docker, ML Inference, REST API, gRPC

Description: Deploy TensorFlow Serving using Portainer to serve trained TensorFlow and Keras models as production REST and gRPC inference endpoints.

---

TensorFlow Serving is the production-grade inference server for TensorFlow models. It handles model versioning, hot-swapping models without downtime, and serves predictions via REST or gRPC APIs. Portainer makes it easy to deploy and manage TF Serving containers.

## Step 1: Prepare Your Saved Model

Export your trained model in the TensorFlow SavedModel format:

```python
# export_model.py - run this in your training environment

import tensorflow as tf
import numpy as np

# Build and train a simple model (example)
model = tf.keras.Sequential([
    tf.keras.layers.Dense(128, activation="relu", input_shape=(784,)),
    tf.keras.layers.Dense(10, activation="softmax")
])
model.compile(optimizer="adam", loss="sparse_categorical_crossentropy")

# Save in SavedModel format - required by TF Serving
# Directory structure must follow: /models/<model-name>/<version>/
model.save("/opt/tf-models/mnist-classifier/1")
print("Model exported to /opt/tf-models/mnist-classifier/1")
```

## Step 2: Deploy TF Serving via Portainer Stack

```yaml
# tensorflow-serving-stack.yml
version: "3.8"

services:
  tf-serving:
    image: tensorflow/serving:2.15.0
    command: >
      --model_config_file=/models/models.config
      --model_config_file_poll_wait_seconds=60
    volumes:
      # Bind-mount the models directory from the host
      - /opt/tf-models:/models:ro
    ports:
      - "8500:8500"    # gRPC inference endpoint
      - "8501:8501"    # REST inference endpoint
    restart: unless-stopped
    networks:
      - ml-serving

  # GPU-accelerated TF Serving (alternative service definition)
  # tf-serving-gpu:
  #   image: tensorflow/serving:2.15.0-gpu
  #   deploy:
  #     resources:
  #       reservations:
  #         devices:
  #           - driver: nvidia
  #             count: 1
  #             capabilities: [gpu]

networks:
  ml-serving:
    driver: bridge
```

## Step 3: Create the Model Configuration File

Create `/opt/tf-models/models.config` on the host:

```protobuf
# models.config - defines which models TF Serving loads
# Supports multiple models and version policies

model_config_list {
  config {
    name: "mnist-classifier"
    base_path: "/models/mnist-classifier"
    model_platform: "tensorflow"
    model_version_policy {
      # Serve all versions (for A/B testing)
      all {}
    }
  }
  config {
    name: "image-classifier"
    base_path: "/models/image-classifier"
    model_platform: "tensorflow"
    model_version_policy {
      # Serve only the latest version
      latest { num_versions: 1 }
    }
  }
}
```

## Step 4: Make Predictions via REST API

```python
# predict_rest.py - client code for the REST API
import requests
import numpy as np
import json

# Prepare input data (batch of 2 MNIST-style images)
input_data = np.random.rand(2, 784).tolist()

# Call the REST prediction endpoint
response = requests.post(
    "http://localhost:8501/v1/models/mnist-classifier:predict",
    json={"instances": input_data}
)

# Parse predictions
result = response.json()
predictions = result["predictions"]
print(f"Predictions: {np.argmax(predictions, axis=1)}")
```

## Step 5: Make Predictions via gRPC

```python
# predict_grpc.py - more efficient gRPC client
import grpc
import numpy as np
from tensorflow_serving.apis import predict_pb2, prediction_service_pb2_grpc
import tensorflow as tf

# Connect to TF Serving gRPC endpoint
channel = grpc.insecure_channel("localhost:8500")
stub = prediction_service_pb2_grpc.PredictionServiceStub(channel)

# Build request
request = predict_pb2.PredictRequest()
request.model_spec.name = "mnist-classifier"
request.model_spec.signature_name = "serving_default"

# Add input tensor
input_array = np.random.rand(1, 784).astype(np.float32)
request.inputs["dense_input"].CopyFrom(
    tf.make_tensor_proto(input_array)
)

# Get prediction (gRPC is ~2x faster than REST for high throughput)
response = stub.Predict(request, timeout=5.0)
print(response.outputs)
```

## Step 6: Zero-Downtime Model Updates

TF Serving polls the model directory every 60 seconds (as configured above). To update a model:

1. Copy the new model to `/opt/tf-models/mnist-classifier/2` (increment the version number)
2. TF Serving automatically loads the new version and serves it
3. Old versions are unloaded after the new version is healthy
4. No container restart needed - update is seamless

## Monitoring

TF Serving exposes Prometheus metrics at `/monitoring/prometheus/metrics` on port 8501. Add it to your scrape config to track request rates, latencies, and model loading status.

## Summary

TensorFlow Serving deployed via Portainer provides a production-ready ML inference service with automatic model versioning, REST and gRPC endpoints, and zero-downtime updates. Portainer's log viewer and environment variable management make it easy to operate TF Serving in production.
