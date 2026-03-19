# How to Install and Configure the Rancher CLI

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, CLI, Installation

Description: Complete guide to installing and configuring the Rancher CLI on Linux, macOS, and Windows for managing Kubernetes clusters.

The Rancher CLI is a standalone binary that lets you interact with your Rancher server from the terminal. This guide covers installation on every major platform, initial configuration, and troubleshooting common setup issues.

## Downloading the Rancher CLI

### Method 1: Download from the Rancher UI

The easiest way to get the correct CLI version is directly from your Rancher instance:

1. Log into your Rancher UI
2. Click the question mark icon or help menu in the bottom-left corner
3. Click **CLI Downloads**
4. Select the appropriate binary for your operating system

This ensures version compatibility between the CLI and your server.

### Method 2: Download from GitHub Releases

You can download the CLI from the official Rancher CLI GitHub releases page. Choose the release that matches your Rancher server version.

### Method 3: Install via Package Managers

On macOS with Homebrew:

```bash
brew install rancher-cli
```

On Arch Linux:

```bash
pacman -S rancher-cli
```

## Installing on Linux

Download and install the binary:

```bash
# Download the latest release (adjust version as needed)
curl -LO https://github.com/rancher/cli/releases/download/v2.8.0/rancher-linux-amd64-v2.8.0.tar.gz

# Extract the archive
tar -xzf rancher-linux-amd64-v2.8.0.tar.gz

# Move the binary to your PATH
sudo mv rancher-v2.8.0/rancher /usr/local/bin/rancher

# Make it executable
sudo chmod +x /usr/local/bin/rancher

# Verify the installation
rancher --version
```

For ARM-based Linux systems (like Raspberry Pi):

```bash
curl -LO https://github.com/rancher/cli/releases/download/v2.8.0/rancher-linux-arm64-v2.8.0.tar.gz
tar -xzf rancher-linux-arm64-v2.8.0.tar.gz
sudo mv rancher-v2.8.0/rancher /usr/local/bin/rancher
sudo chmod +x /usr/local/bin/rancher
```

## Installing on macOS

### Using Homebrew

```bash
brew install rancher-cli
```

### Manual Installation

```bash
# For Intel Macs
curl -LO https://github.com/rancher/cli/releases/download/v2.8.0/rancher-darwin-amd64-v2.8.0.tar.gz

# For Apple Silicon Macs
curl -LO https://github.com/rancher/cli/releases/download/v2.8.0/rancher-darwin-arm64-v2.8.0.tar.gz

# Extract and install
tar -xzf rancher-darwin-*-v2.8.0.tar.gz
sudo mv rancher-v2.8.0/rancher /usr/local/bin/rancher
chmod +x /usr/local/bin/rancher

rancher --version
```

## Installing on Windows

1. Download `rancher-windows-amd64-v2.8.0.zip` from the GitHub releases page
2. Extract the ZIP file
3. Move `rancher.exe` to a directory in your PATH (e.g., `C:\Windows\System32` or a custom tools directory)
4. Open a new Command Prompt or PowerShell and verify:

```powershell
rancher --version
```

Alternatively, using the command line:

```powershell
# Download
Invoke-WebRequest -Uri "https://github.com/rancher/cli/releases/download/v2.8.0/rancher-windows-amd64-v2.8.0.zip" -OutFile rancher.zip

# Extract
Expand-Archive rancher.zip -DestinationPath C:\rancher

# Add to PATH
$env:PATH += ";C:\rancher"

# Verify
rancher --version
```

## Initial Configuration

### Logging In for the First Time

Once installed, log in to your Rancher server:

```bash
rancher login https://rancher.example.com
```

The CLI will prompt you for a Bearer Token. You can generate one from the Rancher UI under **Account & API Keys**.

To log in non-interactively (useful for scripts):

```bash
rancher login https://rancher.example.com \
  --token token-xxxxx:yyyyyyyyyyyyyyyy \
  --skip-verify
```

### Understanding the Configuration File

After logging in, the CLI creates a configuration file at `~/.rancher/cli2.json`. This file contains:

```json
{
  "Servers": {
    "rancherDefault": {
      "accessKey": "token-xxxxx",
      "secretKey": "yyyyyyyyyyyyyyyy",
      "tokenKey": "token-xxxxx:yyyyyyyyyyyyyyyy",
      "url": "https://rancher.example.com",
      "project": "c-m-abc12345:p-xyz789",
      "cacert": ""
    }
  },
  "CurrentServer": "rancherDefault"
}
```

### Working with Multiple Servers

You can configure multiple Rancher servers by logging in with different names:

```bash
# Log in to production
rancher login https://rancher-prod.example.com \
  --token token-prod:xxxx \
  --context c-m-prod:p-default

# Log in to staging with a different server name
rancher server add staging https://rancher-staging.example.com \
  --token token-staging:xxxx
```

Switch between servers:

```bash
rancher server switch
```

### Configuring for Self-Signed Certificates

If your Rancher server uses a self-signed certificate, you have two options.

Skip TLS verification:

```bash
rancher login https://rancher.example.com --token ${TOKEN} --skip-verify
```

Provide the CA certificate:

```bash
rancher login https://rancher.example.com \
  --token ${TOKEN} \
  --cacert /path/to/ca.crt
```

### Setting Up Environment Variables

For convenience, set these environment variables in your shell profile:

```bash
# Add to ~/.bashrc or ~/.zshrc
export RANCHER_URL="https://rancher.example.com"
export RANCHER_TOKEN="token-xxxxx:yyyyyyyyyyyyyyyy"
export RANCHER_SKIP_VERIFY=true
```

## Setting Up Shell Completion

### Bash Completion

```bash
# Generate completion script
rancher completion bash | sudo tee /etc/bash_completion.d/rancher > /dev/null

# Load immediately
source /etc/bash_completion.d/rancher
```

### Zsh Completion

```bash
# Generate completion script
rancher completion zsh > "${fpath[1]}/_rancher"

# Reload completions
autoload -U compinit && compinit
```

### Fish Completion

```bash
rancher completion fish > ~/.config/fish/completions/rancher.fish
```

## Verifying Your Setup

Run these commands to confirm everything is working:

```bash
# Check CLI version
rancher --version

# Verify connectivity
rancher clusters ls

# Check current context
rancher context current

# Test kubectl proxy
rancher kubectl get nodes
```

## Troubleshooting Common Issues

### "x509: certificate signed by unknown authority"

This means your Rancher server uses a certificate not trusted by your system. Use `--skip-verify` or provide the CA certificate with `--cacert`.

### "401 Unauthorized"

Your token may be expired or invalid. Generate a new one from the Rancher UI and log in again.

### "Unable to connect to the server"

Verify the Rancher URL is correct and reachable:

```bash
curl -sk https://rancher.example.com/ping
```

The response should be `pong`.

### CLI Version Mismatch

If you see unexpected errors, ensure your CLI version is compatible with your server version:

```bash
rancher --version
```

Compare this with your Rancher server version shown in the UI footer. Use the CLI version downloaded from your specific Rancher instance for guaranteed compatibility.

### Configuration File Corruption

If the CLI behaves unexpectedly, reset the configuration:

```bash
rm ~/.rancher/cli2.json
rancher login https://rancher.example.com --token ${TOKEN}
```

## Summary

Installing and configuring the Rancher CLI is a quick process on any platform. Download the binary, authenticate with your server, and set up shell completion for a productive command-line workflow. Keep the CLI version aligned with your server version, and use environment variables to simplify repeated operations.
