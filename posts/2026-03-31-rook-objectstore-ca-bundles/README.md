# How to Configure CA Bundles for Rook Object Store

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, TLS, Object Storage, Kubernetes

Description: Learn how to configure custom CA bundles for Rook object store so clients can verify TLS certificates from internal or self-signed certificate authorities.

---

## Why CA Bundles Are Needed

When you configure your Rook object store with TLS (`securePort`), clients connecting to it must trust the certificate authority (CA) that signed the RGW certificate. Public CAs are trusted by default in most systems, but if you use an internal CA or self-signed certificates, clients will reject the connection unless the CA certificate is explicitly trusted.

Rook supports injecting a custom CA bundle into the RGW deployment so that both the gateway and the operator can validate certificates from internal PKI.

## Creating the CA Bundle Secret

Store your CA certificate in a Kubernetes secret. The key name must be `ca.crt`:

```bash
kubectl -n rook-ceph create secret generic rgw-ca-bundle \
  --from-file=ca.crt=/path/to/your/internal-ca.crt
```

Verify:

```bash
kubectl -n rook-ceph get secret rgw-ca-bundle -o jsonpath='{.data.ca\.crt}' | base64 -d
```

## Referencing the CA Bundle in CephObjectStore

Add the `caBundleRef` field to the `gateway` section:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: my-store
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPool:
    replicated:
      size: 3
  gateway:
    port: 80
    securePort: 443
    sslCertificateRef: rgw-tls-secret
    caBundleRef: rgw-ca-bundle
    instances: 2
```

After applying, Rook mounts the CA bundle into the RGW pods and the operator uses it when making health check requests.

## Verifying the CA Bundle is Mounted

Check that the secret is mounted into the RGW pod:

```bash
RGW_POD=$(kubectl -n rook-ceph get pod -l app=rook-ceph-rgw -o name | head -1)
kubectl -n rook-ceph exec $RGW_POD -- ls /var/lib/rook/ceph-client/
```

You should see a `ca.crt` file in the mounted directory.

## Configuring Clients to Trust the CA

For applications accessing the object store from within Kubernetes, inject the same CA certificate into the client pods:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: s3-client
  namespace: my-app
spec:
  volumes:
    - name: ca-bundle
      secret:
        secretName: rgw-ca-bundle
  containers:
    - name: app
      image: amazon/aws-cli
      volumeMounts:
        - name: ca-bundle
          mountPath: /etc/ssl/certs/internal-ca.crt
          subPath: ca.crt
      env:
        - name: AWS_CA_BUNDLE
          value: /etc/ssl/certs/internal-ca.crt
```

## Testing Secure Connectivity

Once CA bundles are in place, test HTTPS connectivity:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  curl --cacert /path/to/ca.crt \
  https://rook-ceph-rgw-my-store.rook-ceph.svc:443/
```

A successful connection returns an RGW XML response without certificate errors.

## Rotating the CA Bundle

To rotate a CA bundle, update the secret content:

```bash
kubectl -n rook-ceph create secret generic rgw-ca-bundle \
  --from-file=ca.crt=/path/to/new-ca.crt \
  --dry-run=client -o yaml | kubectl apply -f -
```

Restart the RGW pods to pick up the new CA:

```bash
kubectl -n rook-ceph rollout restart deployment -l app=rook-ceph-rgw
```

## Summary

CA bundles for Rook object store are stored as Kubernetes secrets with a `ca.crt` key and referenced via the `caBundleRef` field in the `CephObjectStore` gateway spec. Rook injects the CA into RGW pods and uses it for internal health checks. Client pods accessing the HTTPS endpoint must also be configured with the same CA bundle using environment variables like `AWS_CA_BUNDLE`.
