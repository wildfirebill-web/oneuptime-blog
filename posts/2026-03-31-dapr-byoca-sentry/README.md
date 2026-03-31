# How to Bring Your Own CA Certificate for Dapr Sentry

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Sentry, Certificate, CA, Security

Description: Configure Dapr Sentry to use your own CA certificate instead of the auto-generated one, enabling integration with enterprise PKI and existing certificate infrastructure.

---

## Why Use a Custom CA?

By default, Dapr Sentry generates a self-signed root CA at startup. In enterprise environments, you may need to use your organization's existing PKI infrastructure so that workload certificates chain to a trusted enterprise root. This also allows certificate validation by external systems.

## Generating a CA Certificate

If you don't have an existing CA, generate one with the Dapr CLI:

```bash
# Install step CLI for certificate generation
brew install step

# Generate root CA
step certificate create \
  "Dapr Root CA" \
  ca.crt ca.key \
  --profile root-ca \
  --no-password \
  --insecure \
  --not-after 87600h  # 10 years
```

Or use openssl:

```bash
openssl genrsa -out ca.key 4096
openssl req -new -x509 -days 3650 \
  -key ca.key \
  -out ca.crt \
  -subj "/C=US/O=MyOrg/CN=Dapr Root CA"
```

## Creating the Kubernetes Secret

Store the CA certificate and key as a Kubernetes secret:

```bash
kubectl create secret generic dapr-trust-bundle \
  --from-file=ca.crt=./ca.crt \
  --from-file=issuer.crt=./issuer.crt \
  --from-file=issuer.key=./issuer.key \
  -n dapr-system
```

## Generating an Issuer Certificate

The issuer certificate is an intermediate cert signed by your root CA:

```bash
# Generate issuer key and CSR
step certificate create \
  "Dapr Issuer" \
  issuer.crt issuer.key \
  --profile intermediate-ca \
  --ca ca.crt \
  --ca-key ca.key \
  --no-password \
  --insecure \
  --not-after 8760h  # 1 year
```

## Configuring Sentry to Use the Custom CA

Reference the custom trust bundle in your Helm values:

```yaml
dapr_sentry:
  trustAnchorsFile: ""  # Empty to use secret
  issuerCertFile: ""
  issuerKeyFile: ""
```

Or provide the files directly in Helm:

```bash
helm upgrade dapr dapr/dapr \
  --namespace dapr-system \
  --set-file dapr_sentry.trustAnchors=./ca.crt \
  --set-file dapr_sentry.issuerCertificate=./issuer.crt \
  --set-file dapr_sentry.issuerKey=./issuer.key \
  --reuse-values
```

## Verifying the Custom CA Is in Use

After restarting Sentry, verify it is using your custom CA:

```bash
kubectl get secret dapr-trust-bundle -n dapr-system \
  -o jsonpath='{.data.ca\.crt}' | base64 -d | \
  openssl x509 -subject -issuer -noout
```

The output should show your custom CA details, not a Dapr auto-generated name.

## Summary

Using a custom CA with Dapr Sentry allows integration with enterprise PKI. Generate or provide root CA and issuer certificates, store them as a Kubernetes secret, and reference them in your Helm deployment. Verify Sentry is using your CA by inspecting the trust bundle secret after the restart.
