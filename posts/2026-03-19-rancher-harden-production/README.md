# How to Harden Rancher for Production

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Security

Description: Learn how to apply security hardening best practices to your Rancher deployment for production environments.

Running Rancher in production requires more than a default installation. Security hardening reduces your attack surface and protects your Kubernetes infrastructure from common threats. This guide covers the essential hardening steps for a production Rancher deployment.

## Prerequisites

- Rancher v2.5 or later
- kubectl and Helm 3 access
- Admin privileges on the Rancher management cluster
- A valid TLS certificate for the Rancher hostname

## Step 1: Use a Hardened Kubernetes Distribution

Deploy Rancher on a hardened Kubernetes distribution such as RKE2, which is designed for security-focused environments. RKE2 ships with CIS benchmark compliance by default.

Install RKE2 with CIS hardening profile:

```bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE=server sh -

cat > /etc/rancher/rke2/config.yaml << 'EOF'
profile: cis-1.6
selinux: true
secrets-encryption: true
EOF

systemctl enable rke2-server
systemctl start rke2-server
```

## Step 2: Configure TLS with a Trusted Certificate

Never use self-signed certificates in production. Use a certificate from a trusted CA or Let's Encrypt:

```bash
helm install rancher rancher-latest/rancher \
  -n cattle-system \
  --create-namespace \
  --set hostname=rancher.yourdomain.com \
  --set ingress.tls.source=letsEncrypt \
  --set letsEncrypt.email=admin@yourdomain.com \
  --set letsEncrypt.ingress.class=nginx \
  --set replicas=3
```

For a custom certificate:

```bash
kubectl create secret tls tls-rancher-ingress \
  -n cattle-system \
  --cert=tls.crt \
  --key=tls.key

helm install rancher rancher-latest/rancher \
  -n cattle-system \
  --create-namespace \
  --set hostname=rancher.yourdomain.com \
  --set ingress.tls.source=secret \
  --set replicas=3
```

## Step 3: Restrict Network Access

Limit who can reach the Rancher UI and API by configuring network policies and firewall rules.

Create a NetworkPolicy to restrict access to the Rancher namespace:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-rancher-access
  namespace: cattle-system
spec:
  podSelector:
    matchLabels:
      app: rancher
  policyTypes:
  - Ingress
  ingress:
  - from:
    - ipBlock:
        cidr: 10.0.0.0/8
    - ipBlock:
        cidr: 172.16.0.0/12
    ports:
    - protocol: TCP
      port: 443
```

## Step 4: Enable Audit Logging

Enable Kubernetes API audit logging to track all API requests:

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
metadata:
  name: rancher-audit-policy
rules:
- level: Metadata
  resources:
  - group: ""
    resources: ["secrets", "configmaps"]
- level: RequestResponse
  resources:
  - group: ""
    resources: ["pods", "services"]
- level: Request
  verbs: ["create", "update", "patch", "delete"]
  resources:
  - group: "management.cattle.io"
```

Configure Rancher's built-in audit log:

```bash
helm upgrade rancher rancher-latest/rancher \
  -n cattle-system \
  --set auditLog.level=2 \
  --set auditLog.destination=hostPath \
  --set auditLog.hostPath=/var/log/rancher/audit
```

## Step 5: Configure RBAC with Least Privilege

Avoid using the default admin account for daily operations. Create role-based access:

1. In Rancher, go to **Users & Authentication** > **Users**.
2. Create individual user accounts.
3. Go to **Cluster Management** and assign users to specific clusters with appropriate roles:
   - **Cluster Owner**: Full control of a single cluster.
   - **Cluster Member**: Deploy workloads but cannot manage cluster settings.
   - **Read-Only**: View-only access.

Remove the default admin password and use an external authentication provider:

```bash
helm upgrade rancher rancher-latest/rancher \
  -n cattle-system \
  --set 'extraEnv[0].name=CATTLE_RESTRICT_DEFAULT_ADMIN' \
  --set 'extraEnv[0].value=true'
```

## Step 6: Enable External Authentication

Configure an external identity provider instead of local accounts:

1. Go to **Users & Authentication** > **Auth Provider**.
2. Select your provider (LDAP, Active Directory, SAML, GitHub, Google OAuth, etc.).
3. Configure the connection details.
4. Test the configuration.
5. Enable the provider.

For LDAP configuration via Helm:

```bash
helm upgrade rancher rancher-latest/rancher \
  -n cattle-system \
  --set 'extraEnv[0].name=CATTLE_AUTH_PROVIDER' \
  --set 'extraEnv[0].value=activedirectory'
```

## Step 7: Restrict the Rancher API

Limit API access using API keys with scoped permissions:

1. Go to **API & Keys** in the user menu.
2. Create API keys with specific scopes and expiration times.
3. Never use admin-level API keys in automation; use scoped keys instead.

Set API key expiration:

```bash
helm upgrade rancher rancher-latest/rancher \
  -n cattle-system \
  --set 'extraEnv[0].name=CATTLE_TOKEN_MAX_TTL_MINUTES' \
  --set 'extraEnv[0].value=1440'
```

## Step 8: Secure etcd

Ensure etcd communication is encrypted and access is restricted:

```yaml
# RKE2 config
etcd-arg:
  - "client-cert-auth=true"
  - "peer-client-cert-auth=true"
```

Enable etcd encryption at rest:

```bash
# Already enabled with secrets-encryption: true in RKE2 config
```

## Step 9: Run Security Scans

Use the CIS Benchmark scanning feature in Rancher:

1. Navigate to the cluster.
2. Go to **CIS Benchmark** in the cluster menu.
3. Run a scan to identify security issues.
4. Review and remediate findings.

## Step 10: Keep Rancher Updated

Stay current with security patches:

```bash
helm repo update
helm search repo rancher-latest/rancher --versions | head -5
```

Subscribe to Rancher security advisories and apply updates promptly.

## Security Hardening Checklist

- [ ] Use RKE2 or a hardened Kubernetes distribution
- [ ] Deploy with trusted TLS certificates
- [ ] Enable audit logging
- [ ] Configure RBAC with least privilege
- [ ] Use external authentication (LDAP/SAML)
- [ ] Restrict network access to Rancher
- [ ] Enable etcd encryption
- [ ] Run CIS benchmark scans
- [ ] Set API key expiration
- [ ] Keep Rancher and Kubernetes up to date
- [ ] Enable Pod Security Standards or Policies
- [ ] Configure backup encryption

## Conclusion

Hardening Rancher for production is an ongoing process that involves multiple layers of security. By following these steps, you significantly reduce the attack surface of your Rancher management server and the clusters it manages. Combine these hardening measures with regular security scanning and prompt patching to maintain a strong security posture.
