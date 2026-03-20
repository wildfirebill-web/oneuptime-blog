# How to Set Up AKS with Azure Policy Using OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Azure, AKS, Azure Policy, OPA Gatekeeper, Governance, Infrastructure as Code

Description: Learn how to configure Azure Policy for AKS with OpenTofu to enforce Kubernetes security standards, resource limits, and compliance requirements using OPA Gatekeeper.

## Introduction

Azure Policy for AKS uses OPA Gatekeeper as an admission controller to enforce policies on Kubernetes resources at creation and update time. Policies can deny non-compliant resources (Deny effect), audit existing resources (Audit effect), or automatically remediate them. Built-in policy initiatives include Kubernetes cluster pod security baseline standards and restricted standards, covering common security requirements like privileged container restrictions, host path mounts, and required resource limits.

## Prerequisites

- OpenTofu v1.6+
- Azure credentials with AKS and Policy permissions
- An existing AKS cluster or create one with the Policy add-on

## Step 1: Enable Azure Policy Add-on on AKS

```hcl
resource "azurerm_kubernetes_cluster" "policy_enabled" {
  name                = "${var.project_name}-aks"
  location            = var.location
  resource_group_name = var.resource_group_name
  dns_prefix          = var.project_name
  kubernetes_version  = "1.28"

  default_node_pool {
    name                = "system"
    vm_size             = "Standard_D4s_v3"
    node_count          = 3
    min_count           = 3
    max_count           = 10
    enable_auto_scaling = true
    vnet_subnet_id      = var.subnet_id
  }

  identity {
    type = "SystemAssigned"
  }

  # Enable Azure Policy add-on
  azure_policy_enabled = true

  network_profile {
    network_plugin    = "azure"
    load_balancer_sku = "standard"
  }

  tags = {
    Name = "${var.project_name}-aks-policy"
  }
}
```

## Step 2: Assign Built-in Kubernetes Policy Initiative

```hcl
# Assign the Kubernetes cluster pod security baseline standards initiative
resource "azurerm_resource_policy_assignment" "k8s_baseline" {
  name                 = "k8s-pod-security-baseline"
  resource_id          = azurerm_kubernetes_cluster.policy_enabled.id
  policy_definition_id = "/providers/Microsoft.Authorization/policySetDefinitions/a8640138-9b0a-4a28-b8cb-1666c838647d"

  description = "Enforce Kubernetes pod security baseline standards"

  parameters = jsonencode({
    effect = {
      value = "deny"  # deny, audit, or disabled
    }
    excludedNamespaces = {
      value = ["kube-system", "gatekeeper-system", "azure-arc"]
    }
  })
}

# Require resource limits on containers
resource "azurerm_resource_policy_assignment" "require_limits" {
  name                 = "k8s-require-resource-limits"
  resource_id          = azurerm_kubernetes_cluster.policy_enabled.id
  policy_definition_id = "/providers/Microsoft.Authorization/policyDefinitions/e345eecc-fa47-480f-9e88-67dcc122b164"

  parameters = jsonencode({
    effect = {
      value = "deny"
    }
    excludedNamespaces = {
      value = ["kube-system"]
    }
    cpuLimit = {
      value = "2000m"
    }
    memoryLimit = {
      value = "2Gi"
    }
  })
}
```

## Step 3: Custom Policy - Require Specific Labels

```hcl
# Custom policy definition to require specific labels
resource "azurerm_policy_definition" "require_labels" {
  name         = "${var.project_name}-require-k8s-labels"
  policy_type  = "Custom"
  mode         = "Microsoft.Kubernetes.Data"  # Mode for AKS policies
  display_name = "Require environment label on pods"

  policy_rule = jsonencode({
    if = {
      allOf = [
        {
          field = "type"
          equals = "Microsoft.Kubernetes/connectedClusters"
        }
      ]
    }
    then = {
      effect = "deny"
      details = {
        templateInfo = {
          sourceType = "PublicURL"
          url        = "https://raw.githubusercontent.com/Azure/azure-policy/master/built-in-references/Kubernetes/require-labels/template.yaml"
        }
        apiGroups = [""]
        kinds     = ["Pod"]
        values = {
          labelsList = ["environment", "app", "version"]
        }
      }
    }
  })
}

resource "azurerm_resource_policy_assignment" "require_labels" {
  name                 = "require-k8s-labels"
  resource_id          = azurerm_kubernetes_cluster.policy_enabled.id
  policy_definition_id = azurerm_policy_definition.require_labels.id

  parameters = jsonencode({
    effect = { value = "deny" }
  })
}
```

## Step 4: Policy Compliance Reporting

```hcl
# Alert when policy compliance drops
resource "azurerm_monitor_metric_alert" "policy_noncompliant" {
  name                = "${var.project_name}-policy-noncompliant-alert"
  resource_group_name = var.resource_group_name
  scopes              = [azurerm_kubernetes_cluster.policy_enabled.id]
  description         = "Alert when Kubernetes pods are non-compliant with policy"
  severity            = 2

  criteria {
    metric_namespace = "Microsoft.ContainerService/managedClusters"
    metric_name      = "cluster_autoscaler_unschedulable_pods_count"  # Example metric
    aggregation      = "Average"
    operator         = "GreaterThan"
    threshold        = 5
  }

  action {
    action_group_id = var.alert_action_group_id
  }
}
```

## Step 5: Deploy

```bash
tofu init
tofu plan
tofu apply

# Check policy compliance
az policy state summarize \
  --resource <aks-resource-id> \
  --query "results.nonCompliantResources"

# List non-compliant resources
az policy state list \
  --resource <aks-resource-id> \
  --filter "complianceState eq 'NonCompliant'" \
  --output table

# Check Gatekeeper status in cluster
kubectl get constrainttemplate
kubectl get constraints
```

## Conclusion

Azure Policy for AKS runs as OPA Gatekeeper inside the cluster—deploying it may take 15-20 minutes after enabling the add-on as Gatekeeper syncs policies. Always include system namespaces (`kube-system`, `gatekeeper-system`) in `excludedNamespaces` to avoid breaking cluster operations. Start with `effect = "audit"` to assess current compliance before switching to `effect = "deny"`—Audit mode reports violations without blocking deployments, giving teams time to remediate existing workloads. Use policy initiatives (sets of related policies) rather than individual policies to enforce comprehensive security baselines with a single assignment.
