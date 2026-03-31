# How to Use Dapr with HashiCorp Vault Auth Methods

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, HashiCorp Vault, Authentication, Secret, Security

Description: Configure Dapr's HashiCorp Vault secret store component using Vault auth methods including Kubernetes auth, token auth, and AppRole for secure secret access.

---

## Dapr and HashiCorp Vault

Dapr's Vault secret store component allows applications to read secrets from HashiCorp Vault without embedding Vault credentials in code. Dapr handles authentication with Vault using configurable auth methods and makes secrets available through the Dapr secrets API.

## Configuring Kubernetes Auth Method (Recommended)

The Kubernetes auth method is the most secure approach for Kubernetes-hosted workloads:

```bash
# Enable Kubernetes auth in Vault
vault auth enable kubernetes

# Configure Kubernetes auth
vault write auth/kubernetes/config \
  kubernetes_host="https://$(kubectl get endpoint kubernetes -o jsonpath='{.subsets[0].addresses[0].ip}'):443" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

# Create a policy for Dapr app
vault policy write dapr-app-policy - <<EOF
path "secret/data/myapp/*" {
  capabilities = ["read"]
}
EOF

# Create Kubernetes auth role
vault write auth/kubernetes/role/dapr-app-role \
  bound_service_account_names=my-app \
  bound_service_account_namespaces=production \
  policies=dapr-app-policy \
  ttl=1h
```

Configure the Dapr component to use Kubernetes auth:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: vault-secrets
spec:
  type: secretstores.hashicorp.vault
  version: v1
  metadata:
    - name: vaultAddr
      value: "http://vault.vault-system.svc.cluster.local:8200"
    - name: skipVerify
      value: "false"
    - name: tlsServerName
      value: "vault.vault-system.svc.cluster.local"
    - name: vaultKVPrefix
      value: "myapp"
    - name: auth
      value: "kubernetes"
    - name: vaultTokenMountPath
      value: "/var/run/secrets/kubernetes.io/serviceaccount/token"
    - name: vaultKubernetesRole
      value: "dapr-app-role"
```

## Configuring AppRole Auth Method

AppRole is appropriate for non-Kubernetes workloads:

```bash
# Enable AppRole
vault auth enable approle

# Create AppRole
vault write auth/approle/role/dapr-role \
  token_policies=dapr-app-policy \
  token_ttl=1h

# Get RoleID and SecretID
vault read auth/approle/role/dapr-role/role-id
vault write -f auth/approle/role/dapr-role/secret-id
```

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: vault-secrets
spec:
  type: secretstores.hashicorp.vault
  version: v1
  metadata:
    - name: vaultAddr
      value: "https://vault.example.com:8200"
    - name: vaultRoleID
      secretKeyRef:
        name: vault-approle
        key: roleId
    - name: vaultRoleSecretID
      secretKeyRef:
        name: vault-approle
        key: secretId
```

## Accessing Secrets in Your Application

```python
from dapr.clients import DaprClient

with DaprClient() as client:
    secret = client.get_secret(
        store_name="vault-secrets",
        key="database-password"
    )
    db_password = secret.secret["database-password"]
    print(f"Retrieved secret: {db_password[:3]}***")
```

## Verifying Vault Integration

```bash
# Test secret retrieval via Dapr API
curl http://localhost:3500/v1.0/secrets/vault-secrets/database-password
```

## Summary

Dapr integrates with HashiCorp Vault through multiple auth methods. Use Kubernetes auth on Kubernetes clusters for credential-free authentication - Dapr presents the pod's service account token to Vault for verification. Use AppRole for non-Kubernetes environments with RoleID and SecretID stored as Kubernetes secrets. Access Vault secrets in your app through the Dapr secrets API without any Vault SDK dependencies.
