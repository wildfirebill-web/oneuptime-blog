# How to Use the indent Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the indent function in OpenTofu to add consistent indentation to multi-line strings for embedding them in YAML or configuration templates.

## Introduction

The `indent` function in OpenTofu adds a specified number of spaces to the beginning of each line in a multi-line string (except the first line). This is particularly useful when embedding multi-line content inside YAML, JSON templates, or other indentation-sensitive formats using `templatefile`.

## Syntax

```hcl
indent(num_spaces, string)
```

- **num_spaces** — the number of spaces to prepend to each line (except the first)
- **string** — the multi-line string to indent
- Returns the string with all lines after the first indented by the given amount

## Basic Examples

```hcl
output "basic_indent" {
  value = indent(4, "line1\nline2\nline3")
  # Returns:
  # "line1\n    line2\n    line3"
}

output "yaml_indent" {
  value = "key: |\n  ${indent(2, "line1\nline2\nline3")}"
  # Returns:
  # "key: |\n  line1\n  line2\n  line3"
}
```

Note: The first line is NOT indented. This is intentional — it allows you to position the first line inline in a template.

## Practical Use Cases

### Embedding Scripts in YAML Templates

```hcl
# templates/cloud-init.yaml.tpl
# user_data:
#   - path: /etc/setup.sh
#     content: |
#       ${indent(6, setup_script)}

variable "setup_script" {
  type    = string
  default = "#!/bin/bash\necho 'Setting up...'\napt-get update\napt-get install -y nginx"
}

locals {
  cloud_init = templatefile("${path.module}/templates/cloud-init.yaml.tpl", {
    setup_script = var.setup_script
  })
}
```

### Embedding JSON Policy in YAML

```hcl
locals {
  policy_json = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:GetObject"]
      Resource = "*"
    }]
  })

  # Embed the JSON inside a YAML block with proper indentation
  yaml_with_policy = <<-EOT
    config:
      policy: |
        ${indent(8, local.policy_json)}
  EOT
}
```

### Kubernetes ConfigMap with Indented Scripts

```hcl
locals {
  init_script = <<-SCRIPT
    #!/bin/bash
    set -e
    echo "Initializing..."
    for i in 1 2 3; do
      echo "Step $i"
    done
  SCRIPT

  configmap_yaml = <<-YAML
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: app-scripts
    data:
      init.sh: |
        ${indent(8, trimspace(local.init_script))}
  YAML
}

resource "kubernetes_manifest" "scripts" {
  manifest = yamldecode(local.configmap_yaml)
}
```

### Multi-Line UserData in AWS

```hcl
locals {
  bootstrap_commands = join("\n", [
    "apt-get update -y",
    "apt-get install -y docker.io",
    "systemctl start docker",
    "systemctl enable docker"
  ])

  user_data = base64encode(<<-HEREDOC
    #!/bin/bash
    # Bootstrap script
    ${indent(0, local.bootstrap_commands)}
  HEREDOC
  )
}
```

## Step-by-Step Usage

1. Prepare the multi-line string you want to embed.
2. Decide the indentation level (typically matching the YAML/template structure).
3. In your template, use `${indent(n, content)}` where the first line aligns with its position.
4. Test in `tofu console`:

```bash
tofu console

> indent(4, "a\nb\nc")
"a\n    b\n    c"
```

## Why the First Line is Not Indented

The design allows this pattern in templates:

```
key: |
  ${indent(2, multi_line_value)}
```

The `${...}` is already at column 2. The first line of `multi_line_value` goes there inline. All subsequent lines get the additional `2` spaces to match.

## Combining with trimspace

```hcl
locals {
  raw = <<-EOF
    line one
    line two
    line three
  EOF

  # trimspace first, then indent for embedding
  indented = indent(4, trimspace(local.raw))
}
```

## Conclusion

The `indent` function is a precise tool for multi-line string embedding in OpenTofu. It solves the common problem of keeping YAML, configuration files, and templates properly formatted when they contain embedded multi-line content. Pair it with `templatefile`, `trimspace`, and heredoc strings for clean, maintainable infrastructure templates.
