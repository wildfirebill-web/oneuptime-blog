# How to Configure Elemental Network Settings

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Elemental, Kubernetes, Networking, Edge, Configuration

Description: Configure network settings for Elemental nodes including static IPs, bonding, VLANs, and DNS using cloud-config and NetworkManager.

## Introduction

Proper network configuration is essential for Elemental nodes, especially in edge environments where DHCP may not be available or where specific network topologies are required. Elemental supports full NetworkManager-based network configuration through cloud-config, allowing you to define static IPs, bonded interfaces, VLANs, and custom DNS settings.

## Basic DHCP Configuration (Default)

By default, Elemental nodes use DHCP. This is configured automatically during registration. No additional steps are needed for DHCP environments.

## Configuring Static IP Addresses

```yaml
# static-ip-config.yaml - Include in your MachineRegistration cloud-config
cloud-config:
  write_files:
    # Create NetworkManager connection profile for static IP
    - path: /etc/NetworkManager/system-connections/eth0.nmconnection
      content: |
        [connection]
        id=eth0-static
        type=ethernet
        interface-name=eth0
        autoconnect=true

        [ethernet]
        auto-negotiate=true

        [ipv4]
        method=manual
        address1=192.168.1.100/24,192.168.1.1
        dns=8.8.8.8;8.8.4.4;
        dns-search=example.com;

        [ipv6]
        method=auto
      permissions: "0600"

  runcmd:
    # Reload NetworkManager to apply configuration
    - nmcli connection reload
    - nmcli connection up eth0-static
```

## Configuring Network Bonding

```yaml
cloud-config:
  write_files:
    # Bond master connection
    - path: /etc/NetworkManager/system-connections/bond0.nmconnection
      content: |
        [connection]
        id=bond0
        type=bond
        interface-name=bond0
        autoconnect=true

        [bond]
        mode=active-backup
        miimon=100
        primary=eth0

        [ipv4]
        method=manual
        address1=10.0.1.50/24,10.0.1.1
        dns=10.0.1.1;
      permissions: "0600"

    # Bond slave 1
    - path: /etc/NetworkManager/system-connections/eth0-bond.nmconnection
      content: |
        [connection]
        id=eth0-bond-slave
        type=ethernet
        interface-name=eth0
        master=bond0
        slave-type=bond
        autoconnect=true
      permissions: "0600"

    # Bond slave 2
    - path: /etc/NetworkManager/system-connections/eth1-bond.nmconnection
      content: |
        [connection]
        id=eth1-bond-slave
        type=ethernet
        interface-name=eth1
        master=bond0
        slave-type=bond
        autoconnect=true
      permissions: "0600"
```

## Configuring VLANs

```yaml
cloud-config:
  write_files:
    # VLAN 100 - Management
    - path: /etc/NetworkManager/system-connections/vlan100.nmconnection
      content: |
        [connection]
        id=vlan100
        type=vlan
        interface-name=eth0.100
        autoconnect=true

        [vlan]
        id=100
        parent=eth0

        [ipv4]
        method=manual
        address1=172.16.100.50/24,172.16.100.1
      permissions: "0600"
```

## Configuring Custom DNS

```yaml
cloud-config:
  write_files:
    # Custom resolv.conf
    - path: /etc/resolv.conf
      content: |
        # Primary DNS
        nameserver 10.0.0.1
        # Secondary DNS
        nameserver 8.8.8.8
        # Search domains
        search example.com internal.example.com
      permissions: "0644"

    # Disable systemd-resolved interference
    - path: /etc/NetworkManager/conf.d/dns.conf
      content: |
        [main]
        dns=none
      permissions: "0644"
```

## Setting Static Hostname

```yaml
cloud-config:
  runcmd:
    # Set static hostname
    - hostnamectl set-hostname node-datacenter1-001

    # Or derive hostname from hardware info
    - |
      SERIAL=$(dmidecode -s system-serial-number | tr ' ' '-' | tr '[:upper:]' '[:lower:]')
      hostnamectl set-hostname "node-${SERIAL}"
```

## Conclusion

Elemental's cloud-config networking support gives you full control over node network configuration at provisioning time. Whether you need simple static IPs, complex bonded interfaces for high availability, or VLAN segmentation, NetworkManager profiles deployed via cloud-config provide a reliable and reproducible approach to network setup across your entire edge fleet.
