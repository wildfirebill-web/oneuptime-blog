# How to Configure Custom MIME Types for Rook Object Store

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Object Store, Configuration, S3

Description: Learn how to configure custom MIME types for the Rook Ceph object store so RGW serves correct Content-Type headers for your objects.

---

## Why Custom MIME Types Matter

When you serve objects from a Rook-managed S3-compatible object store, browsers and HTTP clients rely on the `Content-Type` response header to interpret content correctly. By default, Ceph RGW uses a built-in MIME type database. If you serve custom file extensions - for example `.wasm`, `.avif`, or internal data formats - you need to extend that database so RGW responds with the correct content type.

## How Ceph RGW Handles MIME Types

RGW looks up MIME types from a file on the filesystem, typically at `/etc/mime.types` inside the RGW container. If a file extension is not present there, RGW falls back to `application/octet-stream`. You can provide a custom or extended MIME types file by injecting it via a ConfigMap and mounting it into the RGW pod.

## Creating a Custom MIME Types ConfigMap

First, create a ConfigMap containing your extended MIME types file:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rgw-custom-mime-types
  namespace: rook-ceph
data:
  mime.types: |
    application/wasm wasm
    image/avif avif
    application/x-parquet parquet
    text/markdown md
    application/geo+json geojson
    font/woff2 woff2
```

Apply it to the cluster:

```bash
kubectl apply -f rgw-custom-mime-types.yaml
```

## Mounting the MIME Types File into RGW

Use `rgwCommandFlags` to point RGW at your custom file, and add a volume mount via a Rook patch or a custom deployment overlay. First tell RGW where to find the file:

```yaml
spec:
  gateway:
    rgwCommandFlags:
      rgw_mime_types_file: "/etc/ceph/custom-mime.types"
```

Then patch the RGW deployment to mount the ConfigMap. If you manage the cluster with Helm or Kustomize, add a strategic merge patch:

```yaml
# rgw-mime-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rook-ceph-rgw-my-store-a
  namespace: rook-ceph
spec:
  template:
    spec:
      volumes:
        - name: mime-types
          configMap:
            name: rgw-custom-mime-types
      containers:
        - name: rgw
          volumeMounts:
            - name: mime-types
              mountPath: /etc/ceph/custom-mime.types
              subPath: mime.types
```

## Verifying MIME Type Resolution

Upload a test file and check the response headers:

```bash
aws s3 cp hello.wasm s3://my-bucket/ \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph.svc:80

aws s3api head-object \
  --bucket my-bucket \
  --key hello.wasm \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph.svc:80
```

The response should show `"ContentType": "application/wasm"`.

## Using S3 Content-Type Override

As an alternative to server-side MIME configuration, clients can explicitly set the `Content-Type` header at upload time using the `--content-type` flag:

```bash
aws s3 cp hello.wasm s3://my-bucket/ \
  --content-type application/wasm \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph.svc:80
```

RGW stores and returns exactly what was provided, bypassing MIME lookup entirely. This works well when you control the upload pipeline.

## Summary

Custom MIME types for Rook object store are configured by providing an extended MIME types file, pointing RGW at it via `rgwCommandFlags`, and mounting it into the RGW pod. For cases where server-side MIME resolution is not flexible enough, uploaders can set `Content-Type` explicitly. Both approaches ensure HTTP clients receive accurate content type headers when downloading objects.
