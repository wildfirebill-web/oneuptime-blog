# How to Use Dapr with Azure API Management

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Azure, API Management, Gateway, Microservice

Description: Integrate Dapr with Azure API Management to expose Dapr service invocation endpoints as managed APIs, adding rate limiting, authentication, and caching at the gateway layer.

---

Azure API Management (APIM) acts as a gateway for Dapr services, providing rate limiting, authentication, transformation, and developer portal features. APIM can route requests to Dapr sidecar endpoints using the Dapr self-hosted mode or via AKS.

## Architecture Overview

```text
Client -> APIM -> Dapr Sidecar -> Microservice
                      |
              [State/PubSub/Secrets]
```

APIM calls the Dapr sidecar HTTP API, which handles service discovery and mTLS to the target service.

## Configure APIM Backend for Dapr

```bash
# Create APIM instance
az apim create \
  --name my-dapr-apim \
  --resource-group my-rg \
  --publisher-email admin@example.com \
  --publisher-name "My Organization" \
  --sku-name Developer \
  --location eastus
```

## Create an API that Proxies Dapr Service Invocation

```xml
<!-- APIM Policy to forward to Dapr sidecar -->
<policies>
  <inbound>
    <base />
    <set-backend-service
      base-url="http://dapr-sidecar:3500/v1.0/invoke/order-service/method" />
    <set-header name="Content-Type" exists-action="override">
      <value>application/json</value>
    </set-header>
    <rate-limit-by-key
      calls="100"
      renewal-period="60"
      counter-key="@(context.Request.Headers.GetValueOrDefault("X-Customer-Id","anonymous"))" />
  </inbound>
  <backend>
    <base />
  </backend>
  <outbound>
    <base />
    <set-header name="X-Powered-By" exists-action="delete" />
  </outbound>
  <on-error>
    <base />
    <return-response>
      <set-status code="500" reason="Internal Server Error" />
      <set-body>{"error": "Service invocation failed"}</set-body>
    </return-response>
  </on-error>
</policies>
```

## Add JWT Validation at the Gateway

```xml
<policies>
  <inbound>
    <base />
    <validate-jwt header-name="Authorization" failed-validation-httpcode="401">
      <openid-config url="https://login.microsoftonline.com/YOUR_TENANT_ID/v2.0/.well-known/openid-configuration" />
      <required-claims>
        <claim name="aud">
          <value>api://my-dapr-api</value>
        </claim>
      </required-claims>
    </validate-jwt>
    <set-backend-service
      base-url="http://dapr-sidecar:3500/v1.0/invoke/order-service/method" />
  </inbound>
</policies>
```

## Expose Dapr Pub/Sub Through APIM

```xml
<!-- APIM operation to publish a Dapr event -->
<policies>
  <inbound>
    <base />
    <set-backend-service
      base-url="http://dapr-sidecar:3500/v1.0/publish/orderpubsub/orders" />
    <set-method>POST</set-method>
    <set-header name="Content-Type" exists-action="override">
      <value>application/json</value>
    </set-header>
  </inbound>
</policies>
```

## Deploy a Dapr Sidecar for APIM Integration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apim-dapr-proxy
spec:
  template:
    metadata:
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "apim-proxy"
        dapr.io/app-port: "8080"
```

## API Definition in APIM

```bash
# Import API definition
az apim api import \
  --resource-group my-rg \
  --service-name my-dapr-apim \
  --api-id orders-api \
  --path /orders \
  --specification-format OpenApi \
  --specification-url https://myrepo.com/openapi/orders.yaml \
  --api-type http

# Set the backend URL to the Dapr sidecar
az apim api update \
  --resource-group my-rg \
  --service-name my-dapr-apim \
  --api-id orders-api \
  --service-url "http://dapr-sidecar:3500/v1.0/invoke/order-service/method"
```

## Summary

Azure API Management provides enterprise-grade API gateway features for Dapr services, including authentication, rate limiting, request transformation, and the developer portal. APIM routes requests to the Dapr sidecar HTTP API, which then handles service discovery, mTLS, and retries to the target microservice. This pattern is ideal when you need to expose internal Dapr services to external consumers while maintaining security and governance.
