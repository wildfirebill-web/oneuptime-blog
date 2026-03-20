# How to Configure Elemental Cloud-Config

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Elemental, Cloud-Config, Kubernetes, Edge, Configuration

Description: A detailed guide to configuring Elemental cloud-config for customizing OS initialization, users, networking, and services on bare metal and edge nodes.

## Introduction

Elemental cloud-config is a YAML-based configuration system (compatible with cloud-init) that runs during OS installation and first boot. It allows you to configure users, SSH keys, network settings, custom scripts, and system services declaratively, ensuring nodes are correctly configured when they join the fleet.

## Cloud-Config Structure

```yaml
# Structure of Elemental cloud-config
cloud-config:
  # User accounts
  users: []
  # SSH keys
  ssh_authorized_keys: []
  # Files to create
  write_files: []
  # Commands to run
  runcmd: []
  # Packages to install (on mutable systems)
  packages: []

elemental:
  registration: {}
  install: {}
  reset: {}
  upgrade: {}
```

## Configuring Users and Authentication

```yaml
cloud-config:
  users:
    # Configure the root user
    - name: root
      # bcrypt-hashed password (use: openssl passwd -6 yourpassword)
      passwd: "$6$rounds=4096$saltsalt$hashedpasswordhere"

    # Create an admin user
    - name: admin
      groups:
        - wheel
        - sudo
      shell: /bin/bash
      # Public SSH key for access
      ssh_authorized_keys:
        - "ssh-rsa AAAAB3NzaC1yc2E... admin@example.com"
      # Lock the password (SSH key only)
      lock_passwd: false
      passwd: "$6$rounds=4096$saltsalt$hashedpasswordhere"
```

## Configuring SSH

```yaml
cloud-config:
  # Global SSH authorized keys (applied to root)
  ssh_authorized_keys:
    - "ssh-rsa AAAAB3NzaC1yc2E... ops-team@example.com"
    - "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5... automation@example.com"

  # Write SSH daemon configuration
  write_files:
    - path: /etc/ssh/sshd_config.d/custom.conf
      content: |
        # Disable password authentication
        PasswordAuthentication no
        # Allow only specific users
        AllowUsers admin root
        # Enable pubkey authentication
        PubkeyAuthentication yes
        # Disable root login via password
        PermitRootLogin prohibit-password
      permissions: "0600"
```

## Writing Custom Files

```yaml
cloud-config:
  write_files:
    # Custom hostname based on MAC address
    - path: /etc/hostname-script.sh
      content: |
        #!/bin/bash
        # Generate hostname from MAC address
        MAC=$(cat /sys/class/net/eth0/address | tr ':' '-')
        hostnamectl set-hostname "node-${MAC}"
      permissions: "0755"

    # Custom network configuration
    - path: /etc/NetworkManager/conf.d/custom.conf
      content: |
        [main]
        dns=none
        [keyfile]
        unmanaged-devices=interface-name:lo
      permissions: "0644"

    # Custom sysctl settings
    - path: /etc/sysctl.d/99-custom.conf
      content: |
        # Increase connection tracking table size
        net.netfilter.nf_conntrack_max = 131072
        # Enable IP forwarding for Kubernetes
        net.ipv4.ip_forward = 1
        net.ipv6.conf.all.forwarding = 1
      permissions: "0644"
```

## Running Custom Commands

```yaml
cloud-config:
  runcmd:
    # Apply sysctl settings immediately
    - sysctl --system

    # Set timezone
    - timedatectl set-timezone UTC

    # Enable NTP
    - timedatectl set-ntp true

    # Configure hostname from hardware serial number
    - |
      SERIAL=$(dmidecode -s system-serial-number | tr ' ' '-')
      hostnamectl set-hostname "node-${SERIAL}"

    # Start custom services
    - systemctl enable --now my-monitoring-agent

    # Configure firewall
    - firewall-cmd --permanent --add-port=10250/tcp
    - firewall-cmd --permanent --add-port=30000-32767/tcp
    - firewall-cmd --reload
```

## Configuring Systemd Services

```yaml
cloud-config:
  write_files:
    # Create a custom systemd service
    - path: /etc/systemd/system/node-setup.service
      content: |
        [Unit]
        Description=Node Initial Setup
        After=network-online.target
        Wants=network-online.target
        ConditionPathExists=!/var/lib/node-setup-done

        [Service]
        Type=oneshot
        ExecStart=/usr/local/bin/node-setup.sh
        ExecStartPost=/usr/bin/touch /var/lib/node-setup-done
        RemainAfterExit=yes

        [Install]
        WantedBy=multi-user.target
      permissions: "0644"

  runcmd:
    # Enable the service
    - systemctl daemon-reload
    - systemctl enable node-setup.service
```

## Elemental-Specific Configuration

```yaml
elemental:
  install:
    # Target disk device
    device: /dev/sda
    # Reboot after install
    reboot: true
    # Additional kernel boot parameters
    extra-partitions: []
    # Disable SELinux
    selinux: false

  # Configuration applied during reset
  reset:
    reboot: true
    reset-persistent: true
    reset-oem: false
```

## Validating Cloud-Config

```bash
# Validate cloud-config syntax using cloud-init tools
cloud-init devel schema --config-file cloud-config.yaml

# Or use a linter
pip install cloud-init
cloud-init schema --config-file cloud-config.yaml
```

## Conclusion

Elemental cloud-config provides a powerful, declarative way to configure nodes at provisioning time. By combining user management, file creation, command execution, and service configuration, you can ensure every node in your fleet starts with exactly the right configuration. This approach eliminates configuration drift and makes node provisioning fully repeatable.
