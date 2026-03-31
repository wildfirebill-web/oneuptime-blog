# How to Use Dapr with Confidential Computing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Confidential Computing, Security, TEE, Attestation, Azure

Description: Learn how to run Dapr services inside confidential computing enclaves using Trusted Execution Environments to protect secrets and state data from privileged access.

---

## What Is Confidential Computing?

Confidential computing protects data in use by running code inside a Trusted Execution Environment (TEE) - hardware-isolated memory regions that even the cloud provider or host OS cannot read. Technologies include:
- Intel SGX (Software Guard Extensions)
- AMD SEV (Secure Encrypted Virtualization)
- ARM TrustZone

Dapr's secret management and state APIs become more secure when the workloads using them run inside TEEs.

## Why Combine Dapr with Confidential Computing

Dapr handles secret retrieval from Vault or Kubernetes secrets. In a standard deployment, the host OS or a privileged admin could potentially access in-memory secrets. Inside a TEE:
- Secrets loaded via Dapr API are encrypted in TEE memory
- The state written to Redis via Dapr is encrypted before leaving the enclave
- Remote attestation proves to external parties that the code is genuine

## Running Dapr on Azure Confidential Containers

Azure Confidential Containers (CoCo) use AMD SEV-SNP for hardware isolation:

```bash
# Create an AKS cluster with confidential node pool
az aks create \
  --resource-group myRG \
  --name myAKS \
  --node-vm-size Standard_DC4as_cc_v5 \
  --node-count 3 \
  --enable-oidc-issuer \
  --enable-workload-identity

# Install Dapr on the confidential cluster
dapr init -k
```

## Configuring Dapr with Confidential Secret Store

Use Azure Key Vault with workload identity for secret access inside the TEE:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: azurekeyvault
spec:
  type: secretstores.azure.keyvault
  version: v1
  metadata:
    - name: vaultName
      value: my-confidential-vault
    - name: azureClientId
      value: "{{ workload-identity-client-id }}"
```

Workload identity ensures the secret store is only accessible from attested pods.

## Enabling State Encryption at Rest

Configure Dapr to encrypt state before writing to the backend:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.redis
  version: v1
  metadata:
    - name: redisHost
      value: redis:6379
    - name: primaryEncryptionKey
      secretKeyRef:
        name: azurekeyvault
        key: state-encryption-key
    - name: encryptionKeyName
      value: state-encryption-key
```

Data written via `client.SaveState()` is encrypted by the Dapr sidecar before transmission to Redis.

## Remote Attestation Verification

Verify that the Dapr workload is running in a genuine TEE before trusting its requests:

```bash
# For AMD SEV-SNP on Azure
az confcom katapolicygen \
  --yaml-input deployment.yaml \
  --print-policy

# Verify attestation report
az confcom attestation verify \
  --attestation-endpoint https://sharedeus2.eus2.attest.azure.net
```

## mTLS Between Confidential Services

Dapr's built-in mTLS using the Sentry service adds another layer on top of TEE isolation:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: secure-config
spec:
  mtls:
    enabled: true
    workloadCertTTL: 1h     # Short-lived certs inside the TEE
    allowedClockSkew: 15m
```

Short certificate TTLs reduce the impact of key compromise.

## Audit Logging

Log all Dapr secret access for compliance:

```bash
# Enable audit logging via Dapr middleware
kubectl logs -l app=order-service -c daprd \
  | grep "secret\|GetSecret" \
  | jq '{time: .time, app: .app_id, secret: .name}'
```

## Summary

Combining Dapr with confidential computing runs secret retrieval and state operations inside hardware-isolated TEE memory, protecting data from privileged host access. Azure Confidential Containers with AMD SEV-SNP, Dapr workload identity for Key Vault access, state encryption at rest, and short-lived mTLS certificates create a layered security architecture suitable for regulated workloads processing sensitive data.
