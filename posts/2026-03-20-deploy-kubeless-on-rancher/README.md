# How to Deploy Kubeless on Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubeless, Serverless, Functions, Kubernetes, Event-Driven

Description: Deploy Kubeless serverless framework on Rancher to run functions as Kubernetes native resources with event triggers and HTTP endpoints.

## Introduction

Kubeless is a Kubernetes-native serverless framework that uses Custom Resource Definitions (CRDs) to represent functions. Functions are deployed as Kubernetes resources, making them manageable via standard kubectl commands. Kubeless supports Python, Node.js, Ruby, PHP, and Go runtimes.

## Step 1: Install Kubeless

```bash
# Create the Kubeless namespace

kubectl create namespace kubeless

# Install Kubeless
export RELEASE=$(curl -s https://api.github.com/repos/vmware-archive/kubeless/releases/latest | grep tag_name | cut -d '"' -f 4)
kubectl create -f https://github.com/vmware-archive/kubeless/releases/download/$RELEASE/kubeless-non-rbac-$RELEASE.yaml

# Verify installation
kubectl get pods -n kubeless
kubectl get customresourcedefinition
```

## Step 2: Install the Kubeless CLI

```bash
# macOS
brew install kubeless

# Linux
export RELEASE=$(curl -s https://api.github.com/repos/vmware-archive/kubeless/releases/latest | grep tag_name | cut -d '"' -f 4)
curl -OL https://github.com/vmware-archive/kubeless/releases/download/$RELEASE/kubeless_linux-amd64.zip
unzip kubeless_linux-amd64.zip
sudo mv kubeless /usr/local/bin
```

## Step 3: Deploy Your First Function

```python
# hello.py
def hello(event, context):
    print(event)
    return event['data']
```

```bash
# Deploy the Python function
kubeless function deploy hello \
  --runtime python3.8 \
  --from-file hello.py \
  --handler hello.hello

# Verify the function pod is running
kubectl get function hello
kubectl get pod -l function=hello
```

## Step 4: Call the Function

```bash
# Using kubeless CLI
kubeless function call hello --data '{"greeting": "Hello from Rancher"}'

# Using kubectl proxy
kubectl proxy &
curl -L --data '{"greeting": "Hello"}' \
  --header "Content-Type:application/json" \
  localhost:8001/api/v1/namespaces/default/services/hello:8080/proxy/

# Using Ingress (if configured)
curl https://hello.example.com
```

## Step 5: Create an HTTP Trigger

```bash
# Create an HTTP trigger for external access
kubeless trigger http create hello \
  --function-name hello \
  --hostname hello.example.com \
  --path /hello

# Verify the trigger
kubeless trigger http list
```

## Step 6: Create a Kafka Trigger

```bash
# Create a Kafka trigger for event-driven functions
kubeless trigger kafka create hello-kafka-trigger \
  --function-selector function=hello \
  --trigger-topic orders

# Function will be invoked for each message in the 'orders' topic
```

## Step 7: Update a Function

```bash
# Update function code
kubeless function update hello --from-file hello-v2.py

# Check deployment status
kubectl rollout status deployment/hello
```

## Conclusion

Kubeless on Rancher provides Kubernetes-native serverless function deployment with familiar kubectl-based management. Functions are first-class Kubernetes resources with full RBAC, namespace isolation, and resource limit support. The Kafka and Cron trigger types make Kubeless suitable for event-driven processing and scheduled tasks.
