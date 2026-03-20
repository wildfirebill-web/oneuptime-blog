# How to Use asdf to Manage Multiple OpenTofu Versions

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, asdf, Version Management, Infrastructure as Code, DevOps

Description: A guide to using asdf as a universal version manager to install and switch between multiple OpenTofu versions.

## Introduction

asdf is a universal version manager that supports dozens of programming languages and tools through plugins. Using asdf for OpenTofu version management is ideal when you already use asdf for other tools like Node.js, Python, or Ruby, allowing you to manage all versions in one place.

## Installing asdf

```bash
# Clone asdf
git clone https://github.com/asdf-vm/asdf.git ~/.asdf --branch v0.14.0

# Add to shell (bash)
echo '. "$HOME/.asdf/asdf.sh"' >> ~/.bashrc
echo '. "$HOME/.asdf/completions/asdf.bash"' >> ~/.bashrc
source ~/.bashrc

# For Zsh
echo '. "$HOME/.asdf/asdf.sh"' >> ~/.zshrc
source ~/.zshrc

# Via Homebrew (macOS)
brew install asdf
echo '. $(brew --prefix asdf)/libexec/asdf.sh' >> ~/.zshrc
```

## Installing the OpenTofu Plugin

```bash
# Add the OpenTofu plugin
asdf plugin add opentofu https://github.com/virtualroot/asdf-opentofu.git

# Verify plugin was added
asdf plugin list
```

## Installing OpenTofu Versions

```bash
# List all available versions
asdf list all opentofu

# Filter for specific version
asdf list all opentofu | grep "^1.9"

# Install the latest version
asdf install opentofu latest

# Install a specific version
asdf install opentofu 1.9.0
asdf install opentofu 1.8.5

# List installed versions
asdf list opentofu
```

## Setting the OpenTofu Version

```bash
# Set the global default version
asdf global opentofu 1.9.0

# Set the local version for a specific project
cd /path/to/your/project
asdf local opentofu 1.8.5

# This creates a .tool-versions file
cat .tool-versions
# opentofu 1.8.5

# Verify active version
tofu version
```

## The .tool-versions File

```bash
# Example .tool-versions file with multiple tools
cat .tool-versions
```

```
nodejs 20.11.0
python 3.12.0
opentofu 1.9.0
terraform 1.5.7
```

```bash
# Install all versions specified in .tool-versions
asdf install

# All team members can run this to get the right versions
```

## Managing Multiple Tools Together

```bash
# A typical .tool-versions for an infrastructure project
cat > .tool-versions <<EOF
opentofu 1.9.0
kubectl 1.29.0
helm 3.14.0
awscli 2.15.0
EOF

# Install all at once
asdf install
```

## Version Resolution Order

asdf resolves versions in this order:
1. `ASDF_OPENTOFU_VERSION` environment variable
2. `.tool-versions` in current directory
3. `.tool-versions` in parent directories
4. Global version (`~/.tool-versions`)

```bash
# Override via environment variable
ASDF_OPENTOFU_VERSION=1.8.5 tofu version

# Or set per shell session
export ASDF_OPENTOFU_VERSION=1.8.5
tofu version
```

## Verifying Versions

```bash
# Check which version asdf would use in current directory
asdf current opentofu

# List all installed versions
asdf list opentofu

# Where is the binary?
asdf which opentofu
```

## Reshimming After Installation

```bash
# If tofu command not found after install, reshim
asdf reshim opentofu

# Verify
which tofu
tofu version
```

## Updating the Plugin

```bash
# Update the OpenTofu plugin to get new versions available
asdf plugin update opentofu

# List new versions available
asdf list all opentofu | tail -10
```

## Conclusion

asdf provides a unified approach to managing all your development tool versions, including OpenTofu. When you already use asdf for other languages and tools, extending it to OpenTofu keeps your versioning workflow consistent and your `.tool-versions` files serve as comprehensive environment specifications for your entire project toolchain.
