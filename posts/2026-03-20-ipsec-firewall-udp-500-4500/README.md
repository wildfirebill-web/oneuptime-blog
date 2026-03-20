# How to Configure IPsec Firewall Rules for UDP 500 and 4500

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPsec, iptables, Firewall, UDP 500, NAT-T, Linux

Description: Configure iptables and firewalld rules to allow IPsec IKE traffic on UDP 500 and NAT-T traffic on UDP 4500 for VPN tunnel establishment.

IPsec requires specific UDP ports and the ESP protocol to be open through the firewall. Without correct firewall rules, IKE negotiation fails before the tunnel can establish.

## Ports Required by IPsec

| Protocol/Port | Purpose |
|---|---|
| UDP 500 | IKE (Internet Key Exchange) |
| UDP 4500 | IKE NAT Traversal (NAT-T) |
| IP Protocol 50 (ESP) | Encrypted payload packets |
| IP Protocol 51 (AH) | Authentication Header (rarely used now) |

## iptables Rules for IPsec

```bash
# Allow IKE traffic (UDP 500) - required for all IPsec
sudo iptables -A INPUT -p udp --dport 500 -j ACCEPT

# Allow NAT-T traffic (UDP 4500) - required when behind NAT
sudo iptables -A INPUT -p udp --dport 4500 -j ACCEPT

# Allow ESP (protocol 50) - the actual encrypted data
sudo iptables -A INPUT -p esp -j ACCEPT

# Allow AH (protocol 51) - authentication without encryption
sudo iptables -A INPUT -p ah -j ACCEPT

# Allow outbound IKE and ESP
sudo iptables -A OUTPUT -p udp --dport 500 -j ACCEPT
sudo iptables -A OUTPUT -p udp --dport 4500 -j ACCEPT
sudo iptables -A OUTPUT -p esp -j ACCEPT
```

## Allow Forwarding for IPsec Traffic

```bash
# Allow traffic to pass through when protected by IPsec
sudo iptables -A FORWARD -m policy --dir in --pol ipsec -j ACCEPT
sudo iptables -A FORWARD -m policy --dir out --pol ipsec -j ACCEPT

# Or with specific subnet restrictions:
sudo iptables -A FORWARD -s 192.168.1.0/24 -m policy --dir in --pol ipsec -j ACCEPT
sudo iptables -A FORWARD -d 192.168.1.0/24 -m policy --dir out --pol ipsec -j ACCEPT
```

## Restricting to Specific Peer IPs

```bash
# Only allow IPsec from known peer IPs (more secure)
PEER_IP="5.6.7.8"

sudo iptables -A INPUT -s $PEER_IP -p udp --dport 500 -j ACCEPT
sudo iptables -A INPUT -s $PEER_IP -p udp --dport 4500 -j ACCEPT
sudo iptables -A INPUT -s $PEER_IP -p esp -j ACCEPT

# Drop everything else on IPsec ports
sudo iptables -A INPUT -p udp --dport 500 -j DROP
sudo iptables -A INPUT -p udp --dport 4500 -j DROP
```

## firewalld Rules (RHEL/CentOS)

```bash
# Using firewalld services
sudo firewall-cmd --permanent --add-service=ipsec
sudo firewall-cmd --reload

# Verify the service was added
sudo firewall-cmd --list-services

# What the ipsec service includes:
# UDP 500, UDP 4500, protocol: esp, protocol: ah
sudo firewall-cmd --info-service=ipsec
```

## UFW Rules (Ubuntu)

```bash
# UFW doesn't handle ESP directly well; use iptables underneath
# Allow IKE
sudo ufw allow 500/udp
sudo ufw allow 4500/udp

# Allow ESP via direct iptables rule (UFW doesn't support protocol 50 directly)
sudo iptables -A INPUT -p esp -j ACCEPT
sudo iptables -A OUTPUT -p esp -j ACCEPT

# Persist the iptables rule
echo "-A INPUT -p esp -j ACCEPT" | sudo tee -a /etc/ufw/before.rules
```

## Verifying Rules are Active

```bash
# Check UDP 500 rule
sudo iptables -L INPUT -n | grep "500\|dpt:500"

# Check ESP rule
sudo iptables -L INPUT -n | grep "esp\|proto 50"

# Test from the remote peer side
nc -zuv <GATEWAY_IP> 500
nc -zuv <GATEWAY_IP> 4500
```

## NAT Considerations

If the IPsec gateway is behind NAT, the NAT device must:
1. Allow UDP 500 and 4500 inbound
2. Forward those ports to the internal gateway
3. Allow ESP protocol (protocol 50) passthrough — many NAT devices block this

```bash
# If ESP is being NATed (behind NAT), force NAT-T in strongSwan config:
# forceencaps=yes
# This wraps ESP in UDP 4500, which NAT can forward
```

Correct firewall rules for IPsec ports are a prerequisite for any successful tunnel establishment.
