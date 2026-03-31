# How to Configure Firewall Rules for Podman Containers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Container, DevOps, Networking, Firewall, Security, firewalld

Description: Learn how to configure firewall rules to control inbound and outbound traffic for Podman containers.

---

> Firewall rules give you fine-grained control over which traffic can reach your containers and which traffic they can send.

Podman containers bind to host ports when you publish them. Without firewall rules, those ports are accessible from anywhere the host is reachable. Configuring your firewall lets you restrict access to specific source IPs, block outbound connections, and enforce security policies that go beyond network isolation.

---

## Understanding Podman and firewalld

On systems using `firewalld`, Podman container traffic passes through firewall zones. You can use `firewall-cmd` to create rules that apply to container traffic.

```bash
# Check if firewalld is running

sudo systemctl status firewalld

# List active zones
sudo firewall-cmd --get-active-zones
```

## Allowing a Published Port Through the Firewall

```bash
# Run a container with a published port
podman run -d --name webserver -p 8080:80 docker.io/library/nginx:alpine

# Allow port 8080 through the firewall
sudo firewall-cmd --zone=public --add-port=8080/tcp --permanent
sudo firewall-cmd --reload

# Verify the rule
sudo firewall-cmd --zone=public --list-ports
```

## Restricting Access to a Specific Source IP

```bash
# Create a rich rule that only allows traffic from a specific IP
sudo firewall-cmd --zone=public --add-rich-rule='
  rule family="ipv4"
  source address="192.168.1.100"
  port protocol="tcp" port="8080"
  accept' --permanent

# Remove the blanket port rule
sudo firewall-cmd --zone=public --remove-port=8080/tcp --permanent
sudo firewall-cmd --reload
```

## Creating a Dedicated Zone for Containers

```bash
# Create a new firewall zone for container traffic
sudo firewall-cmd --permanent --new-zone=containers
sudo firewall-cmd --reload

# Add the container bridge interface to the zone
BRIDGE=$(podman network inspect podman --format '{{.NetworkInterface}}')
sudo firewall-cmd --zone=containers --add-interface="$BRIDGE" --permanent

# Allow only specific ports in the container zone
sudo firewall-cmd --zone=containers --add-port=80/tcp --permanent
sudo firewall-cmd --zone=containers --add-port=443/tcp --permanent
sudo firewall-cmd --reload
```

## Blocking Outbound Traffic from Containers

```bash
# Block all outbound traffic from the container zone
sudo firewall-cmd --zone=containers --add-rich-rule='
  rule family="ipv4"
  destination not address="10.88.0.0/16"
  drop' --permanent

sudo firewall-cmd --reload
```

## Verifying Firewall Rules

```bash
# List all rules in the container zone
sudo firewall-cmd --zone=containers --list-all

# Test from inside a container
podman exec webserver curl -s --max-time 5 http://example.com
# Should fail if outbound is blocked
```

## Summary

Firewall rules complement Podman network isolation by controlling traffic at the host level. Use `firewalld` zones and rich rules to restrict which sources can reach published ports and whether containers can make outbound connections. A dedicated firewall zone for container interfaces keeps rules organized and auditable.
