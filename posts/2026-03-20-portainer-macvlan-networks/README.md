# How to Configure Macvlan Networks for Direct LAN Access in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Macvlan, Docker Networking, LAN Access, Network, IP Address

Description: Learn how to configure Docker macvlan networks in Portainer so containers get their own IP addresses on your physical LAN, appearing as separate devices.

---

Macvlan allows Docker containers to appear as physical devices on your LAN, each with their own MAC and IP address. This is ideal for home automation systems, network appliances (Pi-hole, AdGuard Home), and any service that needs to be directly reachable on the local network.

## Prerequisites

- A physical network interface (e.g., `eth0`)
- IP address range on your LAN available for container assignment
- Promiscuous mode enabled on the interface (often required on VMs)

```bash
# Enable promiscuous mode on the host interface (VMs only)
sudo ip link set eth0 promisc on

# Make it persistent
echo 'ACTION=="add", KERNEL=="eth0", RUN+="/sbin/ip link set %k promisc on"' \
  | sudo tee /etc/udev/rules.d/99-promisc.rules
```

## Creating a Macvlan Network in Portainer

In Portainer, go to **Networks > Add network**:

- **Driver**: `macvlan`
- **Configuration > Parent network card**: `eth0` (your physical interface)
- **Subnet**: Your LAN subnet, e.g., `192.168.1.0/24`
- **Gateway**: Your router, e.g., `192.168.1.1`
- **IP range**: A range of IPs reserved for containers, e.g., `192.168.1.200/29` (gives .200-.207)

Or create via CLI:

```bash
docker network create \
  --driver macvlan \
  --subnet 192.168.1.0/24 \
  --gateway 192.168.1.1 \
  --ip-range 192.168.1.200/29 \
  --opt parent=eth0 \
  lan_macvlan
```

## Using the Macvlan Network in a Stack

Assign a specific IP to each container:

```yaml
version: "3.8"

services:
  pihole:
    image: pihole/pihole:latest
    hostname: pihole
    networks:
      lan_macvlan:
        ipv4_address: 192.168.1.200   # Fixed LAN IP for Pi-hole
    environment:
      TZ: America/New_York
      WEBPASSWORD: adminpassword
      DNSMASQ_LISTENING: all
    volumes:
      - pihole_data:/etc/pihole
      - dnsmasq_data:/etc/dnsmasq.d

  adguard:
    image: adguard/adguardhome:latest
    hostname: adguard
    networks:
      lan_macvlan:
        ipv4_address: 192.168.1.201   # Different LAN IP for AdGuard
    volumes:
      - adguard_data:/opt/adguardhome/work
      - adguard_conf:/opt/adguardhome/conf

volumes:
  pihole_data:
  dnsmasq_data:
  adguard_data:
  adguard_conf:

networks:
  lan_macvlan:
    external: true   # Reference the pre-created macvlan network
```

## Accessing Containers from the Host

A limitation of macvlan is that the host cannot directly communicate with containers on the macvlan network (they have different MAC addresses on the same interface). Create a macvlan interface on the host to work around this:

```bash
# Create a host-side macvlan interface to communicate with containers
sudo ip link add macvlan0 link eth0 type macvlan mode bridge
sudo ip addr add 192.168.1.205/32 dev macvlan0
sudo ip link set macvlan0 up

# Add a route to the container IP range
sudo ip route add 192.168.1.200/29 dev macvlan0
```

## 802.1Q VLAN Tagging

For setups with VLAN-tagged traffic, use a dot-notation parent:

```bash
# First create the VLAN interface on the host
sudo ip link add link eth0 name eth0.100 type vlan id 100
sudo ip link set eth0.100 up

# Create a macvlan on the VLAN interface
docker network create \
  --driver macvlan \
  --opt parent=eth0.100 \
  --subnet 192.168.100.0/24 \
  --gateway 192.168.100.1 \
  vlan100_net
```

## When to Use Macvlan vs Bridge

| Scenario | Use |
|----------|-----|
| Container needs a LAN IP | Macvlan |
| Avoid NAT for specific services | Macvlan |
| Normal app stack | Bridge |
| Containers communicate with each other only | Bridge with internal |
| Swarm multi-host | Overlay |
