# How to Set Up Distributed Training on Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Distributed Training, Kubernetes, PyTorch, TensorFlow, GPU, Training Operator

Description: Configure distributed ML model training on Rancher using the Kubeflow Training Operator with PyTorchJob and TFJob for multi-GPU and multi-node training.

## Introduction

Training large ML models requires distributing work across multiple GPUs and nodes. The Kubeflow Training Operator provides Kubernetes-native distributed training with PyTorchJob and TFJob custom resources, enabling both data-parallel and model-parallel training strategies.

## Step 1: Install the Training Operator

```bash
kubectl apply -k "github.com/kubeflow/training-operator/manifests/overlays/standalone?ref=v1.7.0"

# Verify installation
kubectl get pods -n kubeflow
kubectl get crd | grep kubeflow
```

## Step 2: Configure GPU Nodes

```bash
# Install NVIDIA GPU Operator (handles driver and plugin installation)
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm install gpu-operator nvidia/gpu-operator \
  --namespace gpu-operator \
  --create-namespace

# Verify GPUs are available
kubectl get nodes -o json | jq '.items[].status.capacity | select(."nvidia.com/gpu" != null)'
```

## Step 3: Launch a PyTorchJob (Data Parallel Training)

```yaml
# pytorch-training-job.yaml
apiVersion: "kubeflow.org/v1"
kind: PyTorchJob
metadata:
  name: bert-fine-tuning
  namespace: ml-training
spec:
  pytorchReplicaSpecs:
    Master:
      replicas: 1
      restartPolicy: OnFailure
      template:
        spec:
          containers:
            - name: pytorch
              image: myregistry/bert-trainer:latest
              command:
                - python
                - -m
                - torch.distributed.launch
                - --nproc_per_node=2
                - train.py
                - --epochs=10
                - --batch-size=32
              resources:
                requests:
                  nvidia.com/gpu: "2"    # 2 GPUs per master pod
              env:
                - name: MLFLOW_TRACKING_URI
                  value: https://mlflow.example.com
    Worker:
      replicas: 3    # 3 worker nodes
      restartPolicy: OnFailure
      template:
        spec:
          containers:
            - name: pytorch
              image: myregistry/bert-trainer:latest
              command:
                - python
                - -m
                - torch.distributed.launch
                - --nproc_per_node=2
                - train.py
              resources:
                requests:
                  nvidia.com/gpu: "2"    # 2 GPUs per worker
```

## Step 4: Distributed Training Code (PyTorch)

```python
# train.py
import torch
import torch.distributed as dist
from torch.nn.parallel import DistributedDataParallel as DDP
import os

def setup():
    """Initialize distributed training."""
    dist.init_process_group("nccl")    # NCCL for GPU communication
    local_rank = int(os.environ["LOCAL_RANK"])
    torch.cuda.set_device(local_rank)

def main():
    setup()

    # Each process handles a slice of the data
    rank = dist.get_rank()
    world_size = dist.get_world_size()

    # Wrap model in DDP
    model = MyModel().cuda()
    model = DDP(model, device_ids=[torch.cuda.current_device()])

    # Distributed sampler splits data across processes
    from torch.utils.data.distributed import DistributedSampler
    sampler = DistributedSampler(dataset, num_replicas=world_size, rank=rank)
    dataloader = DataLoader(dataset, sampler=sampler, batch_size=32)

    # Training loop
    for epoch in range(10):
        sampler.set_epoch(epoch)    # Required for proper shuffling
        for batch in dataloader:
            # Forward pass, backward pass, optimizer step...
            pass

    if rank == 0:    # Only save model from the master process
        torch.save(model.module.state_dict(), "model.pt")

if __name__ == "__main__":
    main()
```

## Step 5: Monitor Training

```bash
# Watch training job status
kubectl get pytorchjobs -n ml-training -w

# View master pod logs
kubectl logs -n ml-training bert-fine-tuning-master-0 -f

# Check GPU utilization
kubectl exec -it bert-fine-tuning-master-0 -n ml-training -- nvidia-smi
```

## Conclusion

Distributed training on Rancher with the Training Operator scales model training from single-GPU to multi-node GPU clusters. The `PyTorchJob` handles process group initialization, inter-node communication setup, and fault tolerance automatically. NCCL backend provides optimal GPU-to-GPU communication for gradient synchronization.
