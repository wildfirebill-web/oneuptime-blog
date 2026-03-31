# How to Understand Dapr SDK Version Compatibility

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, SDK, Versioning, Compatibility, Go, Python, Java

Description: Understand how Dapr SDK versions align with the runtime, what the compatibility matrix means, and how to safely upgrade SDKs without breaking your services.

---

## Dapr SDK and Runtime Version Alignment

Dapr maintains separate SDK repositories for each language. The SDK minor version generally tracks the Dapr runtime minor version. For example, Go SDK `v1.11.0` is compatible with Dapr runtime `1.11.x`.

The official compatibility policy states:
- SDKs support the current runtime release and one prior minor version
- Patch releases are backward-compatible within the same minor version

## Checking Your Current Versions

Check installed SDK versions per language:

```bash
# Go
grep "github.com/dapr/go-sdk" go.mod

# Python
pip show dapr

# Java (Maven)
grep "dapr-sdk" pom.xml

# Node.js
cat package.json | grep "@dapr/dapr"

# .NET
dotnet list package | grep Dapr
```

Check the Dapr runtime version:

```bash
dapr --version
# or in Kubernetes
kubectl get deploy dapr-operator -n dapr-system -o jsonpath='{.spec.template.spec.containers[0].image}'
```

## Compatibility Matrix Example

| Dapr Runtime | Go SDK | Python SDK | Java SDK | .NET SDK |
|---|---|---|---|---|
| 1.13.x | v1.13.x | v1.13.x | v1.13.x | v1.13.x |
| 1.12.x | v1.12.x | v1.12.x | v1.12.x | v1.12.x |
| 1.11.x | v1.11.x | v1.11.x | v1.11.x | v1.11.x |

Always verify against the official compatibility table at `docs.dapr.io/developing-applications/sdks/`.

## Updating the Go SDK

```bash
# Update to a specific version
go get github.com/dapr/go-sdk@v1.11.0

# Update to latest compatible
go get github.com/dapr/go-sdk@latest

# Tidy dependencies
go mod tidy
```

## Updating the Python SDK

```bash
# Pin to a specific version in requirements.txt
dapr==1.11.0
dapr-ext-grpc==1.11.0
opentelemetry-sdk==1.20.0

# Install
pip install -r requirements.txt
```

## Handling Breaking Changes Between SDK Versions

Before upgrading, check the SDK changelog:

```bash
# Go SDK releases
open https://github.com/dapr/go-sdk/releases

# Python SDK releases
open https://github.com/dapr/python-sdk/releases
```

Common breaking changes include:
- Method signature changes (new required parameters)
- Renamed packages or modules
- Removed deprecated APIs

## Testing SDK Compatibility

Run the Dapr conformance tests to verify your component works with your SDK version:

```bash
# Clone the conformance test suite
git clone https://github.com/dapr/components-contrib.git
cd components-contrib

# Run state store conformance tests
go test ./tests/conformance/... -run TestStateConformance
```

## Using Multi-App Run for Integration Testing

Test your services with the actual Dapr runtime version in multi-app run:

```yaml
version: 1
apps:
  - appID: order-service
    appDirPath: ./order-service
    command: go run .
  - appID: inventory-service
    appDirPath: ./inventory-service
    command: go run .
```

```bash
dapr run -f dapr.yaml
```

## Summary

Dapr SDK versions track the runtime minor version, and the SDKs support the current and one prior minor runtime release. Always verify your SDK version against the official compatibility matrix, read the changelog before upgrading, and use the `dapr run -f` multi-app run feature to test SDK and runtime compatibility together in your local environment.
