# How to Configure a Credentials Helper in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Credential, Authentication, Registry, Infrastructure as Code, Security

Description: Learn how to configure a credentials helper in OpenTofu to authenticate with private module registries and OCI artifact stores.

---

OpenTofu supports credentials helpers - external executables that supply authentication tokens for private module registries and other services. This avoids storing credentials directly in configuration files.

---

## Built-in Credentials in .terraformrc / .tofurc

```hcl
# ~/.tofurc

credentials "registry.example.com" {
  token = "my-private-token"
}
```

---

## Configure a Credentials Helper

```hcl
# ~/.tofurc
credentials_helper "my-helper" {
  args = ["--config", "/etc/my-helper/config.json"]
}
```

The helper binary must be named `terraform-credentials-<name>` or `tofu-credentials-<name>` and be on the `PATH`.

---

## Example Credentials Helper Script

```bash
#!/bin/bash
# /usr/local/bin/tofu-credentials-vault
# Fetches credentials from HashiCorp Vault

ACTION=$1  # "get", "store", or "forget"
HOST=$(cat /dev/stdin | jq -r '.url // empty' 2>/dev/null)

case "$ACTION" in
  get)
    TOKEN=$(vault kv get -field=token secret/tofu/${HOST})
    echo "{"token": "${TOKEN}"}"
    ;;
  store|forget)
    # Read-only helper - do nothing
    ;;
esac
```

Make it executable:
```bash
chmod +x /usr/local/bin/tofu-credentials-vault
```

---

## Reference a Private Module Registry

```hcl
module "network" {
  source  = "registry.example.com/myorg/network/aws"
  version = "~> 2.0"
}
```

OpenTofu calls the credentials helper to get the token for `registry.example.com` before fetching the module.

---

## Use AWS CodeArtifact as a Credentials Source

```bash
# Generate an auth token
TOKEN=$(aws codeartifact get-authorization-token   --domain mydomain   --domain-owner 123456789012   --query authorizationToken   --output text)

# Write to .tofurc temporarily
cat > ~/.tofurc <<EOF
credentials "mydomain-123456789012.d.codeartifact.us-east-1.amazonaws.com" {
  token = "${TOKEN}"
}
EOF
```

---

## Summary

Configure static credentials in `~/.tofurc` with `credentials` blocks, or delegate to an external binary via `credentials_helper`. Helpers must implement `get`, `store`, and `forget` subcommands and output JSON. This pattern enables integration with secret managers like HashiCorp Vault or AWS Secrets Manager, keeping tokens out of configuration files.
