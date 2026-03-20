# How to Deploy Nextcloud on Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Nextcloud, File Sharing, Kubernetes, Helm, Collaboration

Description: Deploy Nextcloud on Rancher for self-hosted file sharing and collaboration with PostgreSQL backend, persistent storage, and Ingress configuration.

## Introduction

Nextcloud is an open-source file sharing and collaboration platform-a self-hosted alternative to Dropbox, Google Drive, and Microsoft 365. Deploying it on Rancher gives organizations full control over their data with enterprise features.

## Step 1: Deploy Nextcloud with Helm

```bash
helm repo add nextcloud https://nextcloud.github.io/helm/
helm repo update
```

```yaml
# nextcloud-values.yaml

nextcloud:
  host: nextcloud.example.com
  username: admin
  password: "securepassword"
  extraEnv:
    - name: NEXTCLOUD_TRUSTED_DOMAINS
      value: "nextcloud.example.com"

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/proxy-body-size: "10g"    # Allow large file uploads
  tls:
    - secretName: nextcloud-tls
      hosts:
        - nextcloud.example.com

persistence:
  enabled: true
  storageClass: longhorn
  size: 500Gi    # Storage for uploaded files

postgresql:
  enabled: true
  global:
    postgresql:
      auth:
        database: nextcloud
        username: nextcloud
        password: "dbpassword"
  primary:
    persistence:
      enabled: true
      storageClass: longhorn
      size: 20Gi

resources:
  requests:
    memory: "512Mi"
    cpu: "250m"
  limits:
    memory: "2Gi"
    cpu: "2"
```

```bash
kubectl create namespace nextcloud
helm install nextcloud nextcloud/nextcloud \
  --namespace nextcloud \
  --values nextcloud-values.yaml
```

## Step 2: Configure External Object Storage

For large deployments, offload files to S3 instead of local PVCs:

```bash
# Configure S3-compatible external storage via Nextcloud admin
kubectl exec -it nextcloud-pod -n nextcloud -- \
  php occ config:system:set objectstore class --value='\OC\Files\ObjectStore\S3'

kubectl exec -it nextcloud-pod -n nextcloud -- \
  php occ config:system:set objectstore arguments \
    --value='{"bucket":"nextcloud-data","region":"us-east-1","key":"ACCESS_KEY","secret":"SECRET_KEY"}'
```

## Step 3: Configure Background Jobs

```bash
# Set up cron for background jobs
kubectl apply -f - << 'EOF'
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nextcloud-cron
  namespace: nextcloud
spec:
  schedule: "*/5 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: cron
              image: nextcloud:latest
              command: ["php", "-f", "/var/www/html/cron.php"]
              volumeMounts:
                - name: nextcloud-data
                  mountPath: /var/www/html
          restartPolicy: Never
EOF
```

## Step 4: Enable High-Performance Storage (Redis)

```bash
# Deploy Redis for Nextcloud transactional file locking
helm install redis bitnami/redis \
  --namespace nextcloud \
  --set auth.password=redispassword

# Configure Nextcloud to use Redis
kubectl exec -it nextcloud-pod -n nextcloud -- \
  php occ config:system:set redis host --value=redis-master.nextcloud.svc.cluster.local
```

## Conclusion

Nextcloud on Rancher provides a self-hosted collaboration platform with full data sovereignty. The combination of S3 external storage for files and Redis for locking enables horizontal scaling of the Nextcloud application tier while maintaining data consistency.
