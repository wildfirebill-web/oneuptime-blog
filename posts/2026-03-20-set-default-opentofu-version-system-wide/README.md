# How to Set a Default OpenTofu Version System-Wide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Version Management, tofuenv, asdf, Linux, macOS, Infrastructure as Code

Description: Learn how to configure a system-wide default OpenTofu version that applies to all users and projects without a local version override.

---

A system-wide default OpenTofu version is the version used when no project-specific version file (`.opentofu-version` or `.tool-versions`) is present. Setting this correctly ensures new projects start with a sensible baseline and reduces "works on my machine" issues.

---

## Method 1: tofuenv Global Version

```bash
# Set the global (system-wide) default
tofuenv use 1.9.0

# tofuenv stores the global version in
cat ~/.tofuenv/version
# 1.9.0

# Any directory without a .opentofu-version file uses this version
cd /tmp
tofu version
# OpenTofu v1.9.0
```

---

## Method 2: asdf Global Version

```bash
# Set the global default with asdf
asdf global opentofu 1.9.0

# asdf stores global versions in
cat ~/.tool-versions
# opentofu 1.9.0

# Verify
tofu version
# OpenTofu v1.9.0
```

---

## Method 3: System-Wide Binary Installation

For a system-wide installation that all users share (without a version manager):

```bash
TOFU_VERSION="1.9.0"

# Download and install to /usr/local/bin (requires sudo)
curl -LO "https://github.com/opentofu/opentofu/releases/download/v${TOFU_VERSION}/tofu_${TOFU_VERSION}_linux_amd64.zip"
unzip tofu_${TOFU_VERSION}_linux_amd64.zip
sudo install -m 755 tofu /usr/local/bin/tofu

# Verify for all users
tofu version
# OpenTofu v1.9.0

# Check it's in a system-wide PATH location
which tofu
# /usr/local/bin/tofu
```

---

## Method 4: Environment Module System (HPC/Multi-User Servers)

For multi-user servers with the environment modules system:

```bash
# Create an OpenTofu module file
mkdir -p /usr/share/modules/opentofu
cat > /usr/share/modules/opentofu/1.9.0 << 'EOF'
#%Module1.0
prepend-path PATH /opt/opentofu/1.9.0/bin
setenv OPENTOFU_VERSION 1.9.0
EOF

# Set a default version
ln -s /usr/share/modules/opentofu/1.9.0 /usr/share/modules/opentofu/default

# Users load the module
module load opentofu
tofu version
```

---

## Verify the System Default

```bash
# Check what version is active globally (no project override)
cd ~
tofu version

# With tofuenv: see global vs local priority
tofuenv list
# Shows * next to the currently active version with its source
```

---

## CI/CD System-Wide Default

For CI/CD systems (GitHub Actions, GitLab CI), pin the version in the workflow file to ensure consistent defaults.

```yaml
# GitHub Actions — set system-wide default for the pipeline
- name: Install OpenTofu
  uses: opentofu/setup-opentofu@v1
  with:
    tofu_version: "1.9.0"   # this is the "system default" for this workflow

- name: Verify version
  run: tofu version
```

---

## Summary

Setting a system-wide default OpenTofu version depends on your version manager: `tofuenv use <version>` sets the tofuenv global, `asdf global opentofu <version>` sets the asdf global, and direct binary installation to `/usr/local/bin` sets a fixed system version. Project-level `.opentofu-version` files always override the global default.
