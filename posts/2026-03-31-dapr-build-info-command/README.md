# How to Use the dapr build-info Command

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, CLI, Diagnostic, Build, Version

Description: Learn how to use the dapr build-info command to retrieve detailed build metadata about the Dapr CLI for debugging and support purposes.

---

## Overview

The `dapr build-info` command prints detailed build information about the Dapr CLI binary. This includes the exact version, Git commit hash, Go version, OS and architecture, and build timestamp. It is primarily used for bug reports, support tickets, and verifying the exact binary in production environments.

## Basic Usage

```bash
dapr build-info
```

Sample output:

```
Version: 1.13.0
Commit: a1b2c3d4e5f6789012345678901234567890abcd
Date: 2026-01-15T10:00:00Z
Go version: go1.21.5
Os/Arch: darwin/arm64
```

## JSON Output

```bash
dapr build-info --output json
```

Sample JSON output:

```json
{
  "version": "1.13.0",
  "commit": "a1b2c3d4e5f6789012345678901234567890abcd",
  "date": "2026-01-15T10:00:00Z",
  "goVersion": "go1.21.5",
  "os": "darwin",
  "arch": "arm64"
}
```

## When to Use build-info

Use `dapr build-info` in the following scenarios:

1. **Filing a bug report** - include the full output so maintainers can identify if the issue was fixed in a later build
2. **Verifying a custom build** - confirm that a binary built from source matches the expected commit
3. **Auditing environments** - confirm that all team members and CI agents are using identical CLI binaries
4. **Comparing Go versions** - identify potential runtime differences caused by different Go compiler versions

## Capturing Build Info in CI

Store build information as a CI artifact for traceability:

```bash
#!/bin/bash
BUILD_INFO=$(dapr build-info --output json)
echo $BUILD_INFO | jq '.' > dapr-cli-build-info.json

echo "CLI Build Info:"
echo $BUILD_INFO | jq '{version, commit, goVersion}'
```

## Comparing CLI Builds Across Machines

If behavior differs between environments, compare build info:

```bash
# Machine A
dapr build-info --output json | jq '{commit, goVersion}' > machine-a-info.json

# Machine B
dapr build-info --output json | jq '{commit, goVersion}' > machine-b-info.json

diff machine-a-info.json machine-b-info.json
```

Any difference in commit hash means the binaries are different builds.

## Difference Between build-info and version

| Command | Output |
|---|---|
| `dapr version` | Shows CLI and runtime version numbers |
| `dapr build-info` | Shows CLI commit hash, Go version, OS/arch, build date |

Use `dapr version` for day-to-day compatibility checks and `dapr build-info` for deep diagnostics and support.

## Summary

`dapr build-info` provides the full provenance of your Dapr CLI binary. While rarely needed in normal operation, it is invaluable when filing bug reports, auditing production toolchains, or diagnosing subtle differences between environments where the same version string could mask different commit hashes.
