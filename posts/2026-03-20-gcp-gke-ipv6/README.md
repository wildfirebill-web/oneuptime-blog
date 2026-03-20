# How to Configure GKE Clusters with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GCP, IPv6, GKE, Kubernetes, Dual-Stack, Google Cloud, Containers

Description: Create Google Kubernetes Engine clusters with IPv6 support, configure dual-stack pod and service CIDR ranges, and deploy IPv6-aware workloads on GKE.

## Introduction

Google Kubernetes Engine (GKE) supports dual-stack networking, allowing pods and services to receive both IPv4 and IPv6 addresses. GKE dual-stack clusters require a dual-stack subnet and use GKE's native VPC-native networking (Alias IPs). IPv6 in GKE requires GKE version 1.21+ and standard mode clusters - Autopilot clusters do not yet support dual-stack.

## Create a Dual-Stack GKE Cluster

```bash
PROJECT="my-project"
REGION="us-east1"
ZONE="us-east1-b"

# First, create dual-stack subnet for GKE

gcloud compute networks subnets create subnet-gke \
    --network=vpc-main \
    --region="$REGION" \
    --range=10.0.10.0/24 \
    --stack-type=IPV4_IPV6 \
    --ipv6-access-type=INTERNAL \
    --secondary-range pods=10.100.0.0/16,services=10.200.0.0/20 \
    --project="$PROJECT"

# Create dual-stack GKE cluster
gcloud container clusters create gke-dual-stack \
    --project="$PROJECT" \
    --zone="$ZONE" \
    --network=vpc-main \
    --subnetwork=subnet-gke \
    --cluster-secondary-range-name=pods \
    --services-secondary-range-name=services \
    --stack-type=IPV4_IPV6 \
    --enable-ip-alias \
    --release-channel=regular \
    --machine-type=n2-standard-4 \
    --num-nodes=3

# Get cluster credentials
gcloud container clusters get-credentials gke-dual-stack \
    --project="$PROJECT" \
    --zone="$ZONE"

# Verify dual-stack configuration
kubectl get nodes -o wide
kubectl describe node | grep -A5 "PodCIDR"
```

## Terraform GKE Dual-Stack Cluster

```hcl
# gke_ipv6.tf

variable "project_id" {}
variable "region" { default = "us-east1" }

# Dual-stack subnet for GKE
resource "google_compute_subnetwork" "gke" {
  name          = "subnet-gke"
  ip_cidr_range = "10.0.10.0/24"
  region        = var.region
  network       = google_compute_network.main.id
  project       = var.project_id

  stack_type       = "IPV4_IPV6"
  ipv6_access_type = "INTERNAL"

  secondary_ip_range {
    range_name    = "pods"
    ip_cidr_range = "10.100.0.0/16"
  }

  secondary_ip_range {
    range_name    = "services"
    ip_cidr_range = "10.200.0.0/20"
  }
}

# GKE dual-stack cluster
resource "google_container_cluster" "main" {
  name     = "gke-dual-stack"
  location = "${var.region}-b"
  project  = var.project_id

  network    = google_compute_network.main.name
  subnetwork = google_compute_subnetwork.gke.name

  # Enable VPC-native (required for IPv6)
  ip_allocation_policy {
    cluster_secondary_range_name  = "pods"
    services_secondary_range_name = "services"
  }

  # Enable dual-stack
  stack_type = "IPV4_IPV6"

  # Remove default node pool
  remove_default_node_pool = true
  initial_node_count       = 1

  release_channel {
    channel = "REGULAR"
  }
}

# Separate node pool
resource "google_container_node_pool" "main" {
  name     = "main-nodes"
  cluster  = google_container_cluster.main.id
  location = "${var.region}-b"
  project  = var.project_id

  node_count = 3

  node_config {
    machine_type = "n2-standard-4"

    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]
  }
}
```

## Deploy IPv6-Aware Workloads

```yaml
# dual-stack-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  ipFamilyPolicy: PreferDualStack
  ipFamilies:
    - IPv4
    - IPv6
  type: ClusterIP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: nginx:latest
          ports:
            - containerPort: 8080
```

## Verify IPv6 in GKE

```bash
# Check pod IPv6 addresses
kubectl get pods -o wide
kubectl get pod web-xxx -o jsonpath='{.status.podIPs}'
# Output: [{"ip":"10.100.x.x"},{"ip":"fd20::x"}]

# Check service ClusterIPs
kubectl get service web-service -o jsonpath='{.spec.clusterIPs}'

# Test IPv6 connectivity between pods
kubectl exec -it test-pod -- ping6 fd20::x

# Check node dual-stack CIDRs
kubectl get node gke-node-xxx -o jsonpath='{.spec.podCIDRs}'
```

## Conclusion

GKE dual-stack clusters require `--stack-type=IPV4_IPV6` on both the subnet and the cluster, with VPC-native networking enabled. Use `ipFamilyPolicy: PreferDualStack` on Services to get both IPv4 and IPv6 ClusterIPs. Pods in dual-stack clusters automatically receive both address families. Verify with `kubectl get pods -o wide` and check `podIPs` for multiple addresses. GKE handles all the underlying IPv6 routing within the VPC automatically.
