# How to Integrate Jenkins with Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Jenkins, CI/CD

Description: Integrate Jenkins with Rancher to enable Kubernetes-native CI/CD pipelines, including dynamic agent provisioning and automated cluster deployments.

## Introduction

Jenkins is widely used for CI/CD pipelines. Integrating Jenkins with Rancher allows you to run Jenkins pipelines on dynamic Kubernetes-based agents, deploy applications to Rancher-managed clusters, and trigger cluster-level operations from pipeline steps. This guide covers setting up Jenkins on Rancher and configuring it to deploy to downstream clusters.

## Step 1: Deploy Jenkins on a Rancher-Managed Cluster

```bash
# Add the Jenkins Helm chart
helm repo add jenkins https://charts.jenkins.io
helm repo update

# Create a values file for production Jenkins
cat << 'EOF' > jenkins-values.yaml
controller:
  # Set admin password
  adminUser: admin
  adminPassword: ChangeMeNow!

  # Configure JVM options
  javaOpts: "-Xms512m -Xmx2g"

  # Install required plugins
  installPlugins:
    - kubernetes:latest
    - workflow-aggregator:latest
    - git:latest
    - configuration-as-code:latest
    - blueocean:latest
    - kubernetes-credentials:latest
    - rancher:latest         # Optional: Rancher plugin

  # Persistent storage
  persistence:
    enabled: true
    size: 50Gi
    storageClass: "default"

agent:
  enabled: true
  defaultsProviderTemplate: ""
  resources:
    requests:
      cpu: 500m
      memory: 512Mi
    limits:
      cpu: 2
      memory: 2Gi
EOF

# Install Jenkins
helm install jenkins jenkins/jenkins \
  --namespace jenkins \
  --create-namespace \
  -f jenkins-values.yaml

# Get the admin password
kubectl get secret -n jenkins jenkins \
  -o jsonpath="{.data.jenkins-admin-password}" | base64 -d
```

## Step 2: Configure Kubernetes Plugin for Dynamic Agents

Jenkins' Kubernetes plugin provisions pods as build agents on demand:

```groovy
// Jenkinsfile — Declare a Kubernetes agent
pipeline {
  agent {
    kubernetes {
      yaml '''
        apiVersion: v1
        kind: Pod
        spec:
          containers:
          - name: maven
            image: maven:3.9-jdk-17
            command: [sleep]
            args: [infinity]
            resources:
              requests:
                cpu: 500m
                memory: 1Gi
          - name: docker
            image: docker:24-dind
            securityContext:
              privileged: true
            command: [dockerd-entrypoint.sh]
      '''
    }
  }
  stages {
    stage('Build') {
      steps {
        container('maven') {
          sh 'mvn clean package -DskipTests'
        }
      }
    }
  }
}
```

## Step 3: Create Kubeconfig Credentials for Rancher Clusters

```bash
# Generate a kubeconfig for the target cluster from Rancher
# In Rancher UI: Cluster → Download KubeConfig

# Or via API:
curl -sk -X POST \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  "https://<rancher-url>/v3/clusters/<cluster-id>?action=generateKubeconfig" \
  | jq -r .config > /tmp/target-cluster.kubeconfig

# Store as a Jenkins credential
# Jenkins UI: Manage Jenkins → Credentials → Global → Add Credential
# Type: Secret file
# File: target-cluster.kubeconfig
# ID: rancher-prod-kubeconfig
```

## Step 4: Build a Deploy Pipeline

```groovy
// Jenkinsfile — Build, push, and deploy to Rancher cluster
pipeline {
  agent {
    kubernetes {
      yaml '''
        apiVersion: v1
        kind: Pod
        spec:
          containers:
          - name: maven
            image: maven:3.9-jdk-17
            command: [sleep]
            args: [infinity]
          - name: kubectl
            image: bitnami/kubectl:latest
            command: [sleep]
            args: [infinity]
      '''
    }
  }

  environment {
    KUBECONFIG = credentials('rancher-prod-kubeconfig')
    IMAGE_TAG  = "${env.GIT_COMMIT[0..7]}"
  }

  stages {
    stage('Build') {
      steps {
        container('maven') {
          sh 'mvn clean package -DskipTests'
        }
      }
    }

    stage('Deploy to Rancher') {
      steps {
        container('kubectl') {
          sh """
            # Update the deployment image
            kubectl set image deployment/myapp \
              myapp=registry.example.com/myapp:${IMAGE_TAG} \
              --kubeconfig=\${KUBECONFIG}

            # Wait for rollout
            kubectl rollout status deployment/myapp \
              --timeout=5m \
              --kubeconfig=\${KUBECONFIG}
          """
        }
      }
    }
  }

  post {
    failure {
      // Auto-rollback on failure
      container('kubectl') {
        sh 'kubectl rollout undo deployment/myapp --kubeconfig=${KUBECONFIG}'
      }
    }
  }
}
```

## Step 5: Use the Rancher API from Jenkins

```groovy
// Trigger a Rancher cluster operation via API
stage('Scale Cluster') {
  steps {
    script {
      withCredentials([string(credentialsId: 'rancher-api-token', variable: 'RANCHER_TOKEN')]) {
        sh """
          curl -sk -X POST \
            -H "Authorization: Bearer \${RANCHER_TOKEN}" \
            -H "Content-Type: application/json" \
            -d '{"nodeCount": 5}' \
            "https://rancher.example.com/v3/nodepools/<nodepool-id>?action=scaleNodePool"
        """
      }
    }
  }
}
```

## Step 6: Trigger Pipelines on Rancher Events (Webhook)

```bash
# Create a Jenkins webhook trigger for Rancher alert events
# In Jenkins: Pipeline → Triggers → Generic Webhook Trigger Plugin

# In Rancher Alertmanager, configure a webhook alert:
# Monitoring → Alerting → Add Receiver → Webhook
# URL: https://jenkins.example.com/generic-webhook-trigger/invoke?token=my-token
```

## Conclusion

Integrating Jenkins with Rancher creates a powerful CI/CD foundation: dynamic Kubernetes build agents eliminate static infrastructure, and kubeconfig-based deployments enable precise multi-cluster targeting. By combining Jenkins' mature pipeline ecosystem with Rancher's cluster management APIs, you can build sophisticated deployment workflows that span multiple environments and clouds.
