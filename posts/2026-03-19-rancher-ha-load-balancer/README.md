# How to Set Up Rancher HA Behind a Load Balancer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, High Availability, Load Balancer, Installation

Description: Configure Rancher high availability with an external load balancer for production-grade reliability and traffic distribution.

A load balancer in front of your Rancher HA cluster provides a single entry point, health checking, and automatic failover. Instead of DNS round-robin, traffic is routed through a dedicated load balancer that only sends requests to healthy nodes. This guide covers setting up a Rancher HA cluster with both a software-based load balancer (Nginx) and cloud load balancer options.

## Prerequisites

- Three servers for the Rancher cluster (Ubuntu 22.04, 4 GB RAM, 2 CPUs each)
- One additional server for the Nginx load balancer (or a cloud load balancer)
- Network connectivity between all servers
- A domain name
- SSH access to all servers

## Architecture

The load balancer sits in front of three K3s server nodes running Rancher:

```plaintext
                    +--> Node 1 (192.168.1.101)
Client --> LB (192.168.1.100) +--> Node 2 (192.168.1.102)
                    +--> Node 3 (192.168.1.103)
```

## Step 1: Set Up the K3s HA Cluster

On the first node, initialize K3s with embedded etcd:

```bash
ssh ubuntu@192.168.1.101

curl -sfL https://get.k3s.io | sh -s - server \
  --cluster-init \
  --write-kubeconfig-mode 644 \
  --tls-san rancher.example.com \
  --tls-san 192.168.1.100

TOKEN=$(sudo cat /var/lib/rancher/k3s/server/node-token)
echo "Join token: $TOKEN"
```

Join the second and third nodes:

```bash
# On node 2

ssh ubuntu@192.168.1.102
curl -sfL https://get.k3s.io | sh -s - server \
  --server https://192.168.1.101:6443 \
  --token <node-token> \
  --write-kubeconfig-mode 644 \
  --tls-san rancher.example.com \
  --tls-san 192.168.1.100

# On node 3
ssh ubuntu@192.168.1.103
curl -sfL https://get.k3s.io | sh -s - server \
  --server https://192.168.1.101:6443 \
  --token <node-token> \
  --write-kubeconfig-mode 644 \
  --tls-san rancher.example.com \
  --tls-san 192.168.1.100
```

Verify the cluster on any node:

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
kubectl get nodes
```

## Step 2: Install Rancher on the Cluster

On the first node:

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Install cert-manager
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true

# Wait for cert-manager
kubectl -n cert-manager wait --for=condition=ready pod -l app=cert-manager --timeout=120s

# Install Rancher
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update
kubectl create namespace cattle-system
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=rancher.example.com \
  --set bootstrapPassword=admin \
  --set replicas=3
```

## Step 3: Set Up the Nginx Load Balancer

SSH into the load balancer server:

```bash
ssh ubuntu@192.168.1.100
```

Install Nginx:

```bash
sudo apt update
sudo apt install -y nginx
```

Create the load balancer configuration:

```bash
sudo tee /etc/nginx/nginx.conf > /dev/null <<'EOF'
worker_processes auto;

events {
    worker_connections 1024;
}

stream {
    upstream rancher_https {
        least_conn;
        server 192.168.1.101:443 max_fails=3 fail_timeout=10s;
        server 192.168.1.102:443 max_fails=3 fail_timeout=10s;
        server 192.168.1.103:443 max_fails=3 fail_timeout=10s;
    }

    upstream rancher_http {
        least_conn;
        server 192.168.1.101:80 max_fails=3 fail_timeout=10s;
        server 192.168.1.102:80 max_fails=3 fail_timeout=10s;
        server 192.168.1.103:80 max_fails=3 fail_timeout=10s;
    }

    server {
        listen 443;
        proxy_pass rancher_https;
        proxy_timeout 30s;
        proxy_connect_timeout 5s;
    }

    server {
        listen 80;
        proxy_pass rancher_http;
        proxy_timeout 30s;
        proxy_connect_timeout 5s;
    }
}
EOF
```

Test and restart Nginx:

```bash
sudo nginx -t
sudo systemctl restart nginx
sudo systemctl enable nginx
```

## Step 4: Using a Cloud Load Balancer (Alternative)

If you are running on a cloud provider, you can use their managed load balancer instead of Nginx.

### AWS Application Load Balancer

```bash
# Create a target group
aws elbv2 create-target-group \
  --name rancher-tg \
  --protocol HTTPS \
  --port 443 \
  --vpc-id vpc-xxxxxxxx \
  --target-type instance \
  --health-check-protocol HTTPS \
  --health-check-path /healthz

# Register targets
aws elbv2 register-targets \
  --target-group-arn <target-group-arn> \
  --targets Id=i-node1 Id=i-node2 Id=i-node3

# Create the load balancer
aws elbv2 create-load-balancer \
  --name rancher-lb \
  --subnets subnet-xxx subnet-yyy \
  --security-groups sg-xxxxxxxx \
  --scheme internet-facing
```

### DigitalOcean Load Balancer

```bash
doctl compute load-balancer create \
  --name rancher-lb \
  --region nyc3 \
  --forwarding-rules "entry_protocol:https,entry_port:443,target_protocol:https,target_port:443,tls_passthrough:true" \
  --health-check "protocol:https,port:443,path:/healthz,check_interval_seconds:10,response_timeout_seconds:5,healthy_threshold:3,unhealthy_threshold:3" \
  --droplet-ids node1-id,node2-id,node3-id
```

## Step 5: Configure DNS

Create a DNS A record pointing your domain to the load balancer IP:

```plaintext
rancher.example.com  A  192.168.1.100
```

For cloud load balancers, use a CNAME record pointing to the load balancer DNS name.

## Step 6: Verify the Setup

Access `https://rancher.example.com` in your browser. The load balancer routes your request to one of the healthy backend nodes.

Test failover by stopping K3s on one node:

```bash
ssh ubuntu@192.168.1.101
sudo systemctl stop k3s
```

The load balancer detects the failure and routes traffic to the remaining healthy nodes. Rancher continues to operate normally.

Restart the node:

```bash
sudo systemctl start k3s
```

## Health Check Configuration

The load balancer uses the `/healthz` endpoint on each node to determine health. This endpoint returns a 200 status when the node is healthy. Configure your health checks to poll this endpoint every 10 seconds with a timeout of 5 seconds.

## SSL Termination Options

You have two options for SSL handling:

1. **TLS Passthrough** (recommended): The load balancer passes encrypted traffic directly to the backend nodes. This is the simplest configuration and lets Rancher handle its own certificates.

2. **SSL Termination at the Load Balancer**: The load balancer decrypts traffic and forwards plain HTTP to the backend nodes. This requires installing your SSL certificate on the load balancer.

## Summary

You have set up Rancher in a high-availability configuration behind a load balancer. This architecture provides a single entry point, automatic failover, and health-based routing. Whether you use Nginx or a cloud-managed load balancer, this setup ensures your Rancher installation remains accessible even when individual nodes experience issues.
