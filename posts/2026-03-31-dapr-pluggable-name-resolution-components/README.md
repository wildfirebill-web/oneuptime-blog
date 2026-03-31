# How to Develop Dapr Pluggable Name Resolution Components

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Pluggable Component, Name Resolution, Service Discovery, gRPC

Description: Build a custom Dapr name resolution component to integrate with non-standard service registries using the pluggable component SDK.

---

## Dapr Name Resolution

Dapr uses name resolution components to discover the network address of services when handling service invocation calls. Built-in resolvers include Kubernetes (DNS-based), mDNS (local development), and Consul. A pluggable name resolver lets you integrate with custom service registries like Eureka, etcd, or an internal service catalog.

## How Name Resolution Works

When Service A calls Service B via Dapr:
1. Service A calls Dapr sidecar: `POST /v1.0/invoke/service-b/method/endpoint`
2. Dapr calls the name resolver: "Where is service-b?"
3. The resolver returns an address (e.g., `10.0.1.42:3500`)
4. Dapr connects to Service B's sidecar at that address

## Project Setup

```bash
mkdir dapr-custom-resolver && cd dapr-custom-resolver
go mod init github.com/myorg/dapr-custom-resolver
go get github.com/dapr-sandbox/components-go-sdk@latest
```

## Implementing the Name Resolver Interface

```go
package main

import (
    "context"
    "fmt"
    "net/http"
    "encoding/json"

    dapr "github.com/dapr-sandbox/components-go-sdk"
    nr "github.com/dapr-sandbox/components-go-sdk/nameresolution/v1"
    proto "github.com/dapr/dapr/pkg/proto/components/v1"
)

type ServiceCatalogResolver struct {
    catalogURL string
    httpClient *http.Client
}

type ServiceEntry struct {
    Host string `json:"host"`
    Port int    `json:"port"`
}

func (r *ServiceCatalogResolver) Init(ctx context.Context, req *proto.NameResolutionInitRequest) (*proto.NameResolutionInitResponse, error) {
    for _, m := range req.Metadata.Properties {
        if m.Key == "catalogURL" {
            r.catalogURL = m.Value
        }
    }
    r.httpClient = &http.Client{}
    return &proto.NameResolutionInitResponse{}, nil
}

func (r *ServiceCatalogResolver) Resolve(ctx context.Context, req *proto.ResolveRequest) (*proto.ResolveResponse, error) {
    // Look up the service in your registry
    url := fmt.Sprintf("%s/services/%s", r.catalogURL, req.Id)

    resp, err := r.httpClient.Get(url)
    if err != nil {
        return nil, fmt.Errorf("failed to query catalog for %q: %w", req.Id, err)
    }
    defer resp.Body.Close()

    if resp.StatusCode == http.StatusNotFound {
        return nil, fmt.Errorf("service %q not found in catalog", req.Id)
    }

    var entry ServiceEntry
    if err := json.NewDecoder(resp.Body).Decode(&entry); err != nil {
        return nil, err
    }

    return &proto.ResolveResponse{
        Address: fmt.Sprintf("%s:%d", entry.Host, entry.Port),
    }, nil
}

func (r *ServiceCatalogResolver) Ping(ctx context.Context, req *proto.PingRequest) (*proto.PingResponse, error) {
    return &proto.PingResponse{}, nil
}
```

## Main Entry Point

```go
func main() {
    dapr.Register("custom-catalog-resolver", dapr.WithNameResolver(func() nr.Resolver {
        return &ServiceCatalogResolver{}
    }))

    dapr.MustRun()
}
```

## Component Manifest

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: catalog-resolver
spec:
  type: nameresolution.custom-catalog-resolver
  version: v1
  metadata:
    - name: catalogURL
      value: "http://service-catalog.internal:8080"
```

## Dapr Configuration to Use Custom Resolver

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: appconfig
spec:
  nameResolution:
    component: "catalog-resolver"
    configuration:
      refreshInterval: "30s"
```

## Testing the Name Resolver

```bash
# Build and run the resolver
go build -o custom-resolver .
DAPR_COMPONENT_SOCKET_FOLDER=/tmp/dapr-components ./custom-resolver &

# Test service invocation with custom resolver
curl http://localhost:3500/v1.0/invoke/payment-service/method/charge \
  -H "Content-Type: application/json" \
  -d '{"amount": 100, "currency": "USD"}'

# Check resolver logs for resolution calls
tail -f /tmp/custom-resolver.log
```

## Caching Resolution Results

Add a simple cache to reduce registry load:

```go
type ServiceCatalogResolver struct {
    catalogURL string
    httpClient *http.Client
    cache      sync.Map
    cacheTTL   time.Duration
}

type cacheEntry struct {
    address   string
    expiresAt time.Time
}

func (r *ServiceCatalogResolver) Resolve(ctx context.Context, req *proto.ResolveRequest) (*proto.ResolveResponse, error) {
    if entry, ok := r.cache.Load(req.Id); ok {
        ce := entry.(cacheEntry)
        if time.Now().Before(ce.expiresAt) {
            return &proto.ResolveResponse{Address: ce.address}, nil
        }
    }
    // ... fetch from catalog and cache result
}
```

## Summary

Dapr pluggable name resolution components enable service discovery via custom registries by implementing a simple Resolve gRPC method. This is particularly valuable in organizations with existing service catalogs - Consul, Eureka, or proprietary registries - allowing Dapr's service invocation building block to work seamlessly without migrating to Kubernetes-native DNS.
