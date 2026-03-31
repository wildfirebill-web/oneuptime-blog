# How to Deploy Windows Containers in Rancher - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Window, Containers, Kubernetes, .NET

Description: Deploy Windows containers in Rancher Kubernetes clusters using proper node selectors, compatible base images, and Windows-specific configuration.

## Introduction

Windows containers allow you to run Windows-native applications-including legacy .NET Framework apps, IIS, and Windows services-in Kubernetes. This guide covers deploying Windows containers in Rancher with correct image selection, scheduling configuration, and compatibility considerations.

## Prerequisites

- Rancher cluster with Windows worker nodes
- Windows container images in your registry
- kubectl configured for the cluster
- Understanding of Windows container base images

## Step 1: Choose the Right Windows Base Image

```dockerfile
# Windows container image compatibility matrix:

# - Windows Server 2022 nodes: can run Server 2022, LTSC 2019 images (with Hyper-V)
# - Windows Server 2019 nodes: can run LTSC 2019 images

# For .NET Framework 4.x applications
FROM mcr.microsoft.com/dotnet/framework/runtime:4.8-windowsservercore-ltsc2022

# For .NET 8 applications
FROM mcr.microsoft.com/dotnet/runtime:8.0-nanoserver-ltsc2022

# For ASP.NET Core on Windows
FROM mcr.microsoft.com/dotnet/aspnet:8.0-nanoserver-ltsc2022

# For IIS-based applications
FROM mcr.microsoft.com/windows/servercore/iis:windowsservercore-ltsc2022

# Build stage
FROM mcr.microsoft.com/dotnet/sdk:8.0-windowsservercore-ltsc2022 AS build
WORKDIR /app
COPY *.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app/publish

# Runtime stage
FROM mcr.microsoft.com/dotnet/aspnet:8.0-nanoserver-ltsc2022
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

## Step 2: Deploy a Windows Container

```yaml
# windows-deployment.yaml - Deploy Windows container
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dotnet-app
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: dotnet-app
  template:
    metadata:
      labels:
        app: dotnet-app
    spec:
      # REQUIRED: Schedule on Windows nodes
      nodeSelector:
        kubernetes.io/os: windows

      # Required if Windows nodes are tainted
      tolerations:
        - key: os
          operator: Equal
          value: windows
          effect: NoSchedule

      containers:
        - name: dotnet-app
          image: registry.example.com/dotnet-app:v1.0
          ports:
            - containerPort: 8080
              protocol: TCP
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: Production
            - name: ASPNETCORE_URLS
              value: http://+:8080
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: 2000m
              memory: 2Gi
          # Windows containers use different readiness probe timing
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 60  # Windows containers take longer to start
            periodSeconds: 10
            failureThreshold: 5
```

## Step 3: Service and Ingress for Windows Apps

```yaml
# windows-service.yaml - Expose Windows app
apiVersion: v1
kind: Service
metadata:
  name: dotnet-app
  namespace: production
spec:
  selector:
    app: dotnet-app
  ports:
    - name: http
      port: 80
      targetPort: 8080
  type: ClusterIP
---
# windows-ingress.yaml - Ingress for Windows app
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dotnet-app
  namespace: production
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: dotnet-app
                port:
                  number: 80
```

## Step 4: ConfigMap and Secrets for Windows Apps

```yaml
# windows-config.yaml - Configuration for Windows containers
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  # Windows path separators use backslash in app, but forward slash works too
  log-path: "C:\\app\\logs"
  config-path: "C:\\app\\config"
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
type: Opaque
stringData:
  connection-string: "Server=sql.example.com;Database=AppDB;User=app;Password=secret"
  api-key: "my-api-key-value"
```

## Step 5: Windows Job for Batch Processing

```yaml
# windows-job.yaml - Windows container as a Kubernetes Job
apiVersion: batch/v1
kind: Job
metadata:
  name: data-migration
  namespace: production
spec:
  template:
    spec:
      nodeSelector:
        kubernetes.io/os: windows
      tolerations:
        - key: os
          value: windows
          effect: NoSchedule
      restartPolicy: Never
      containers:
        - name: migration
          image: registry.example.com/migration-tool:v1.0
          command:
            - "powershell.exe"
            - "-Command"
            - "& C:\\migration\\Migrate.ps1"
          env:
            - name: DB_CONNECTION
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: connection-string
          resources:
            requests:
              cpu: 1000m
              memory: 1Gi
```

## Step 6: StatefulSet with Windows Persistent Storage

```yaml
# windows-statefulset.yaml - Windows app with persistent storage
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: legacy-app
  namespace: production
spec:
  serviceName: legacy-app
  replicas: 1
  selector:
    matchLabels:
      app: legacy-app
  template:
    spec:
      nodeSelector:
        kubernetes.io/os: windows
      tolerations:
        - key: os
          value: windows
          effect: NoSchedule
      containers:
        - name: app
          image: registry.example.com/legacy-app:v1.0
          volumeMounts:
            - name: data
              mountPath: C:\app\data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: local-path
        resources:
          requests:
            storage: 10Gi
```

## Step 7: Debugging Windows Containers

```powershell
# Exec into a running Windows container
kubectl exec -it \
  $(kubectl get pod -l app=dotnet-app -o name | head -1) \
  -n production \
  -- powershell.exe

# View Windows container logs
kubectl logs deployment/dotnet-app -n production --tail=100

# Describe pod for Windows-specific events
kubectl describe pod -l app=dotnet-app -n production
```

## Conclusion

Deploying Windows containers in Rancher enables hybrid workloads where legacy Windows applications run alongside modern Linux microservices in the same cluster. The key requirements are matching Windows host OS version with container image version, always specifying `nodeSelector: kubernetes.io/os: windows`, and accounting for longer Windows container startup times in readiness probe configuration. The Windows Server Core base image provides maximum compatibility with legacy applications, while Nano Server offers a smaller footprint for modern .NET applications.
