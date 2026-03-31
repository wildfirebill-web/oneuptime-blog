# How to Contribute a Component to the Dapr Project

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Open Source, Contribution, Component, Go

Description: Learn how to contribute a new component to the Dapr components-contrib repository, from implementation and tests to documentation and PR submission.

---

## Contributing to Dapr components-contrib

The `dapr/components-contrib` repository hosts all official Dapr components. Contributing a new component makes it available to every Dapr user without requiring custom builds. The process involves implementing the component interface, writing conformance tests, providing documentation, and submitting a pull request following the project's contribution guidelines.

## Setting Up the Development Environment

```bash
git clone https://github.com/dapr/components-contrib.git
cd components-contrib

# Install Go 1.22+
go version

# Verify tests pass before making changes
go test ./... -short
```

## Implementing a State Store Component

Create the component directory and implementation file:

```bash
mkdir -p state/mystore
touch state/mystore/mystore.go
touch state/mystore/mystore_test.go
touch state/mystore/README.md
```

Implement the `state.Store` interface:

```go
package mystore

import (
    "context"
    "fmt"

    "github.com/dapr/components-contrib/metadata"
    "github.com/dapr/components-contrib/state"
    "github.com/dapr/kit/logger"
)

type MyStore struct {
    logger logger.Logger
    client *myStoreClient
}

func NewMyStore(logger logger.Logger) state.Store {
    return &MyStore{logger: logger}
}

func (s *MyStore) GetComponentMetadata() (metadataInfo metadata.MetadataMap) {
    metadataStruct := myStoreMetadata{}
    metadata.GetMetadataInfoFromStructType(reflect.TypeOf(metadataStruct), &metadataInfo, metadata.StateStoreType)
    return
}

func (s *MyStore) Init(ctx context.Context, metadata state.Metadata) error {
    cfg, err := parseMetadata(metadata.Properties)
    if err != nil {
        return fmt.Errorf("mystore: invalid config: %w", err)
    }
    s.client = newMyStoreClient(cfg)
    return s.client.Connect(ctx)
}

func (s *MyStore) Get(ctx context.Context, req *state.GetRequest) (*state.GetResponse, error) {
    data, etag, err := s.client.Get(ctx, req.Key)
    if err != nil {
        return nil, err
    }
    return &state.GetResponse{Data: data, ETag: &etag}, nil
}

func (s *MyStore) Set(ctx context.Context, req *state.SetRequest) error {
    return s.client.Set(ctx, req.Key, req.Value, req.ETag)
}

func (s *MyStore) Delete(ctx context.Context, req *state.DeleteRequest) error {
    return s.client.Delete(ctx, req.Key)
}
```

## Registering the Component

Add to the component registry in `state/registry.go`:

```go
import mystore "github.com/dapr/components-contrib/state/mystore"

func (s *stateStoreRegistry) DefaultComponents() []state.Store {
    return []state.Store{
        // ... existing components
        mystore.NewMyStore,
    }
}
```

## Writing Conformance Tests

Add a conformance test configuration:

```json
{
  "componentName": "mystore",
  "componentType": "state",
  "operations": ["get", "set", "delete", "bulk", "etag", "ttl"],
  "config": {
    "url": "http://localhost:8080"
  }
}
```

Run the conformance suite:

```bash
go test ./tests/conformance/... \
  -run TestStateStoreConformance/mystore \
  -v
```

## PR Checklist

Before submitting a PR, verify:

```bash
# Run linter
golangci-lint run ./state/mystore/...

# Run unit tests
go test ./state/mystore/... -v

# Verify no breaking changes to existing components
go test ./... -short

# Check documentation exists
ls state/mystore/README.md
```

## Summary

Contributing a component to Dapr's `components-contrib` repository requires implementing the appropriate interface, registering the component in the registry, writing conformance tests that validate spec compliance, and providing a README with configuration documentation. Following the contribution checklist and running linters before submitting a PR accelerates the review process and increases the chances of acceptance on the first submission.
