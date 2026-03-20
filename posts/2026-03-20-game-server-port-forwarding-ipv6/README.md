# How to Configure Game Server Port Forwarding for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Port Forwarding, Game Server, Firewall, Networking, Linux

Description: Configure port forwarding for game servers using IPv6, covering router settings, firewall rules, NAT64 considerations, and direct IPv6 addressing strategies.

---

IPv6 changes the game server networking model fundamentally: because IPv6 addresses are globally routable, traditional NAT port forwarding is often unnecessary. Instead, you configure firewall rules to allow inbound connections directly to your server's IPv6 address.

## IPv6 vs IPv4 Port Forwarding

With IPv4, most home and enterprise networks use NAT requiring explicit port forwards. With IPv6, each device typically has a globally routable address, but you still need firewall rules to permit inbound connections.

```bash
# Check your server's global IPv6 address
ip -6 addr show scope global

# Verify outbound IPv6 connectivity
ping6 google.com

# Confirm no NAT on IPv6 (usually none)
ip6tables -t nat -L
```

## Firewall Rules for Common Games over IPv6

```bash
# Minecraft (TCP/UDP 25565)
sudo ip6tables -A INPUT -p tcp --dport 25565 -j ACCEPT
sudo ip6tables -A INPUT -p udp --dport 25565 -j ACCEPT

# Counter-Strike 2 (UDP 27015)
sudo ip6tables -A INPUT -p udp --dport 27015 -j ACCEPT

# Valheim (UDP 2456-2458)
sudo ip6tables -A INPUT -p udp --dport 2456:2458 -j ACCEPT

# Factorio (UDP 34197)
sudo ip6tables -A INPUT -p udp --dport 34197 -j ACCEPT

# ARK (UDP 7777, 27015)
sudo ip6tables -A INPUT -p udp --dport 7777 -j ACCEPT

# Rust (UDP/TCP 28015)
sudo ip6tables -A INPUT -p udp --dport 28015 -j ACCEPT
sudo ip6tables -A INPUT -p tcp --dport 28015 -j ACCEPT

# Save IPv6 rules
sudo ip6tables-save > /etc/ip6tables/rules.v6
```

## Router/Gateway Configuration for IPv6

```bash
# On home routers with IPv6, configure firewall rules (not NAT)
# Most modern routers expose IPv6 firewall settings separately

# On Linux-based router/gateway
# Allow inbound to specific game server from any IPv6
ip6tables -A FORWARD -d 2001:db8::gameserver -p udp --dport 25565 -j ACCEPT
ip6tables -A FORWARD -d 2001:db8::gameserver -p tcp --dport 25565 -j ACCEPT

# Apply stateful tracking
ip6tables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT
```

## nftables for IPv6 Port Forwarding

```bash
# /etc/nftables.conf - IPv6 game server rules
table ip6 filter {
  chain input {
    type filter hook input priority 0; policy drop;

    # Allow established connections
    ct state established,related accept

    # Allow loopback
    iif lo accept

    # Allow ICMPv6 (required!)
    ip6 nexthdr icmpv6 accept

    # Game server ports
    tcp dport { 25565, 28015, 27015 } accept
    udp dport { 25565, 27015, 2456-2458, 34197, 7777, 28015 } accept
  }
}
```

```bash
sudo nft -f /etc/nftables.conf
sudo systemctl enable --now nftables
```

## Handling Dual-Stack Scenarios

```bash
# If clients connect from both IPv4 and IPv6, bind server to both
# Example: bind game server to all interfaces
./game_server --ip "::" --port 25565  # Binds to both

# Verify dual-stack listening
ss -tlnp | grep 25565     # IPv4
ss -6 -tlnp | grep 25565  # IPv6
```

## Testing Inbound IPv6 Connectivity

```bash
# From external machine, test port accessibility
nmap -6 -sU -p 25565 2001:db8::gameserver
nmap -6 -sT -p 25565 2001:db8::gameserver

# Test from player's machine
telnet 2001:db8::gameserver 25565

# Use online IPv6 port checker
# Check from: https://ipv6.chappell-family.com/ipv6tcptest/
```

Unlike IPv4 port forwarding which requires NAT configuration, IPv6 game server access requires only firewall rules permitting inbound connections to the server's global IPv6 address, simplifying network configuration while maintaining security through stateful firewall policies.
