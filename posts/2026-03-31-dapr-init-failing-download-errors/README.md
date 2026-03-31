# How to Fix 'dapr init' Failing with Download Errors

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Installation, Troubleshooting, Docker, CLI

Description: Learn how to diagnose and fix 'dapr init' command failures caused by download errors, network issues, and proxy settings.

---

Running `dapr init` is the first step to getting Dapr up and running, but network issues, proxy settings, and rate limits can cause it to fail before you even write a line of application code.

## Common Causes of Download Failures

The `dapr init` command downloads several components: the Dapr runtime binaries, the default Redis and Zipkin Docker images, and the Dapr dashboard. Any of these can fail due to:

- Firewall or proxy restrictions blocking GitHub or Docker Hub
- Rate limiting on Docker Hub (especially for unauthenticated pulls)
- Slow or intermittent network connections timing out
- Incorrect proxy environment variables

## Checking What Failed

Run init with verbose output to see exactly where it stalls:

```bash
dapr init --log-as-json
```

You can also check the Dapr binary directory after a partial install:

```bash
ls ~/.dapr/bin/
```

If the directory is empty or missing binaries, the download itself failed. If Docker containers are missing, run:

```bash
docker ps -a | grep dapr
```

## Fixing Proxy and Network Issues

If you are behind a corporate proxy, set the standard proxy environment variables before running init:

```bash
export HTTP_PROXY=http://proxy.example.com:3128
export HTTPS_PROXY=http://proxy.example.com:3128
export NO_PROXY=localhost,127.0.0.1
dapr init
```

For Docker Hub rate limiting, log in before initializing:

```bash
docker login
dapr init
```

## Specifying a Custom Runtime Version

If the latest release download is failing, try a specific stable version:

```bash
dapr init --runtime-version 1.13.0
```

You can list available versions from GitHub:

```bash
curl -s https://api.github.com/repos/dapr/dapr/releases | jq '.[].tag_name' | head -10
```

## Using a Private Registry Mirror

In air-gapped or heavily restricted environments, use a registry mirror and a custom GitHub endpoint:

```bash
dapr init \
  --image-registry myregistry.internal/dapr \
  --runtime-version 1.13.0
```

Pre-pull the required images to your mirror:

```bash
docker pull daprio/dapr:1.13.0
docker tag daprio/dapr:1.13.0 myregistry.internal/dapr/dapr:1.13.0
docker push myregistry.internal/dapr/dapr:1.13.0
```

## Verifying a Successful Init

After resolving the issue, confirm everything is running:

```bash
dapr status
docker ps | grep dapr
```

You should see `dapr_redis`, `dapr_zipkin`, and `dapr_placement` containers running.

## Summary

`dapr init` failures are almost always caused by network restrictions blocking downloads from GitHub or Docker Hub. Setting proxy variables, authenticating with Docker, or pointing to a private registry mirror resolves most issues. Use verbose logging to identify the exact failure point before troubleshooting.
