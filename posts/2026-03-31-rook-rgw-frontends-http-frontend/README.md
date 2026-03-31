# How to Configure rgw_frontends for HTTP Frontend in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, HTTP, Configuration, Object Storage

Description: Learn how to configure the rgw_frontends parameter to set the HTTP frontend for Ceph RGW, including port binding, SSL, and beast vs. civetweb options.

---

The `rgw_frontends` configuration parameter controls how Ceph RGW listens for incoming S3 and Swift API requests. Getting this right is critical for performance, security, and compatibility.

## Frontend Options

Ceph RGW supports two HTTP frontends:

- **beast** - The modern, default frontend based on Boost.Beast (recommended)
- **civetweb** - Older frontend, still available but deprecated

Check the current frontend configuration:

```bash
ceph config get client.rgw rgw_frontends
```

## Configuring the Beast Frontend

The beast frontend accepts options as key=value pairs separated by spaces:

```bash
# Basic HTTP on port 7480
ceph config set client.rgw rgw_frontends "beast port=7480"

# HTTP on a specific IP
ceph config set client.rgw rgw_frontends "beast endpoint=192.168.1.10:7480"

# HTTPS with SSL certificate
ceph config set client.rgw rgw_frontends "beast ssl_port=443 ssl_certificate=/etc/ceph/rgw.crt ssl_private_key=/etc/ceph/rgw.key"

# Both HTTP and HTTPS simultaneously
ceph config set client.rgw rgw_frontends "beast port=80 ssl_port=443 ssl_certificate=/etc/ceph/rgw.crt ssl_private_key=/etc/ceph/rgw.key"
```

## Tuning Thread Count

The beast frontend exposes a thread pool size option:

```bash
# Set number of worker threads
ceph config set client.rgw rgw_frontends "beast port=7480 num_threads=512"
```

## Configuring in a Rook Cluster

In Rook, set `rgw_frontends` via a Ceph config override or directly on the object store. The Rook operator manages RGW pod configuration, so tuning is done via ConfigMap overrides:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rook-config-override
  namespace: rook-ceph
data:
  config: |
    [client.rgw.my-store.a]
    rgw_frontends = beast port=7480 num_threads=256
```

## Civetweb Configuration (Legacy)

For older deployments still using civetweb:

```bash
ceph config set client.rgw rgw_frontends "civetweb port=7480 num_threads=100"
```

## Verifying the Frontend

After applying configuration, restart RGW and check that it is listening:

```bash
kubectl -n rook-ceph rollout restart deployment rook-ceph-rgw-my-store-a

# Confirm the port is open
kubectl -n rook-ceph exec -it deploy/rook-ceph-rgw-my-store-a -- ss -tlnp | grep 7480
```

## Summary

The `rgw_frontends` parameter controls which HTTP server RGW uses and how it is bound. The beast frontend is the current default and supports HTTP, HTTPS, and thread pool tuning via inline options. In Rook deployments, apply this setting via a config override ConfigMap and restart RGW pods to apply changes.
