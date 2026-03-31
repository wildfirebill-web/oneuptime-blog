# How to Read and Navigate the Official Dapr Documentation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Documentation, Learning, Getting Started, Developer Experience

Description: Learn how to effectively navigate the official Dapr documentation to find concepts, API references, component specs, and how-to guides quickly.

---

## Structure of the Dapr Documentation

The official Dapr documentation at `docs.dapr.io` is organized into several top-level sections. Understanding this structure saves significant time when looking for specific information.

- **Concepts** - Core building blocks like state management, pub/sub, actors, and workflows
- **Developing Applications** - Language SDKs, API reference, and code samples
- **Operations** - Deploying, configuring, and operating Dapr in production
- **Reference** - Component specs, CLI reference, configuration schema, and API spec

## Finding the Right Component Spec

When you need to configure a Dapr component (e.g., a Redis state store), navigate to:

```text
Reference > Component specs > State stores > Redis
```

Each component page contains:
- Required and optional metadata fields
- Example YAML
- Version compatibility matrix
- Authentication options

## Using the API Reference

The HTTP API and gRPC API are documented under `Developing Applications > Dapr API`. For example, to find the state management API:

```bash
# HTTP API endpoint structure shown in docs:
GET  http://localhost:3500/v1.0/state/{storeName}/{key}
POST http://localhost:3500/v1.0/state/{storeName}
```

Each endpoint shows required headers, query parameters, request body schema, and response codes.

## Searching Effectively

Use the built-in search and filter by version:

```yaml
https://docs.dapr.io/search/?q=resiliency+policy
```

The docs support versioned browsing - always check that the selected version matches your deployed Dapr version:

```bash
# Check your Dapr runtime version
dapr --version
kubectl get deploy dapr-operator -n dapr-system -o jsonpath='{.spec.template.spec.containers[0].image}'
```

## Reading the How-To Guides

How-to guides under `Developing Applications > How-to` provide step-by-step instructions. Each guide follows this pattern:

1. Prerequisites
2. Component configuration YAML
3. Code sample (multi-language tabs)
4. Verification steps
5. Related links

Always check the "Prerequisites" section first - missing a Dapr CLI installation or component deployment is the most common source of confusion.

## Using the GitHub Repository Alongside the Docs

The docs reference code samples hosted in `github.com/dapr/quickstarts`. Clone it to run examples locally:

```bash
git clone https://github.com/dapr/quickstarts.git
cd quickstarts/state_management/go/sdk
dapr run --app-id order-processor --resources-path ../../../components -- go run .
```

## Checking the Dapr Discord and GitHub Discussions

When documentation is unclear, check:
- GitHub Discussions: `github.com/dapr/dapr/discussions`
- Discord: `discord.com/invite/ptHhX6jc34`

Search for your error message or component name before opening a new issue.

## Summary

The Dapr documentation is organized into Concepts, Developing Applications, Operations, and Reference sections. Navigate to component specs for YAML configuration details and to the API reference for endpoint documentation. Always verify your docs version matches your Dapr runtime version, and use the quickstarts repository to run working examples alongside the documentation.
