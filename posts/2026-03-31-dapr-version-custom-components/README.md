# How to Version Custom Dapr Components

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Versioning, Custom Component, SemVer, Compatibility

Description: Learn how to version custom Dapr components with semantic versioning, maintain backward compatibility, and manage component upgrades safely.

---

## Versioning Principles for Dapr Components

Custom Dapr components follow semantic versioning (semver): MAJOR.MINOR.PATCH. Dapr component YAMLs include a `version` field that must match what the component implementation declares. Breaking changes require a version bump and a migration plan for existing state or message data.

## Declaring Component Version in Implementation

Your component must report its version through the gRPC init response and through metadata:

```go
package mystore

const (
    ComponentVersion = "v2"
    ComponentName    = "state.mystore"
)

type MyStore struct {
    logger logger.Logger
    version string
}

func NewMyStore(logger logger.Logger) state.Store {
    return &MyStore{
        logger:  logger,
        version: ComponentVersion,
    }
}

func (s *MyStore) GetComponentMetadata() (metadataInfo metadata.MetadataMap) {
    metadataStruct := MyStoreMetadata{}
    metadata.GetMetadataInfoFromStructType(
        reflect.TypeOf(metadataStruct),
        &metadataInfo,
        metadata.StateStoreType,
    )
    return
}
```

## Semver in Docker Image Tags

Tag images with full semver for clarity:

```bash
# Build with version tags
docker build \
  --label "org.opencontainers.image.version=2.1.0" \
  -t myorg/dapr-custom-state:2.1.0 \
  -t myorg/dapr-custom-state:v2 \
  -t myorg/dapr-custom-state:latest \
  .

docker push myorg/dapr-custom-state:2.1.0
docker push myorg/dapr-custom-state:v2
```

## Component YAML Versioning

The `version` field in the component YAML maps to your component's API version, not the image tag:

```yaml
# v1 component - original API
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: mystore-v1
spec:
  type: state.mystore
  version: v1
  metadata:
  - name: socketFolder
    value: "/tmp/dapr-components-sockets"
---
# v2 component - new API with additional metadata
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: mystore-v2
spec:
  type: state.mystore
  version: v2
  metadata:
  - name: socketFolder
    value: "/tmp/dapr-components-sockets"
  - name: compressionEnabled
    value: "true"
```

## Supporting Multiple Versions Simultaneously

Run both v1 and v2 component processes with different socket names:

```bash
# Start v1 component
./custom-state-v1 --socket /tmp/dapr-components-sockets/mystore-v1.sock &

# Start v2 component
./custom-state-v2 --socket /tmp/dapr-components-sockets/mystore-v2.sock &
```

This allows gradual migration without a hard cutover.

## Changelog and Migration Notes

Maintain a `CHANGELOG.md` in your component repository:

```markdown
## v2.0.0 - 2026-03-31

### Breaking Changes
- Renamed metadata property `url` to `connectionString`
- Removed support for insecure connections (TLS now required)

### New Features
- Added compression support via `compressionEnabled` metadata
- Bulk operations now support up to 1000 items (previously 100)

### Migration
Update component YAML: change `url` to `connectionString`.
Enable TLS on your backend before upgrading.
```

## Summary

Versioning custom Dapr components requires consistent semver tags in Docker images, a `version` field in the component YAML that matches the implementation's declared version, and a clear changelog documenting breaking changes. Running v1 and v2 components simultaneously on different sockets enables zero-downtime migration when breaking changes are unavoidable.
