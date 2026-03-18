# How to Install Podman Desktop on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Podman Desktop, Linux, Installation

Description: Learn how to install Podman Desktop on various Linux distributions for a graphical container management experience.

---

> Podman Desktop on Linux gives you a native graphical interface for container management that integrates directly with your system Podman installation without any virtual machine overhead.

Unlike macOS and Windows, Podman on Linux runs containers natively without a VM. Podman Desktop on Linux connects directly to the local Podman engine, giving you a lightweight and responsive graphical interface. This post covers installation on major Linux distributions including Ubuntu, Fedora, and Arch Linux.

---

## System Requirements

Before installing, ensure your system meets the minimum requirements:

- A 64-bit Linux distribution
- Podman 4.x or later (5.x recommended)
- At least 2 GB of RAM
- A graphical desktop environment (GNOME, KDE, etc.)

```bash
# Check your distribution
cat /etc/os-release

# Check if Podman is installed
podman --version

# Check available memory
free -h
```

## Step 1: Install the Podman Engine

Install Podman first if it is not already on your system.

### Ubuntu / Debian

```bash
# Update package lists
sudo apt-get update

# Install Podman
sudo apt-get install -y podman

# Verify installation
podman --version
```

### Fedora

```bash
# Podman is pre-installed on Fedora, but ensure it is up to date
sudo dnf update -y podman

# Verify installation
podman --version
```

### Arch Linux

```bash
# Install Podman from the official repository
sudo pacman -S podman

# Verify installation
podman --version
```

## Step 2: Install Podman Desktop

### Method A: Flatpak (Recommended for All Distributions)

Flatpak provides a universal installation method that works across all Linux distributions.

```bash
# Install Flatpak if not already present
# Ubuntu/Debian:
sudo apt-get install -y flatpak

# Fedora (usually pre-installed):
sudo dnf install -y flatpak

# Add the Flathub repository
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo

# Install Podman Desktop
flatpak install -y flathub io.podman_desktop.PodmanDesktop

# Launch Podman Desktop
flatpak run io.podman_desktop.PodmanDesktop
```

### Method B: Download the AppImage

AppImage bundles work on any Linux distribution without installation.

```bash
# Download the latest Podman Desktop AppImage
# Visit the download page
xdg-open https://podman-desktop.io/downloads

# After downloading, make it executable
chmod +x podman-desktop-*.AppImage

# Run the AppImage
./podman-desktop-*.AppImage
```

### Method C: Fedora (DNF/RPM)

```bash
# Install from the Fedora repositories if available
sudo dnf install -y podman-desktop

# Or download the RPM package from the releases page
# and install manually
sudo dnf install -y ./podman-desktop-*.rpm
```

### Method D: Ubuntu/Debian (DEB Package)

```bash
# Download the .deb package from the releases page
# Then install it
sudo dpkg -i podman-desktop-*.deb

# Fix any dependency issues
sudo apt-get install -f
```

## Step 3: Launch and Initial Setup

Launch Podman Desktop from your application menu or the command line.

```bash
# Launch from Flatpak
flatpak run io.podman_desktop.PodmanDesktop

# Or launch from the application menu
# Look for "Podman Desktop" in your desktop environment's app launcher
```

On first launch, Podman Desktop will:

1. Detect the local Podman installation
2. Connect to the Podman engine
3. Show the dashboard with containers, images, and pods

Since Linux runs Podman natively, there is no machine initialization step.

## Verifying the Installation

Confirm everything is working.

```bash
# Test Podman from the command line
podman run --rm docker.io/library/hello-world

# List all containers
podman ps -a

# Check Podman system info
podman info
```

The test container should appear in both the CLI output and the Podman Desktop GUI.

## Enabling Rootless Containers

Podman on Linux defaults to rootless mode, which is more secure.

```bash
# Verify rootless mode is working
podman info | grep rootless

# Check your user namespace configuration
cat /etc/subuid | grep $(whoami)
cat /etc/subgid | grep $(whoami)

# If entries are missing, add them
sudo usermod --add-subuids 100000-165535 --add-subgids 100000-165535 $(whoami)

# Apply the changes
podman system migrate
```

## Configuring Registries

Set up your preferred container registries.

```bash
# Edit the registries configuration
sudo nano /etc/containers/registries.conf

# Add unqualified search registries
# unqualified-search-registries = ["docker.io", "quay.io", "ghcr.io"]

# Verify registry configuration
podman info | grep -A 5 registries
```

## Updating Podman Desktop

Keep Podman Desktop up to date.

```bash
# Update via Flatpak
flatpak update io.podman_desktop.PodmanDesktop

# Update via DNF (Fedora)
sudo dnf update podman-desktop

# Update via APT (Ubuntu/Debian)
sudo apt-get update && sudo apt-get upgrade podman-desktop

# For AppImage, download the latest version from the website
```

## Summary

Installing Podman Desktop on Linux is straightforward with multiple options including Flatpak (universal), AppImage (no installation needed), and native packages for Fedora and Ubuntu. Since Podman runs natively on Linux without a VM, the setup is simpler than on macOS or Windows. After installing the Podman engine and Podman Desktop, you get a graphical interface that connects directly to your local Podman instance. Rootless containers work out of the box on most distributions, providing a secure default configuration.
