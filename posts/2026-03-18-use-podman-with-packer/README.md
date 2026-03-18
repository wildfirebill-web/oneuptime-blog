# How to Use Podman with Packer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Packer, Container Images, Image Building, DevOps

Description: Learn how to use Podman with Packer to build container images using Packer's provisioning ecosystem, enabling complex image builds with Ansible, shell scripts, and other provisioners.

---

> Packer with Podman lets you build container images using Packer's rich provisioning ecosystem, bringing the same image-building workflow you use for virtual machines to container image creation.

Packer by HashiCorp is a well-established tool for building machine images. While it is most commonly used with cloud providers and virtual machines, Packer also supports building container images. When combined with Podman, you can leverage Packer's extensive provisioner ecosystem including Ansible, shell scripts, Chef, and Puppet to build container images. This approach is particularly valuable when your organization already uses Packer for VM images and wants a consistent workflow for containers.

---

## Installing Packer

Install Packer on your system:

```bash
# On Fedora/RHEL
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://rpm.releases.hashicorp.com/fedora/hashicorp.repo
sudo dnf install packer

# On Ubuntu/Debian
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt-get update && sudo apt-get install packer

# Verify installation
packer version
```

## Configuring Packer with Podman

Packer uses the Docker builder plugin, which can communicate with Podman through its Docker-compatible socket. Enable the Podman socket:

```bash
systemctl --user enable --now podman.socket
export DOCKER_HOST="unix:///run/user/$(id -u)/podman/podman.sock"
```

Install the Docker plugin for Packer:

```bash
packer plugins install github.com/hashicorp/docker
```

## Basic Container Image Build

Create a Packer template to build a container image:

```hcl
# image.pkr.hcl
packer {
  required_plugins {
    docker = {
      version = ">= 1.0.0"
      source  = "github.com/hashicorp/docker"
    }
  }
}

source "docker" "base" {
  image  = "fedora:40"
  commit = true
  changes = [
    "WORKDIR /app",
    "EXPOSE 8080",
    "CMD [\"/app/start.sh\"]"
  ]
}

build {
  sources = ["source.docker.base"]

  provisioner "shell" {
    inline = [
      "dnf update -y",
      "dnf install -y nodejs npm nginx",
      "dnf clean all",
      "mkdir -p /app"
    ]
  }

  provisioner "file" {
    source      = "app/"
    destination = "/app/"
  }

  provisioner "shell" {
    inline = [
      "cd /app && npm ci --production",
      "chmod +x /app/start.sh"
    ]
  }

  post-processor "docker-tag" {
    repository = "myapp"
    tags       = ["latest", "1.0.0"]
  }
}
```

Build the image:

```bash
packer init image.pkr.hcl
packer build image.pkr.hcl
```

## Using Ansible Provisioner

One of Packer's biggest advantages is the Ansible provisioner, which lets you reuse existing Ansible roles for container images:

```hcl
# ansible-image.pkr.hcl
source "docker" "app" {
  image  = "ubuntu:24.04"
  commit = true
  changes = [
    "WORKDIR /app",
    "EXPOSE 8080",
    "ENTRYPOINT [\"/app/entrypoint.sh\"]"
  ]
}

build {
  sources = ["source.docker.app"]

  provisioner "ansible" {
    playbook_file = "ansible/provision.yml"
    extra_arguments = [
      "--extra-vars", "app_version=1.0.0",
      "--extra-vars", "environment=production"
    ]
  }

  post-processor "docker-tag" {
    repository = "myapp"
    tags       = ["latest"]
  }
}
```

The Ansible playbook:

```yaml
# ansible/provision.yml
---
- name: Provision container image
  hosts: all
  become: true
  tasks:
    - name: Install system packages
      ansible.builtin.apt:
        name:
          - python3
          - python3-pip
          - nginx
          - supervisor
        state: present
        update_cache: true

    - name: Create application directory
      ansible.builtin.file:
        path: /app
        state: directory
        mode: '0755'

    - name: Copy application files
      ansible.builtin.copy:
        src: ../app/
        dest: /app/
        mode: '0755'

    - name: Install Python dependencies
      ansible.builtin.pip:
        requirements: /app/requirements.txt
        state: present

    - name: Configure nginx
      ansible.builtin.template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/sites-available/default
        mode: '0644'

    - name: Configure supervisor
      ansible.builtin.template:
        src: templates/supervisor.conf.j2
        dest: /etc/supervisor/conf.d/app.conf
        mode: '0644'

    - name: Clean package cache
      ansible.builtin.apt:
        autoclean: true
        autoremove: true
```

## Multi-Stage Builds with Packer

Create separate build and runtime images:

```hcl
# multi-stage.pkr.hcl
source "docker" "builder" {
  image  = "golang:1.22"
  commit = true
}

source "docker" "runtime" {
  image  = "alpine:3.19"
  commit = true
  changes = [
    "ENTRYPOINT [\"/app/server\"]",
    "EXPOSE 8080"
  ]
}

build {
  name = "builder"
  sources = ["source.docker.builder"]

  provisioner "file" {
    source      = "src/"
    destination = "/src/"
  }

  provisioner "shell" {
    inline = [
      "cd /src",
      "go mod download",
      "CGO_ENABLED=0 go build -o /output/server ./cmd/server"
    ]
  }

  post-processor "docker-tag" {
    repository = "myapp-builder"
    tags       = ["latest"]
  }
}

build {
  name = "runtime"
  sources = ["source.docker.runtime"]

  provisioner "shell" {
    inline = [
      "apk add --no-cache ca-certificates tzdata",
      "mkdir -p /app"
    ]
  }

  provisioner "shell" {
    inline = [
      "# Copy binary from builder container",
      "BUILDER_ID=$(podman create myapp-builder:latest)",
      "podman cp $BUILDER_ID:/output/server /tmp/server",
      "podman rm $BUILDER_ID"
    ]
  }

  provisioner "file" {
    source      = "/tmp/server"
    destination = "/app/server"
  }

  post-processor "docker-tag" {
    repository = "myapp"
    tags       = ["latest"]
  }
}
```

## Using Variables and Environment-Specific Builds

Parameterize your builds for different environments:

```hcl
# variables.pkr.hcl
variable "base_image" {
  type    = string
  default = "fedora:40"
}

variable "app_version" {
  type    = string
  default = "latest"
}

variable "registry" {
  type    = string
  default = "registry.example.com"
}

variable "environment" {
  type    = string
  default = "production"
}
```

```hcl
# parameterized.pkr.hcl
source "docker" "app" {
  image  = var.base_image
  commit = true
  changes = [
    "ENV APP_VERSION=${var.app_version}",
    "ENV ENVIRONMENT=${var.environment}",
    "EXPOSE 8080",
    "CMD [\"/app/start.sh\"]"
  ]
}

build {
  sources = ["source.docker.app"]

  provisioner "shell" {
    environment_vars = [
      "APP_VERSION=${var.app_version}",
      "ENVIRONMENT=${var.environment}"
    ]
    scripts = [
      "scripts/install-deps.sh",
      "scripts/configure-app.sh",
      "scripts/cleanup.sh"
    ]
  }

  post-processor "docker-tag" {
    repository = "${var.registry}/myapp"
    tags       = [var.app_version, "latest"]
  }
}
```

Build with variables:

```bash
packer build \
  -var "app_version=2.1.0" \
  -var "environment=staging" \
  -var "registry=registry.example.com" \
  parameterized.pkr.hcl
```

## Pushing to a Registry

Push built images to a container registry:

```hcl
# registry-push.pkr.hcl
build {
  sources = ["source.docker.app"]

  provisioner "shell" {
    scripts = ["scripts/build.sh"]
  }

  post-processor "docker-tag" {
    repository = "registry.example.com/myapp"
    tags       = [var.app_version]
  }

  post-processor "docker-push" {
    login          = true
    login_server   = "registry.example.com"
    login_username = var.registry_username
    login_password = var.registry_password
  }
}
```

## CI/CD Integration

Use Packer in a CI pipeline to build images with Podman:

```yaml
# .github/workflows/build-image.yml
name: Build Container Image
on:
  push:
    tags: ['v*']

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Podman and Packer
        run: |
          sudo apt-get update
          sudo apt-get install -y podman
          wget https://releases.hashicorp.com/packer/1.10.0/packer_1.10.0_linux_amd64.zip
          unzip packer_1.10.0_linux_amd64.zip
          sudo mv packer /usr/local/bin/

      - name: Enable Podman socket
        run: |
          systemctl --user enable --now podman.socket

      - name: Build image
        env:
          DOCKER_HOST: "unix:///run/user/$(id -u)/podman/podman.sock"
        run: |
          packer init .
          packer build \
            -var "app_version=${GITHUB_REF_NAME}" \
            image.pkr.hcl
```

## Conclusion

Packer with Podman brings the power of Packer's provisioning ecosystem to container image building. The Ansible provisioner is particularly valuable, letting you reuse existing configuration management code for container images. Variables and post-processors give you the flexibility to build environment-specific images and push them to registries automatically. For organizations already using Packer for virtual machine images, extending it to container images with Podman creates a consistent image-building workflow across all infrastructure types.
