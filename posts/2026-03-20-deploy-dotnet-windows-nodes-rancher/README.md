# How to Deploy .NET Applications on Windows Nodes in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, .NET, Window, ASP.NET Core, Kubernetes, Microservice

Description: Deploy .NET Framework and .NET Core applications on Windows nodes in Rancher with proper image selection, configuration management, and health checks.

## Introduction

Modern .NET applications (ASP.NET Core) run cross-platform on both Linux and Windows, while .NET Framework applications require Windows. This guide covers deploying both types in Rancher, with guidance on when to choose Linux vs Windows nodes.

## Choose the Right Runtime

```text
.NET Framework 4.x → Must use Windows nodes
.NET Core / .NET 5+ → Prefer Linux nodes (smaller images, better performance)
.NET Core targeting Windows APIs → Windows nodes required
```

## Step 1: Dockerfile for .NET Framework

```dockerfile
# .NET Framework 4.8 Web API

FROM mcr.microsoft.com/dotnet/framework/sdk:4.8-windowsservercore-ltsc2022 AS build
WORKDIR /app

COPY *.csproj .
RUN msbuild /t:restore

COPY . .
RUN msbuild /p:Configuration=Release /p:DeployOnBuild=true \
    /p:PublishProfile=FolderProfile

FROM mcr.microsoft.com/dotnet/framework/aspnet:4.8-windowsservercore-ltsc2022
WORKDIR /inetpub/wwwroot
COPY --from=build /app/publish .
```

## Step 2: Dockerfile for .NET 8 on Windows

```dockerfile
# .NET 8 Nano Server (Windows-specific APIs)
FROM mcr.microsoft.com/dotnet/sdk:8.0-nanoserver-ltsc2022 AS build
WORKDIR /app
COPY . .
RUN dotnet publish -c Release -o /app/out

FROM mcr.microsoft.com/dotnet/aspnet:8.0-nanoserver-ltsc2022
WORKDIR /app
COPY --from=build /app/out .
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

## Step 3: Deployment Manifest

```yaml
# dotnet-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dotnet-api
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: dotnet-api
  template:
    metadata:
      labels:
        app: dotnet-api
    spec:
      nodeSelector:
        kubernetes.io/os: windows
      tolerations:
        - key: os
          operator: Equal
          value: windows
          effect: NoSchedule
      containers:
        - name: api
          image: myregistry/dotnet-api:latest
          ports:
            - containerPort: 8080
          env:
            - name: ASPNETCORE_URLS
              value: "http://+:8080"
            - name: ASPNETCORE_ENVIRONMENT
              value: "Production"
            - name: ConnectionStrings__Database
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: db-connection
          resources:
            requests:
              memory: "256Mi"
              cpu: "100m"
            limits:
              memory: "1Gi"
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 15
```

## Step 4: Configure Structured Logging

```csharp
// Program.cs - Kubernetes-friendly logging
using Serilog;
using Serilog.Events;

Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
    .WriteTo.Console(outputTemplate:
        "{Timestamp:yyyy-MM-ddTHH:mm:ssZ} [{Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}")
    .CreateLogger();

builder.Host.UseSerilog();
```

## Step 5: Graceful Shutdown

```csharp
// Configure graceful shutdown for Kubernetes SIGTERM
builder.Services.AddHostedService<GracefulShutdown>();

// The app should stop accepting new requests on SIGTERM
// and complete in-flight requests within terminationGracePeriodSeconds
```

```yaml
# Allow 30 seconds for graceful shutdown
spec:
  terminationGracePeriodSeconds: 30
```

## Conclusion

.NET applications on Windows nodes in Rancher follow the same Kubernetes patterns as Linux workloads. The key differences are node selectors, base image selection, and the startup time for IIS-hosted applications. For new .NET development, consider targeting Linux nodes with .NET 8 for smaller images and better Kubernetes density.
