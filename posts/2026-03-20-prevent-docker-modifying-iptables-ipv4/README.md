# How to Prevent Docker from Modifying iptables Rules for IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker, Networking, iptables, IPv4, Security, daemon.json

Description: Configure Docker to stop modifying iptables rules by disabling iptables management in daemon.json, and manually manage the required firewall rules for container networking.

## Introduction

By default, Docker automatically manages iptables rules to implement port publishing, NAT for containers, and inter-container policies. On servers with strict firewall policies managed by tools like `ufw`, `firewalld`, or custom iptables scripts, Docker's modifications can break or bypass firewall rules. Disabling Docker's iptables management lets you control the rules manually.

## Disabling iptables Management

Edit or create `/etc/docker/daemon.json`:

```bash
sudo tee /etc/docker/daemon.json << 'EOF'
{
  "iptables": false
}
EOF

sudo systemctl restart docker
```

## What Docker Normally Creates

Before disabling, Docker typically creates:
- `DOCKER` chain (container-specific rules)
- `DOCKER-USER` chain (user policy insertion point)
- `DOCKER-ISOLATION-STAGE-1` and `STAGE-2` (inter-network isolation)
- NAT rules in the `nat` table for port publishing
- MASQUERADE rules for outbound container NAT

## Adding Required Rules Manually

After disabling Docker's iptables management, you must add rules manually for containers to work:

```bash
# Allow forwarding between container bridge and host

sudo iptables -A FORWARD -i docker0 -j ACCEPT
sudo iptables -A FORWARD -o docker0 -j ACCEPT

# NAT for outbound container traffic (replace eth0 with your WAN interface)
sudo iptables -t nat -A POSTROUTING -s 172.17.0.0/16 -o eth0 -j MASQUERADE

# Allow established connections back into the container network
sudo iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
```

## Publishing Ports Manually

When `iptables: false`, Docker's `-p` flag does not create iptables rules. Do it yourself:

```bash
# Publish container port 80 as host port 8080
docker run -d --name web nginx

# Get the container IP
CONTAINER_IP=$(docker inspect web --format '{{.NetworkSettings.IPAddress}}')

# Forward host port 8080 to container port 80
sudo iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination $CONTAINER_IP:80
sudo iptables -A FORWARD -p tcp -d $CONTAINER_IP --dport 80 -j ACCEPT
```

## Using DOCKER-USER Chain Instead

If you want Docker to manage most rules but add your own policies, use the `DOCKER-USER` chain (not disabled):

```bash
# Insert your custom rules BEFORE Docker's rules
sudo iptables -I DOCKER-USER -i eth0 -s 203.0.113.0/24 -j DROP

# Allow only trusted IPs to reach published ports
sudo iptables -I DOCKER-USER -i eth0 ! -s 192.168.1.0/24 -j REJECT
```

## Choosing the Right Approach

| Approach | When to Use |
|---|---|
| Default (Docker manages iptables) | Most use cases |
| DOCKER-USER chain | Need to add policies without disabling Docker's rules |
| `iptables: false` + manual rules | Strict firewall environments, server hardening |

## Conclusion

Disable Docker's iptables management with `"iptables": false` in `daemon.json` for maximum control. You must then manually add NAT and FORWARD rules. For most environments, using the `DOCKER-USER` chain to prepend custom policies is a better balance of control and simplicity.
