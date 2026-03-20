# How OpenTofu Credentials Helpers Work

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Credentials Helper, Authentication, Security, Infrastructure as Code

Description: Learn how OpenTofu credentials helpers work — storing and retrieving registry tokens via external programs, enabling integration with system keychains and secrets managers.

## Introduction

OpenTofu credentials helpers are external programs that provide authentication tokens for private registries and backends. Instead of storing tokens in plaintext in `~/.terraformrc`, a credentials helper retrieves tokens from a secure store (macOS Keychain, system secret manager, or custom API) when needed.

## How Credentials Helpers Work

```
tofu init (needs token for private-registry.example.com)
  ↓
Check ~/.terraformrc for credentials block
  ↓
If credentials_helper is configured:
  Execute: terraform-credentials-HELPER-NAME get private-registry.example.com
  ↓
Helper returns JSON: {"token": "actual-token-value"}
  ↓
OpenTofu uses token for registry authentication
```

## Static Credentials (No Helper)

The simplest approach — token stored in plaintext:

```hcl
# ~/.terraformrc
credentials "registry.mycompany.com" {
  token = "my-registry-token"
}
```

Not recommended for security-sensitive environments.

## Credentials Helper Configuration

```hcl
# ~/.terraformrc
credentials_helper "myhelper" {
  args = ["--config", "/etc/credentials-helper.conf"]
}
```

The helper binary must be named `terraform-credentials-myhelper` and be in `$PATH`.

## Writing a Custom Credentials Helper

```go
// cmd/terraform-credentials-myvault/main.go
package main

import (
    "encoding/json"
    "fmt"
    "os"

    vault "github.com/hashicorp/vault/api"
)

type CredentialsResponse struct {
    Token string `json:"token"`
}

func main() {
    if len(os.Args) < 2 {
        fmt.Fprintf(os.Stderr, "Usage: terraform-credentials-myvault <get|store|forget> [hostname]\n")
        os.Exit(1)
    }

    command := os.Args[1]
    hostname := ""
    if len(os.Args) > 2 {
        hostname = os.Args[2]
    }

    switch command {
    case "get":
        token := getTokenFromVault(hostname)
        resp := CredentialsResponse{Token: token}
        json.NewEncoder(os.Stdout).Encode(resp)

    case "store":
        // Read credentials from stdin (for tofu login)
        var creds map[string]interface{}
        json.NewDecoder(os.Stdin).Decode(&creds)
        storeTokenInVault(hostname, creds["token"].(string))

    case "forget":
        removeTokenFromVault(hostname)
    }
}

func getTokenFromVault(hostname string) string {
    client, _ := vault.NewClient(vault.DefaultConfig())
    secret, err := client.Logical().Read(fmt.Sprintf("secret/registry-tokens/%s", hostname))
    if err != nil || secret == nil {
        return ""
    }
    return secret.Data["token"].(string)
}
```

Build and install:

```bash
go build -o terraform-credentials-myvault ./cmd/terraform-credentials-myvault/
sudo mv terraform-credentials-myvault /usr/local/bin/

# Configure in .terraformrc
cat >> ~/.terraformrc << 'EOF'
credentials_helper "myvault" {}
EOF
```

## macOS Keychain Helper

```bash
#!/bin/bash
# terraform-credentials-keychain — use macOS Keychain for tokens

COMMAND="$1"
HOSTNAME="$2"

case "$COMMAND" in
  get)
    TOKEN=$(security find-internet-password \
      -s "$HOSTNAME" \
      -a "opentofu" \
      -w 2>/dev/null)
    if [ -n "$TOKEN" ]; then
      echo "{\"token\": \"$TOKEN\"}"
    else
      echo "{}"
    fi
    ;;

  store)
    TOKEN=$(cat /dev/stdin | python3 -c "import sys,json; print(json.load(sys.stdin).get('token',''))")
    security add-internet-password \
      -s "$HOSTNAME" \
      -a "opentofu" \
      -w "$TOKEN" \
      -U
    ;;

  forget)
    security delete-internet-password \
      -s "$HOSTNAME" \
      -a "opentofu" 2>/dev/null || true
    ;;
esac
```

```bash
chmod +x terraform-credentials-keychain
sudo mv terraform-credentials-keychain /usr/local/bin/

# Add to .terraformrc
echo 'credentials_helper "keychain" {}' >> ~/.terraformrc
```

## CI/CD: Environment Variables Instead of Helpers

For CI/CD, use environment variables — no credentials helper needed:

```bash
# GitHub Actions: pass token as environment variable
- name: OpenTofu Init
  run: tofu init
  env:
    TF_TOKEN_registry_mycompany_com: ${{ secrets.REGISTRY_TOKEN }}
    # Variable name format: TF_TOKEN_<hostname with dots replaced by underscores>
```

```bash
# The TF_TOKEN_ prefix tells OpenTofu to use the value as the token
# for that hostname — no .terraformrc needed
export TF_TOKEN_registry_mycompany_com="my-token-value"
tofu init  # Uses the token automatically
```

## Conclusion

OpenTofu credentials helpers let you store registry tokens in secure external stores (system keychains, Vault, secrets managers) instead of plaintext configuration files. Implement a helper by creating a binary named `terraform-credentials-HELPERNAME` that handles `get`, `store`, and `forget` subcommands, then reference it in `~/.terraformrc`. For CI/CD environments, use `TF_TOKEN_hostname` environment variables — they take precedence over all credentials configuration and require no file-based configuration.
