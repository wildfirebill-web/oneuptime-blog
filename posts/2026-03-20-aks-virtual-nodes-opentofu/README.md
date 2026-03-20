# How to Set Up AKS with Virtual Nodes Using OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Azure, AKS, Virtual Nodes, Azure Container Instances, Serverless, Infrastructure as Code

Description: Learn how to configure AKS Virtual Nodes with OpenTofu to burst workloads to Azure Container Instances (ACI) for serverless scaling without managing VM nodes.

## Introduction

AKS Virtual Nodes use the Virtual Kubelet to schedule pods to Azure Container Instances (ACI) as if they were running on regular nodes. This enables instant scaling (no VM provisioning time) and true serverless billing (pay per second of pod execution). Virtual Nodes are ideal for burst workloads, batch jobs, and event-driven scaling where you need fast scale-out without pre-provisioning nodes. They require Azure CNI networking and a dedicated ACI subnet.

## Prerequisites

- OpenTofu v1.6+
- Azure credentials with AKS and ACI permissions
- A VNet with dedicated subnets for AKS nodes and ACI

## Step 1: Create AKS with Virtual Nodes

```hcl
# Dedicated subnet for ACI (Virtual Nodes) - must be delegated to ACI

resource "azurerm_subnet" "aci" {
  name                 = "aci-subnet"
  resource_group_name  = var.resource_group_name
  virtual_network_name = var.vnet_name
  address_prefixes     = ["10.2.0.0/24"]

  delegation {
    name = "aci-delegation"

    service_delegation {
      name    = "Microsoft.ContainerInstance/containerGroups"
      actions = ["Microsoft.Network/virtualNetworks/subnets/join/action"]
    }
  }
}

resource "azurerm_kubernetes_cluster" "virtual_nodes" {
  name                = "${var.project_name}-aks"
  location            = var.location
  resource_group_name = var.resource_group_name
  dns_prefix          = var.project_name
  kubernetes_version  = "1.28"

  default_node_pool {
    name                = "system"
    vm_size             = "Standard_D4s_v3"
    node_count          = 2
    min_count           = 2
    max_count           = 10
    enable_auto_scaling = true
    vnet_subnet_id      = var.aks_subnet_id
  }

  identity {
    type = "SystemAssigned"
  }

  network_profile {
    network_plugin    = "azure"  # Required for Virtual Nodes
    load_balancer_sku = "standard"
    service_cidr      = "10.200.0.0/16"
    dns_service_ip    = "10.200.0.10"
  }

  # Enable Virtual Nodes add-on
  aci_connector_linux {
    subnet_name = azurerm_subnet.aci.name
  }

  tags = {
    Name = "${var.project_name}-aks-virtual-nodes"
  }
}
```

## Step 2: Deploy Pods to Virtual Nodes

```yaml
# kubernetes/burst-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-burst
  namespace: production
spec:
  replicas: 5
  selector:
    matchLabels:
      app: api-burst
  template:
    metadata:
      labels:
        app: api-burst
    spec:
      # Schedule on virtual nodes (ACI)
      nodeSelector:
        kubernetes.io/role: agent
        beta.kubernetes.io/os: linux
        type: virtual-kubelet

      tolerations:
        - key: virtual-kubelet.io/provider
          operator: Equal
          value: azure
          effect: NoSchedule

      containers:
        - name: api
          image: myapp:latest
          resources:
            requests:
              cpu: "1"
              memory: "1Gi"
            limits:
              cpu: "2"
              memory: "2Gi"
          ports:
            - containerPort: 8080
```

## Step 3: KEDA with Virtual Nodes for Event-Driven Scaling

```hcl
# KEDA scales pods to virtual nodes based on events (queue depth, etc.)
# Install KEDA via Helm after cluster creation
resource "helm_release" "keda" {
  name             = "keda"
  repository       = "https://kedacore.github.io/charts"
  chart            = "keda"
  namespace        = "keda"
  create_namespace = true

  depends_on = [azurerm_kubernetes_cluster.virtual_nodes]
}
```

```yaml
# kubernetes/keda-scaledobject.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: queue-processor-scaledobject
  namespace: production
spec:
  scaleTargetRef:
    name: queue-processor
  minReplicaCount: 0   # Scale to zero when queue is empty
  maxReplicaCount: 100 # Burst to ACI
  triggers:
    - type: azure-servicebus
      metadata:
        queueName: work-queue
        namespace: myservicebus
        messageCount: "5"  # 1 replica per 5 messages
```

## Step 4: HPA with Virtual Node Bursting

```yaml
# kubernetes/hpa.yaml - scale normal pods first, burst to ACI
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 3
  maxReplicas: 50  # Some pods go to virtual nodes when regular nodes are full
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

## Step 5: Deploy

```bash
tofu init
tofu plan
tofu apply

# Get credentials
az aks get-credentials \
  --resource-group <rg> \
  --name <cluster-name>

# Check virtual node status
kubectl get nodes
# Look for node named "virtual-node-aci-linux"

# Deploy to virtual nodes
kubectl apply -f burst-deployment.yaml

# Check pods on virtual node
kubectl get pods -o wide | grep virtual-node
```

## Conclusion

Virtual Nodes are best suited for stateless, batch, or burst workloads-ACI doesn't support persistent volumes, DaemonSets, or many Kubernetes primitives. Pods on Virtual Nodes start in 10-15 seconds versus 3-5 minutes for new VM nodes, making them ideal for queue-based workloads with KEDA. ACI pricing is per-second based on vCPU and memory, which is cost-effective for short-lived workloads but more expensive than VMs for continuously running workloads. Use pod priority and affinity to schedule critical production pods on VM nodes first, only spilling to Virtual Nodes during bursts.
