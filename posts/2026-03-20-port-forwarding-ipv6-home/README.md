# How to Set Up Port Forwarding with IPv6 at Home

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Port Forwarding, Firewall, Home Router, Remote Access

Description: Understand how to enable inbound access to home services over IPv6, which differs fundamentally from IPv4 port forwarding.

## IPv6 Port Forwarding is Different

In IPv4, port forwarding translates a public port to an internal private address (NAT). In IPv6, there is no NAT - every device has a globally routable address. So "port forwarding" in IPv6 means creating a firewall rule that allows inbound traffic to reach a specific device and port.

Think of it as: IPv4 = Port Forwarding, IPv6 = Firewall Allow Rule.

## Step 1: Find Your Device's IPv6 Address

First, identify the static or consistent IPv6 address of the device you want to expose:

```bash
# Linux server: find global IPv6 address

ip -6 addr show scope global

# For home lab server
# Assign a static address for consistency (see home lab guide)
```

Note: SLAAC addresses can change if using privacy extensions. Assign a static address for any service you want to expose consistently.

## Step 2: OpenWRT Firewall Allow Rule

On OpenWRT, add a firewall rule to allow inbound access:

```text
# /etc/config/firewall

config rule
    option name         'Allow-HomeServer-HTTPS'
    option family       ipv6
    option src          wan
    option dest         lan
    option dest_ip      '2001:db8:home::10'    # Your server's IPv6 address
    option dest_port    '443'
    option proto        tcp
    option target       ACCEPT
```

Apply: `service firewall restart`

## Step 3: Allowing Access on Common Router UIs

**Asus Router (ASUSWRT):**
1. Advanced Settings → Firewall → IPv6 Firewall
2. Click + to add rule:
   - Source: Any (or restrict to your remote IP)
   - Destination: Your server's IPv6 address
   - Port: Service port
   - Action: Allow

**UniFi Dream Machine:**
1. Settings → Firewall & Security
2. Select IPv6 Rules tab → Create Entry
3. Type: IPv6 → Direction: WAN In
4. Destination: Your server's /128 address
5. Port: Your service port
6. Action: Accept

**TP-Link Advanced Routers:**
1. Advanced → Security → Firewall
2. IPv6 Rules → Add
3. Configure destination address and port

## Step 4: Test the Rule

From an external network (mobile data, different WiFi):

```bash
# Test TCP connectivity to your service
nc -zv -6 your-server-ipv6 443

# Or curl
curl -6 -v https://[2001:db8:home::10]/

# Or nmap from a remote server
nmap -6 -p 443 2001:db8:home::10
```

## Step 5: Dynamic IPv6 Address Problem

If your ISP changes your prefix regularly, your device's IPv6 address changes too. Solutions:

**Option 1: Dynamic DNS with IPv6 (DDNS)**

Services like Hurricane Electric's `dns.he.net` support AAAA record DDNS:

```bash
# Update DDNS on address change (run via cron or network hook)
#!/bin/bash
IPV6=$(ip -6 addr show scope global dev eth0 | grep -oP '(?<=inet6 )[^/]+')
curl "https://dyn.dns.he.net/nic/update?hostname=myserver.dyn.example.com&password=mypass&myip=$IPV6"
```

**Option 2: Use a /128 reservation**

Configure your router to always give the same IPv6 address to a specific server (based on DUID or MAC), so the host portion stays consistent even if the prefix changes.

## IPv6 vs IPv4 Port Forwarding Comparison

| Feature | IPv4 Port Forwarding | IPv6 Firewall Allow |
|---------|---------------------|---------------------|
| NAT needed | Yes | No |
| Device address | Private (RFC1918) | Global (public) |
| Configuration | Router NAT table | Router firewall rule |
| Multiple services same port | One device per port | Unlimited (each device has unique IP) |

## Conclusion

IPv6 "port forwarding" is actually just a firewall allow rule - the device already has a globally routable address, so you only need to permit inbound traffic. This is simpler than IPv4 NAT and allows unlimited services on the same port across different devices. The main challenge is ensuring device addresses are stable, which requires static address assignment.
