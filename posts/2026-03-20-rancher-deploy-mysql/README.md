# How to Deploy MySQL on Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, MySQL, Database, Helm

Description: Deploy a production-ready MySQL database on Rancher-managed Kubernetes clusters using Helm with persistent storage, replication, and backup configuration.

## Introduction

MySQL is one of the world's most popular relational databases. Deploying MySQL on Rancher-managed Kubernetes clusters enables you to benefit from Kubernetes orchestration-automatic restarts, health checks, and scaling-while maintaining your existing MySQL workloads. This guide covers deploying MySQL using the Bitnami Helm chart with persistent storage and high availability.

## Prerequisites

- Rancher-managed Kubernetes cluster
- Helm 3.x installed
- A StorageClass for persistent volumes
- kubectl with namespace admin access

## Step 1: Add the Bitnami Helm Repository

```bash
# Add Bitnami chart repository

helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Search for available MySQL chart versions
helm search repo bitnami/mysql
```

## Step 2: Create a MySQL Configuration

```yaml
# mysql-values.yaml - Production MySQL configuration
auth:
  # Root password - use a strong password in production
  rootPassword: "MyStr0ngRootP@ss"
  database: "myapp_db"
  username: "myapp_user"
  password: "MyStr0ngP@ss"

# Primary configuration
primary:
  persistence:
    enabled: true
    storageClass: "standard"
    size: 20Gi
  resources:
    requests:
      memory: 512Mi
      cpu: 250m
    limits:
      memory: 1Gi
      cpu: 1000m
  configuration: |
    [mysqld]
    max_connections=200
    innodb_buffer_pool_size=512M
    slow_query_log=1
    slow_query_log_file=/opt/bitnami/mysql/logs/mysqld.log
    long_query_time=2

# Secondary replicas for read scaling
secondary:
  replicaCount: 2
  persistence:
    enabled: true
    storageClass: "standard"
    size: 20Gi
  resources:
    requests:
      memory: 512Mi
      cpu: 250m

# Enable metrics export
metrics:
  enabled: true
  serviceMonitor:
    enabled: true
    namespace: cattle-monitoring-system
    labels:
      release: rancher-monitoring

# Backup configuration
backup:
  enabled: true
  cronjob:
    schedule: "0 2 * * *"  # Daily at 2 AM
    storage:
      size: 50Gi
```

## Step 3: Deploy MySQL

```bash
# Create namespace
kubectl create namespace databases

# Create the secret for MySQL passwords
kubectl create secret generic mysql-passwords \
  --from-literal=mysql-root-password=MyStr0ngRootP@ss \
  --from-literal=mysql-replication-password=ReplicationP@ss \
  --from-literal=mysql-password=MyStr0ngP@ss \
  --namespace=databases

# Install MySQL using Helm
helm install mysql bitnami/mysql \
  --namespace databases \
  --values mysql-values.yaml \
  --set auth.existingSecret=mysql-passwords \
  --wait \
  --timeout 10m

# Check deployment status
kubectl get pods -n databases
kubectl get pvc -n databases
```

## Step 4: Verify MySQL is Running

```bash
# Connect to MySQL primary
kubectl exec -n databases -it $(kubectl get pod -n databases -l app.kubernetes.io/component=primary -o name | head -1) -- \
  mysql -u root -p

# Test query inside the container
kubectl exec -n databases -it mysql-primary-0 -- \
  mysql -u root -p${MYSQL_ROOT_PASSWORD} -e "SHOW DATABASES;"

# Check replication status
kubectl exec -n databases mysql-secondary-0 -- \
  mysql -u root -p${MYSQL_ROOT_PASSWORD} -e "SHOW SLAVE STATUS\G" 2>/dev/null
```

## Step 5: Create a PersistentVolumeClaim

For more control over storage:

```yaml
# mysql-pvc.yaml - Dedicated PVC for MySQL
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data
  namespace: databases
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 50Gi
```

## Step 6: Configure MySQL for Application Access

```yaml
# mysql-service.yaml - Expose MySQL for applications
apiVersion: v1
kind: Service
metadata:
  name: mysql-primary
  namespace: databases
spec:
  selector:
    app.kubernetes.io/name: mysql
    app.kubernetes.io/component: primary
  ports:
    - port: 3306
      targetPort: 3306
  type: ClusterIP
```

Application connection configuration:

```yaml
# app-config.yaml - Application MySQL connection
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-db-config
  namespace: production
data:
  DB_HOST: "mysql-primary.databases.svc.cluster.local"
  DB_PORT: "3306"
  DB_NAME: "myapp_db"
  DB_USER: "myapp_user"
```

## Step 7: Set Up Automated Backups

```yaml
# mysql-backup-cronjob.yaml - Scheduled MySQL backup
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mysql-backup
  namespace: databases
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: mysql-backup
              image: bitnami/mysql:8.0
              command:
                - /bin/sh
                - -c
                - |
                  DATE=$(date +%Y%m%d-%H%M%S)
                  mysqldump \
                    -h mysql-primary \
                    -u root \
                    -p${MYSQL_ROOT_PASSWORD} \
                    --all-databases \
                    --single-transaction \
                    --routines \
                    --triggers \
                    > /backup/mysql-backup-${DATE}.sql
                  echo "Backup completed: mysql-backup-${DATE}.sql"
              env:
                - name: MYSQL_ROOT_PASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: mysql-passwords
                      key: mysql-root-password
              volumeMounts:
                - name: backup-storage
                  mountPath: /backup
          volumes:
            - name: backup-storage
              persistentVolumeClaim:
                claimName: mysql-backup-pvc
          restartPolicy: OnFailure
```

## Step 8: Configure MySQL Monitoring

```bash
# Port forward to access Grafana
kubectl port-forward -n cattle-monitoring-system svc/rancher-monitoring-grafana 3000:80

# Import MySQL dashboard (ID: 7362 for MySQL Overview)
```

## Troubleshooting

```bash
# Check MySQL pod logs
kubectl logs -n databases mysql-primary-0 --tail=100

# Check replication lag
kubectl exec -n databases mysql-secondary-0 -- \
  mysql -u root -p${MYSQL_ROOT_PASSWORD} -e \
  "SELECT * FROM performance_schema.replication_applier_status\G"

# Check MySQL configuration
kubectl exec -n databases mysql-primary-0 -- \
  mysql -u root -p${MYSQL_ROOT_PASSWORD} -e "SHOW VARIABLES LIKE 'innodb_buffer_pool_size';"
```

## Conclusion

Deploying MySQL on Rancher provides enterprise-grade database management with Kubernetes automation. The Bitnami MySQL Helm chart offers a well-tested, configurable deployment with support for replication, backups, and monitoring. For production deployments, always use read replicas to distribute read load, configure automated backups with off-cluster storage, and set up monitoring to detect performance issues early.
