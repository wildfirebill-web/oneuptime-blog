# How to Create Hetzner Cloud SSH Keys with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Hetzner Cloud, SSH Keys, Security, Infrastructure as Code

Description: Learn how to manage Hetzner Cloud SSH keys with OpenTofu to enable secure, password-free server access from the start.

SSH keys in Hetzner Cloud are project-level resources that can be referenced when creating servers to enable key-based authentication from first boot. OpenTofu lets you upload and manage these keys as code alongside your server definitions.

## Uploading an SSH Key

```hcl
resource "hcloud_ssh_key" "default" {
  name       = "opentofu-default"
  public_key = file("~/.ssh/id_ed25519.pub")  # Path to your public key file

  labels = {
    managed_by = "opentofu"
    purpose    = "admin-access"
  }
}
```

## Using a Variable for the Public Key

Prefer passing the key content as a variable for CI/CD pipelines:

```hcl
variable "ssh_public_key" {
  description = "Public SSH key for server access"
  type        = string
}

resource "hcloud_ssh_key" "ci" {
  name       = "ci-runner"
  public_key = var.ssh_public_key
}
```

Set the key in CI:

```bash
export TF_VAR_ssh_public_key="ssh-ed25519 AAAA... ci@pipeline"
tofu apply
```

## Managing Multiple SSH Keys

```hcl
variable "ssh_keys" {
  type = map(string)
  description = "Map of key name to public key content"
  default = {}
}

resource "hcloud_ssh_key" "team" {
  for_each   = var.ssh_keys
  name       = each.key
  public_key = each.value
}
```

```hcl
# team.tfvars
ssh_keys = {
  "alice" = "ssh-ed25519 AAAA...alice"
  "bob"   = "ssh-ed25519 AAAA...bob"
  "ci"    = "ssh-ed25519 AAAA...ci"
}
```

## Attaching SSH Keys to Servers

```hcl
resource "hcloud_server" "web" {
  name        = "web-01"
  image       = "ubuntu-24.04"
  server_type = "cx22"
  location    = "nbg1"

  # Reference all team keys — any matching key can access the server
  ssh_keys = [for key in hcloud_ssh_key.team : key.id]
}
```

Or reference specific keys:

```hcl
resource "hcloud_server" "production" {
  name        = "prod-01"
  image       = "ubuntu-24.04"
  server_type = "cx42"
  location    = "nbg1"

  # Only the CI key for production servers
  ssh_keys = [hcloud_ssh_key.ci.id]
}
```

## Looking Up Existing Keys

Reference SSH keys that were created outside of OpenTofu:

```hcl
data "hcloud_ssh_key" "existing" {
  name = "my-laptop"
}

resource "hcloud_server" "dev" {
  name        = "dev-01"
  image       = "ubuntu-24.04"
  server_type = "cx22"
  location    = "nbg1"
  ssh_keys    = [data.hcloud_ssh_key.existing.id]
}
```

## Getting Key Fingerprint as Output

```hcl
output "ci_key_fingerprint" {
  value = hcloud_ssh_key.ci.fingerprint
}
```

## Conclusion

Hetzner Cloud SSH keys are simple project-level resources. Upload them with OpenTofu using `hcloud_ssh_key`, reference them in server resources by ID, and use `for_each` with a variable map to manage team keys declaratively. Keys added after server creation can be injected via cloud-init `user_data` or configuration management tools.
