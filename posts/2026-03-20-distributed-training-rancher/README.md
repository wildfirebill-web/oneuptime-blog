# How to Set Up Distributed Training on Rancher - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Distributed-training, PyTorch, TensorFlow, Kubernetes

Description: Comprehensive guide to configuring distributed ML model training across multiple GPU nodes in Rancher.

## Introduction

Distributed training splits model training across multiple GPUs or machines, dramatically reducing training time for large models. Rancher provides the GPU node management and Kubernetes orchestration needed for efficient distributed training.

## Distributed Training Strategies

- **Data Parallelism**: Each GPU processes different data batches; gradients are averaged
- **Model Parallelism**: Model layers are split across GPUs
- **Pipeline Parallelism**: Model stages run on different GPUs in parallel
- **Tensor Parallelism**: Individual operations split across GPUs (LLMs)

## Step 1: Install Training Operators

```bash
# Install Kubeflow Training Operator

kubectl apply -k "github.com/kubeflow/training-operator/manifests/overlays/standalone?ref=v1.7.0"

# Verify CRDs
kubectl get crds | grep -E "tfjobs|pytorchjobs|mpijobs"
```

## Step 2: PyTorch Distributed Training (DDP)

```yaml
# pytorch-distributed-job.yaml
apiVersion: kubeflow.org/v1
kind: PyTorchJob
metadata:
  name: resnet-distributed-training
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
            image: registry.example.com/ml/pytorch-trainer:2.1.0-cuda12
            command:
            - python
            - -m
            - torch.distributed.run
            - --nproc_per_node=4          # 4 GPUs on master
            - --nnodes=2                  # 2 nodes total
            - --node_rank=0
            - --master_addr=$(MASTER_ADDR)
            - --master_port=23456
            - train_ddp.py
            - --epochs=100
            - --batch-size=1024
            resources:
              limits:
                nvidia.com/gpu: "4"
                memory: "128Gi"
                cpu: "16"
          nodeSelector:
            nvidia.com/gpu.present: "true"
          tolerations:
          - key: nvidia.com/gpu
            operator: Exists
            effect: NoSchedule
    
    Worker:
      replicas: 1                         # 1 additional worker
      restartPolicy: OnFailure
      template:
        spec:
          containers:
          - name: pytorch
            image: registry.example.com/ml/pytorch-trainer:2.1.0-cuda12
            command:
            - python
            - -m
            - torch.distributed.run
            - --nproc_per_node=4          # 4 GPUs on worker
            - --nnodes=2
            - --node_rank=1
            - --master_addr=$(MASTER_ADDR)
            - --master_port=23456
            - train_ddp.py
            resources:
              limits:
                nvidia.com/gpu: "4"
          nodeSelector:
            nvidia.com/gpu.present: "true"
          tolerations:
          - key: nvidia.com/gpu
            operator: Exists
            effect: NoSchedule
```

## Step 3: PyTorch Training Script with DDP

```python
# train_ddp.py
import torch
import torch.distributed as dist
import torch.nn as nn
from torch.nn.parallel import DistributedDataParallel as DDP
from torch.utils.data.distributed import DistributedSampler
import torchvision.models as models
import argparse

def setup(rank, world_size):
    dist.init_process_group("nccl")

def train(args):
    # Initialize distributed training
    setup(int(os.environ['RANK']), int(os.environ['WORLD_SIZE']))
    
    local_rank = int(os.environ['LOCAL_RANK'])
    torch.cuda.set_device(local_rank)
    
    # Create model and move to GPU
    model = models.resnet50(pretrained=False)
    model = model.cuda(local_rank)
    model = DDP(model, device_ids=[local_rank])
    
    # Create distributed sampler for data
    train_dataset = load_dataset(args.data_path)
    train_sampler = DistributedSampler(train_dataset)
    train_loader = torch.utils.data.DataLoader(
        train_dataset,
        batch_size=args.batch_size,
        sampler=train_sampler,
        num_workers=4,
        pin_memory=True
    )
    
    optimizer = torch.optim.SGD(model.parameters(), lr=0.1 * dist.get_world_size())
    criterion = nn.CrossEntropyLoss().cuda(local_rank)
    
    for epoch in range(args.epochs):
        train_sampler.set_epoch(epoch)
        
        for batch_idx, (data, target) in enumerate(train_loader):
            data, target = data.cuda(local_rank), target.cuda(local_rank)
            
            optimizer.zero_grad()
            output = model(data)
            loss = criterion(output, target)
            loss.backward()
            optimizer.step()
        
        if local_rank == 0:
            print(f"Epoch {epoch}: loss={loss.item():.4f}")
    
    # Save model only from rank 0
    if dist.get_rank() == 0:
        torch.save(model.module.state_dict(), 'resnet50_distributed.pth')
    
    dist.destroy_process_group()
```

## Step 4: TensorFlow Distributed Training

```yaml
# tensorflow-distributed.yaml
apiVersion: kubeflow.org/v1
kind: TFJob
metadata:
  name: tf-distributed-training
  namespace: ml-training
spec:
  tfReplicaSpecs:
    Chief:
      replicas: 1
      restartPolicy: OnFailure
      template:
        spec:
          containers:
          - name: tensorflow
            image: tensorflow/tensorflow:2.14.0-gpu
            command: [python, train_distributed.py,
                      --epochs, "100",
                      --dataset, "/data/imagenet"]
            resources:
              limits:
                nvidia.com/gpu: "2"
                memory: "32Gi"
          nodeSelector:
            nvidia.com/gpu.present: "true"
          tolerations:
          - key: nvidia.com/gpu
            operator: Exists
            effect: NoSchedule
    
    Worker:
      replicas: 4                      # 4 workers x 2 GPUs = 8 GPUs
      restartPolicy: OnFailure
      template:
        spec:
          containers:
          - name: tensorflow
            image: tensorflow/tensorflow:2.14.0-gpu
            command: [python, train_distributed.py]
            resources:
              limits:
                nvidia.com/gpu: "2"
          nodeSelector:
            nvidia.com/gpu.present: "true"
          tolerations:
          - key: nvidia.com/gpu
            operator: Exists
            effect: NoSchedule
    
    PS:                                # Parameter Server
      replicas: 2
      restartPolicy: OnFailure
      template:
        spec:
          containers:
          - name: tensorflow
            image: tensorflow/tensorflow:2.14.0-gpu
            command: [python, train_distributed.py, --ps]
```

## Step 5: Configure NCCL for Multi-Node

```yaml
# nccl-config.yaml
spec:
  template:
    spec:
      containers:
      - name: trainer
        env:
        - name: NCCL_DEBUG
          value: "WARN"
        - name: NCCL_SOCKET_IFNAME
          value: "eth0"           # Network interface for NCCL
        - name: NCCL_IB_DISABLE
          value: "1"              # Set to 0 if InfiniBand available
        - name: NCCL_P2P_DISABLE
          value: "0"              # Enable P2P for same-node GPUs
        - name: NCCL_TIMEOUT
          value: "1800"           # 30 min timeout
```

## Monitoring Distributed Jobs

```bash
# Watch all training pods
kubectl get pods -n ml-training -w

# Check PyTorchJob status
kubectl get pytorchjobs -n ml-training

# View logs from all workers
kubectl logs -n ml-training \
  -l training.kubeflow.org/job-name=resnet-distributed-training \
  --all-containers --follow

# Monitor GPU utilization across nodes
kubectl exec -n gpu-operator \
  $(kubectl get pods -n gpu-operator -l app=nvidia-dcgm-exporter -o name | head -1) \
  -- nvidia-smi dmon -s pu -d 5
```

## Conclusion

Distributed training on Rancher enables training large models that would be impossible or impractical on a single GPU. By combining the PyTorch DDP or TF MirroredStrategy with Kubeflow Training Operators, you get fault-tolerant distributed training with automatic restart on failure. Proper NCCL configuration and GPU-aware scheduling are key to efficient multi-node training.
