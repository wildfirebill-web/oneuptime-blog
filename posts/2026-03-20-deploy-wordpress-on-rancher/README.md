# How to Deploy WordPress on Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, WordPress, Kubernetes, Helm, MySQL, Content Management

Description: Deploy WordPress on Rancher with persistent storage, MySQL database, autoscaling, and Ingress for a production-ready CMS on Kubernetes.

## Introduction

WordPress is the world's most popular CMS. While it wasn't designed for containerization, modern Helm charts make it practical to run on Rancher with proper persistent storage for uploads and a dedicated MySQL database.

## Step 1: Deploy WordPress with Helm

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

```yaml
# wordpress-values.yaml
wordpressUsername: admin
wordpressPassword: "securepassword"
wordpressEmail: admin@example.com
wordpressBlogName: "My WordPress Site"

ingress:
  enabled: true
  ingressClassName: nginx
  hostname: blog.example.com
  tls: true
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod

persistence:
  enabled: true
  storageClass: longhorn
  size: 20Gi    # For media uploads

mariadb:
  enabled: true
  auth:
    database: wordpress
    username: wordpress
    password: "dbpassword"
    rootPassword: "rootpassword"
  primary:
    persistence:
      enabled: true
      storageClass: longhorn
      size: 20Gi

resources:
  requests:
    memory: "256Mi"
    cpu: "100m"
  limits:
    memory: "1Gi"
    cpu: "500m"

replicaCount: 2    # Two WordPress replicas for HA
```

```bash
kubectl create namespace wordpress

helm install wordpress bitnami/wordpress \
  --namespace wordpress \
  --values wordpress-values.yaml
```

## Step 2: Verify Deployment

```bash
# Check all pods are running
kubectl get pods -n wordpress

# Check PVCs are bound
kubectl get pvc -n wordpress

# Verify Ingress
kubectl get ingress -n wordpress
```

## Step 3: Configure Object Storage for Media

For multi-replica deployments, media uploads must be stored in shared object storage (not local filesystem):

```bash
# Install the WP Offload Media plugin (S3) inside WordPress admin
# Or configure directly via wp-config.php

kubectl exec -it wordpress-pod -n wordpress -- \
  wp option update uploads_use_yearmonth_folders 0
```

## Step 4: Configure WordPress for Kubernetes

```yaml
# Add to wordpress-values.yaml
extraEnvVars:
  - name: WORDPRESS_TABLE_PREFIX
    value: wp_
  - name: WORDPRESS_SKIP_INSTALL
    value: "no"
  # Required for reverse proxy deployments
  - name: WORDPRESS_EXTRA_WP_CONFIG_CONTENT
    value: |
      define('FORCE_SSL_ADMIN', true);
      if (strpos($_SERVER['HTTP_X_FORWARDED_PROTO'], 'https') !== false) {
        $_SERVER['HTTPS'] = 'on';
      }
```

## Step 5: Set Up Backups

```bash
# Create a CronJob for WordPress database backup
kubectl apply -f - << 'EOF'
apiVersion: batch/v1
kind: CronJob
metadata:
  name: wordpress-backup
  namespace: wordpress
spec:
  schedule: "0 2 * * *"    # Daily at 2am
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: backup
              image: bitnami/mariadb:latest
              command:
                - /bin/bash
                - -c
                - >
                  mysqldump -h wordpress-mariadb -u wordpress
                  -pdbpassword wordpress |
                  aws s3 cp - s3://my-backups/wordpress-$(date +%Y%m%d).sql
          restartPolicy: Never
EOF
```

## Conclusion

WordPress on Rancher is production-ready with the Bitnami Helm chart handling most configuration. The key challenge for scaled deployments is shared media storage—use S3 offloading for any deployment with more than one WordPress replica, as the local filesystem cannot be shared between pods.
