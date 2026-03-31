# How to Install the Podman AI Lab Extension

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Container, DevOps, AI, Machine Learning, Podman Desktop, AI Lab

Description: Learn how to install and set up the Podman AI Lab extension in Podman Desktop for running local AI models and experiments.

---

> Podman AI Lab brings local AI development to your desktop without sending data to the cloud.

Running AI models locally gives you full control over your data and eliminates cloud dependency. The Podman AI Lab extension for Podman Desktop provides a graphical interface to download, run, and experiment with large language models and other AI tools directly on your machine. This guide walks you through the complete installation process.

---

## Prerequisites

Before installing the AI Lab extension, you need Podman and Podman Desktop installed on your system.

### Install Podman

```bash
# On Fedora/RHEL

sudo dnf install podman -y

# On Ubuntu/Debian
sudo apt update
sudo apt install podman -y

# On macOS with Homebrew
brew install podman

# Initialize and start the Podman machine (macOS/Windows)
podman machine init
podman machine start

# Verify Podman is running
podman info --format '{{.Host.RemoteSocket.Path}}'
```

### Install Podman Desktop

```bash
# On Fedora using Flathub
flatpak install -y flathub io.podman_desktop.PodmanDesktop

# On macOS with Homebrew
brew install podman-desktop

# On Ubuntu/Debian, download the .deb from the official site
# Visit https://podman-desktop.io/downloads and grab the latest .deb
sudo dpkg -i podman-desktop-*.deb
```

## Installing the AI Lab Extension

### Method 1: Install from the Podman Desktop UI

```bash
# Launch Podman Desktop
# On Linux
flatpak run io.podman_desktop.PodmanDesktop

# On macOS
open -a "Podman Desktop"
```

Once Podman Desktop is open:

1. Navigate to the **Extensions** tab in the left sidebar.
2. Search for **Podman AI Lab** in the catalog.
3. Click **Install** next to the Podman AI Lab extension.
4. Wait for the download and installation to complete.
5. The AI Lab icon will appear in the left sidebar.

### Method 2: Install via the CLI

```bash
# Install the AI Lab extension directly from the OCI image
podman desktop extension install ghcr.io/containers/podman-desktop-extension-ai-lab:latest

# Verify the extension is listed
podman desktop extension list
```

## Verifying the Installation

```bash
# Check that the Podman machine has sufficient resources for AI workloads
podman machine inspect --format '{{.Resources.CPUs}} CPUs, {{.Resources.Memory}}MB RAM'

# For AI models, you typically need at least 8GB of RAM allocated
# If your machine has less, increase it
podman machine stop
podman machine set --memory 12288 --cpus 6
podman machine start
```

### Confirm Extension Is Active

```bash
# List installed extensions and check AI Lab status
podman desktop extension list | grep ai-lab

# Check that the AI Lab backend containers are running
podman ps --filter "label=ai-lab" --format "{{.Names}}\t{{.Status}}"
```

## Configuring the Extension

### Set the Model Storage Directory

```bash
# By default, models are stored in the Podman machine
# Check available disk space on the Podman machine
podman machine ssh df -h /

# If you need more space, increase the disk size
podman machine stop
podman machine set --disk-size 100
podman machine start
```

### Adjust Resource Limits

```bash
# Check current resource allocation
podman machine inspect --format 'CPUs: {{.Resources.CPUs}}, Memory: {{.Resources.Memory}}MB, Disk: {{.Resources.DiskSize}}GB'

# Recommended minimums for AI workloads:
# - CPUs: 4+
# - Memory: 8192MB+ (8GB)
# - Disk: 50GB+ (models can be large)

# Apply recommended settings
podman machine stop
podman machine set --cpus 8 --memory 16384 --disk-size 100
podman machine start
```

## Troubleshooting Common Issues

### Extension Fails to Install

```bash
# Clear the extension cache and retry
podman desktop extension remove ghcr.io/containers/podman-desktop-extension-ai-lab || true
podman system prune --volumes -f

# Pull the extension image manually
podman pull ghcr.io/containers/podman-desktop-extension-ai-lab:latest

# Retry installation
podman desktop extension install ghcr.io/containers/podman-desktop-extension-ai-lab:latest
```

### Podman Machine Not Responding

```bash
# Reset the Podman machine if it becomes unresponsive
podman machine stop
podman machine rm -f
podman machine init --cpus 8 --memory 16384 --disk-size 100
podman machine start

# Reinstall the extension after machine reset
podman desktop extension install ghcr.io/containers/podman-desktop-extension-ai-lab:latest
```

### Check Logs for Errors

```bash
# View Podman Desktop logs for debugging
# On macOS
cat ~/Library/Logs/Podman\ Desktop/main.log | tail -50

# On Linux
cat ~/.local/share/podman-desktop/logs/main.log | tail -50
```

## Summary

Installing the Podman AI Lab extension is straightforward with either the graphical Podman Desktop interface or the CLI. The key steps are ensuring Podman and Podman Desktop are installed, allocating sufficient resources to the Podman machine for AI workloads, and installing the extension from the catalog. With the AI Lab extension active, you can begin downloading and running AI models locally without any cloud services.
