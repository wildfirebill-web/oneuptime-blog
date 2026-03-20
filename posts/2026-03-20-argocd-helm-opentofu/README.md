# How to Deploy ArgoCD with Helm and OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, ArgoCD, GitOps, OpenTofu, Helm, CD, Continuous Delivery

Description: Learn how to deploy ArgoCD on Kubernetes using OpenTofu and Helm for GitOps-based continuous delivery with SSO, RBAC, and automated application sync.

## Overview

ArgoCD is a declarative GitOps continuous delivery tool for Kubernetes. It continuously monitors Git repositories and ensures cluster state matches the desired state defined in Git. OpenTofu deploys ArgoCD via Helm and configures SSO, projects, and application resources.

## Step 1: Deploy ArgoCD with Helm

```hcl
# main.tf - Deploy ArgoCD via Helm

resource "helm_release" "argocd" {
  name             = "argocd"
  repository       = "https://argoproj.github.io/argo-helm"
  chart            = "argo-cd"
  version          = "6.7.0"
  namespace        = "argocd"
  create_namespace = true

  values = [yamlencode({
    global = {
      domain = "argocd.example.com"
    }

    server = {
      replicas = 2

      ingress = {
        enabled          = true
        ingressClassName = "nginx"
        annotations = {
          "cert-manager.io/cluster-issuer"                    = "letsencrypt-production"
          "nginx.ingress.kubernetes.io/ssl-passthrough"       = "true"
          "nginx.ingress.kubernetes.io/force-ssl-redirect"    = "true"
        }
        hostname = "argocd.example.com"
        tls      = true
      }

      resources = {
        requests = { cpu = "100m", memory = "128Mi" }
        limits   = { cpu = "500m", memory = "512Mi" }
      }
    }

    # Application controller
    controller = {
      replicas = 1
      resources = {
        requests = { memory = "512Mi", cpu = "250m" }
        limits   = { memory = "1Gi", cpu = "1000m" }
      }
    }

    # Repository server
    repoServer = {
      replicas = 2
    }

    configs = {
      # ArgoCD configuration
      cm = {
        "admin.enabled"          = "true"
        "exec.enabled"           = "true"
        "server.rbac.log.enforce.enable" = "true"

        # OIDC SSO (example with Okta)
        "oidc.config" = yamlencode({
          name         = "Okta"
          issuer       = "https://mycompany.okta.com"
          clientID     = var.oidc_client_id
          clientSecret = "$oidc.okta.clientSecret"
          requestedScopes = ["openid", "profile", "email", "groups"]
        })
      }

      # RBAC policies
      rbac = {
        "policy.default" = "role:readonly"
        "policy.csv"     = <<-RBAC
          p, role:developer, applications, get, */*, allow
          p, role:developer, applications, sync, */*, allow
          p, role:developer, applications, create, dev/*, allow
          g, mycompany:engineering, role:developer
          g, mycompany:platform, role:admin
        RBAC
        "scopes" = "[groups]"
      }

      # Repository credentials
      credentialTemplates = {
        "github-creds" = {
          url           = "https://github.com/mycompany"
          githubAppID   = var.github_app_id
          githubAppInstallationID = var.github_app_install_id
          githubAppPrivateKey     = var.github_app_private_key
        }
      }
    }
  })]
}
```

## Step 2: Create ArgoCD Projects

```hcl
# ArgoCD AppProject for environment isolation
resource "kubernetes_manifest" "argocd_project_production" {
  depends_on = [helm_release.argocd]

  manifest = {
    apiVersion = "argoproj.io/v1alpha1"
    kind       = "AppProject"
    metadata = {
      name      = "production"
      namespace = "argocd"
    }
    spec = {
      description = "Production applications"

      sourceRepos = ["https://github.com/mycompany/*"]

      destinations = [{
        server    = "https://kubernetes.default.svc"
        namespace = "production"
      }]

      # Deny cluster-level resources in production
      clusterResourceWhitelist = []

      namespaceResourceBlacklist = [
        { group = "", kind = "ResourceQuota" }
      ]
    }
  }
}
```

## Step 3: Deploy an Application via ArgoCD

```hcl
# ArgoCD Application resource
resource "kubernetes_manifest" "argocd_app" {
  manifest = {
    apiVersion = "argoproj.io/v1alpha1"
    kind       = "Application"
    metadata = {
      name      = "my-app"
      namespace = "argocd"
      finalizers = ["resources-finalizer.argocd.argoproj.io"]
    }
    spec = {
      project = "production"

      source = {
        repoURL        = "https://github.com/mycompany/k8s-configs"
        targetRevision = "HEAD"
        path           = "apps/my-app"
      }

      destination = {
        server    = "https://kubernetes.default.svc"
        namespace = "production"
      }

      syncPolicy = {
        automated = {
          prune    = true   # Delete resources no longer in Git
          selfHeal = true   # Revert manual changes
        }
        syncOptions = ["CreateNamespace=true"]
        retry = {
          limit = 5
          backoff = {
            duration    = "5s"
            factor      = 2
            maxDuration = "3m"
          }
        }
      }
    }
  }
}
```

## Summary

ArgoCD deployed with OpenTofu enables GitOps continuous delivery where Git is the single source of truth for cluster state. Automated sync with pruning ensures drift is corrected without manual intervention. RBAC policies tied to OIDC group membership enable self-service deployments for developers while preventing unauthorized changes to production environments.
