# How to Use the Rancher CLI

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, CLI, Cluster Management

Description: A practical guide to using the Rancher CLI for managing clusters, projects, apps, and resources from the command line.

The Rancher CLI gives you command-line access to your Rancher server, letting you manage clusters, projects, namespaces, and workloads without opening a browser. This guide covers the most common operations you will perform with the Rancher CLI.

## Logging In

Before you can use any CLI commands, you need to authenticate with your Rancher server:

```bash
rancher login https://rancher.example.com --token token-xxxxx:yyyyyyyyyyyyyyyy
```

If your Rancher instance uses a self-signed certificate, add the `--skip-verify` flag:

```bash
rancher login https://rancher.example.com --token token-xxxxx:yyyyyyyyyyyyyyyy --skip-verify
```

After a successful login, the CLI stores your credentials in `~/.rancher/cli2.json`.

## Switching Contexts

Rancher CLI uses contexts to determine which cluster and project you are working with.

### List Available Contexts

```bash
rancher context switch
```

This displays an interactive menu of all available clusters and projects. Select one by entering its number.

### View Current Context

```bash
rancher context current
```

### Switch to a Specific Project

```bash
rancher context switch --project c-m-abc12345:p-xyz789
```

## Managing Clusters

### List All Clusters

```bash
rancher clusters ls
```

This outputs a table with cluster names, IDs, states, and node counts:

```
CURRENT   ID              NAME          STATE    NODES   PROVIDER
*         c-m-abc12345    production    active   5       rke2
          c-m-def67890    staging       active   3       rke2
          c-m-ghi11111    development   active   2       k3s
```

### Inspect a Cluster

```bash
rancher clusters inspect production
```

### Get Cluster Kubeconfig

```bash
rancher clusters kubeconfig production
```

This outputs the kubeconfig YAML, which you can redirect to a file:

```bash
rancher clusters kubeconfig production > ~/.kube/production.yaml
```

## Managing Projects

### List Projects

```bash
rancher projects ls
```

### Create a New Project

```bash
rancher projects create --cluster production --description "Team Alpha services" team-alpha
```

### Switch to a Project

```bash
rancher project switch team-alpha
```

## Managing Namespaces

### List Namespaces

To list namespaces in your current project:

```bash
rancher namespaces ls
```

### Create a Namespace

```bash
rancher namespaces create my-namespace
```

### Move a Namespace to a Different Project

```bash
rancher namespaces move my-namespace p-newproject
```

## Working with Applications (Catalog Apps)

### List App Catalogs

```bash
rancher catalog ls
```

### Search for Apps

```bash
rancher apps search nginx
```

### Install an App

```bash
rancher apps install cattle-global-data:library-nginx my-nginx \
  --namespace my-namespace \
  --set replicaCount=3
```

### List Installed Apps

```bash
rancher apps ls
```

### Upgrade an App

```bash
rancher apps upgrade my-nginx 1.2.0
```

### Delete an App

```bash
rancher apps delete my-nginx
```

## Running kubectl Commands

The Rancher CLI can proxy kubectl commands to your managed clusters:

```bash
rancher kubectl get pods --all-namespaces
```

```bash
rancher kubectl get nodes -o wide
```

```bash
rancher kubectl apply -f deployment.yaml
```

This routes all kubectl commands through the Rancher server, so you do not need to configure kubeconfig files separately.

## Managing Tokens

### List Your API Tokens

```bash
rancher tokens ls
```

### Create a New Token

```bash
rancher tokens create --description "Automation token" --ttl 86400000
```

### Delete a Token

```bash
rancher tokens delete token-abc12
```

## Working with Multi-Cluster Apps

### Deploy a Multi-Cluster App

```bash
rancher multiclusterapps install \
  --target c-m-abc12345:p-proj1 \
  --target c-m-def67890:p-proj2 \
  cattle-global-data:library-nginx \
  global-nginx
```

### List Multi-Cluster Apps

```bash
rancher multiclusterapps ls
```

## Using Output Formats

The CLI supports different output formats for scripting:

### JSON Output

```bash
rancher clusters ls --format json
```

### Custom Format

```bash
rancher clusters ls --format '{{.Cluster.Name}} {{.Cluster.State}}'
```

### Quiet Mode (IDs only)

```bash
rancher clusters ls -q
```

## Practical Scripting Examples

### Deploy to All Clusters

```bash
#!/bin/bash

for cluster_id in $(rancher clusters ls -q); do
  echo "Deploying to cluster: ${cluster_id}"
  rancher context switch --cluster "${cluster_id}"
  rancher kubectl apply -f deployment.yaml
done
```

### Check Node Status Across Clusters

```bash
#!/bin/bash

for cluster in $(rancher clusters ls --format '{{.Cluster.Name}}'); do
  echo "=== ${cluster} ==="
  rancher kubectl --cluster "${cluster}" get nodes -o wide
  echo ""
done
```

### Export All Cluster Kubeconfigs

```bash
#!/bin/bash

mkdir -p ~/.kube/rancher

for cluster in $(rancher clusters ls --format '{{.Cluster.Name}}'); do
  rancher clusters kubeconfig "${cluster}" > ~/.kube/rancher/${cluster}.yaml
  echo "Exported kubeconfig for ${cluster}"
done
```

## Helpful CLI Tips

### Enable Shell Completion

For bash:

```bash
rancher completion bash > /etc/bash_completion.d/rancher
source /etc/bash_completion.d/rancher
```

For zsh:

```bash
rancher completion zsh > "${fpath[1]}/_rancher"
source ~/.zshrc
```

### Use Environment Variables

You can set default values with environment variables:

```bash
export RANCHER_URL="https://rancher.example.com"
export RANCHER_TOKEN="token-xxxxx:yyyyyyyyyyyyyyyy"
export RANCHER_SKIP_VERIFY=true
```

### Check CLI Version

```bash
rancher --version
```

## Summary

The Rancher CLI provides a fast and scriptable way to manage your entire Rancher environment. From switching contexts and managing clusters to deploying applications and running kubectl commands, everything you can do in the UI is available from the terminal. Combine these commands in shell scripts to build powerful automation workflows.
