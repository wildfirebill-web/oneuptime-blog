# How to Troubleshoot WireGuard IPv4 Connectivity and Handshake Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: WireGuard, VPN, IPv4, Troubleshooting, Networking, Linux

Description: Diagnose and fix common WireGuard IPv4 connectivity problems including failed handshakes, dropped tunnels, and routing issues.

WireGuard is silent by design — it doesn't send error messages when connections fail. This makes debugging require a systematic approach, checking keys, firewall rules, routing, and logs.

## Step 1: Check Interface Status

```bash
# Show WireGuard interface details including peer handshake status
sudo wg show

# Key fields to check:
# - latest handshake: should be recent (within last 3 minutes if active)
# - transfer: bytes should be increasing
# - allowed ips: must match expected subnets
```

If `latest handshake` shows `(none)`, the tunnel has never established.

## Step 2: Verify Key Configuration

Mismatched keys are the most common cause of failed handshakes.

```bash
# On the server, display the public key
sudo cat /etc/wireguard/server_public.key

# On the client, verify the [Peer] PublicKey matches the server's public key
grep PublicKey /etc/wireguard/wg0.conf
```

Also verify that the server has the client's correct public key in its `[Peer]` section.

## Step 3: Check Firewall and UDP Port Access

```bash
# On the server, verify UDP 51820 is open
sudo iptables -L INPUT -n -v | grep 51820

# Test connectivity from the client
nc -zvu <SERVER_PUBLIC_IP> 51820

# Check if the server is actually listening on UDP 51820
sudo ss -ulnp | grep 51820
```

## Step 4: Verify Endpoint Reachability

```bash
# Ping the server's public IP (basic reachability)
ping <SERVER_PUBLIC_IP>

# If ICMP is blocked, test with nmap
nmap -sU -p 51820 <SERVER_PUBLIC_IP>
```

## Step 5: Enable Debug Logging

```bash
# Enable verbose WireGuard logging in the kernel
echo module wireguard +p | sudo tee /sys/kernel/debug/dynamic_debug/control

# Watch kernel logs for WireGuard events
sudo dmesg -w | grep wireguard
```

Look for messages like `Invalid handshake initiation` or `Receiving handshake initiation`.

## Step 6: Check AllowedIPs Routing

```bash
# Verify routes are installed for AllowedIPs
ip route show dev wg0

# For a full tunnel, verify default route points to wg0
ip route get 8.8.8.8
```

## Step 7: Check IP Forwarding on Server

```bash
# Must be 1 for server to route client traffic
sysctl net.ipv4.ip_forward
# If 0: sudo sysctl -w net.ipv4.ip_forward=1
```

## Common Issues and Fixes

| Symptom | Likely Cause | Fix |
|---|---|---|
| No handshake | Wrong public key | Regenerate and re-exchange keys |
| Handshake OK, no traffic | IP forwarding disabled | Enable `net.ipv4.ip_forward` |
| Handshake OK, no internet | NAT not configured | Add MASQUERADE iptables rule |
| Intermittent drops | MTU mismatch | Set `MTU = 1420` in `[Interface]` |
| Client can't reach server | Firewall blocking UDP 51820 | Open the port |

## Resetting WireGuard State

```bash
# Restart the interface to reset state
sudo wg-quick down wg0
sudo wg-quick up wg0
```

WireGuard's stateless design means most issues stem from configuration rather than runtime state. Systematic key and firewall verification resolves the majority of connectivity problems.
