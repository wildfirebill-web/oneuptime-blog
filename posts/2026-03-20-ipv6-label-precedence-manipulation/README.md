# How to Manipulate IPv6 Label and Precedence Values

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, RFC 6724, Policy Table, LABEL, Precedence, Linux, Address Selection

Description: Manipulate IPv6 address selection label and precedence values on Linux to control source address selection and destination sorting for specific traffic patterns.

## Labels vs Precedence

RFC 6724 uses two distinct mechanisms:

| Mechanism | Controls | Tool |
|---|---|---|
| Label | Source/destination pairing (prefer same label) | `ip addrlabel` (kernel) + `gai.conf` |
| Precedence | Destination sort order (higher = tried first) | `gai.conf` only |

Labels are used to pair source and destination addresses. Precedence ranks destination addresses regardless of source.

## Understanding Label Matching

RFC 6724 Rule 6: "Prefer matching label" - if the source address label matches the destination address label, that source-destination pair is preferred.

```bash
# Default label assignments:

# ::1/128        label 0   (loopback)
# ::/0           label 1   (global IPv6)
# ::ffff:0:0/96  label 4   (IPv4-mapped)
# 2002::/16      label 2   (6to4)
# 2001::/32      label 5   (Teredo)
# fc00::/7       label 13  (ULA)

# Result:
# Global IPv6 source (label 1) matches global IPv6 dest (label 1) → preferred
# ULA source (label 13) matches ULA dest (label 13) → stays local
# IPv4-mapped source (label 4) matches IPv4 dest (label 4) → IPv4 path
```

## Manipulating Labels with ip addrlabel

```bash
# View current kernel label table
ip addrlabel list

# Add a custom label for a specific prefix
# Use case: make connections to 2001:db8:cdn::/48 prefer ULA source
ip addrlabel add prefix 2001:db8:cdn::/48 label 13

# Now:
# Destination 2001:db8:cdn:: has label 13
# ULA source fd00:: has label 13 → MATCH (ULA source preferred for CDN)
# Global source 2001:db8:: has label 1 → NO MATCH

# Remove the custom label
ip addrlabel del prefix 2001:db8:cdn::/48

# Flush all custom entries (restore RFC 6724 defaults)
ip addrlabel flush
```

## Manipulating Precedence in gai.conf

```bash
# Higher precedence = destination is tried first
# Useful for controlling IPv4 vs IPv6 and tunnel preference

# Use case 1: Force IPv6 for specific prefix, IPv4 for everything else
cat > /etc/gai.conf << 'EOF'
# Standard labels
label ::1/128        0
label ::/0           1
label ::ffff:0:0/96  4
label 2002::/16      2
label 2001::/32      5
label fc00::/7       13

# Raise IPv6 precedence globally
precedence ::1/128       50
precedence ::/0          40   # global IPv6
precedence ::ffff:0:0/96 35   # IPv4 (lower = tried after IPv6)

# Raise specific prefix even higher (always tried first)
precedence 2001:db8:cdn::/48  100
EOF
```

## Advanced: Creating Traffic Policies with Labels

```bash
# Scenario: 3 subnets, traffic must stay within subnet
# Subnet A: 2001:db8:a::/48 - label 20
# Subnet B: 2001:db8:b::/48 - label 21
# Global:   ::/0            - label 1

# Add labels (kernel table - affects source selection)
ip addrlabel add prefix 2001:db8:a::/48 label 20
ip addrlabel add prefix 2001:db8:b::/48 label 21

# gai.conf - affects getaddrinfo destination sorting
cat >> /etc/gai.conf << 'EOF'
label 2001:db8:a::/48 20
label 2001:db8:b::/48 21
EOF

# Result:
# Host in subnet A connecting to subnet A dest → source from 2001:db8:a:: (label 20 matches)
# Host in subnet A connecting to subnet B dest → source from ::/0 label 1 (no label 20 match)
# Traffic policy enforced purely through address selection
```

## Preventing ULA Leakage to Internet

```bash
# Verify ULA addresses don't get selected for global destinations

# ULA should have label 13 (not label 1)
ip addrlabel list | grep "fc00"
# prefix fc00::/7 label 13  ← correct

# Global destinations have label 1 - no match with ULA label 13
# So global destinations will not prefer ULA source addresses

# Test: ULA source should NOT be chosen for global destination
python3 -c "
import socket
s = socket.socket(socket.AF_INET6, socket.SOCK_DGRAM)
s.connect(('2001:db8::1', 80))
src = s.getsockname()[0]
print(f'Source for global dest: {src}')
# Should NOT be fd00:: or fc00::/7 range
if src.startswith('fd') or src.startswith('fc'):
    print('WARNING: ULA source selected for global destination!')
else:
    print('OK: Non-ULA source selected')
"
```

## Persistent Configuration

```bash
# Save ip addrlabel rules across reboots using systemd
cat > /etc/systemd/system/ipv6-policy.service << 'EOF'
[Unit]
Description=IPv6 Address Selection Policy
After=network.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/bash -c '\
    ip addrlabel flush; \
    ip addrlabel add prefix 2001:db8:cdn::/48 label 13; \
    ip addrlabel add prefix 2001:db8:a::/48 label 20; \
    ip addrlabel add prefix 2001:db8:b::/48 label 21'
ExecStop=/usr/sbin/ip addrlabel flush

[Install]
WantedBy=multi-user.target
EOF

systemctl enable --now ipv6-policy.service
```

## Testing Label Changes

```bash
#!/bin/bash
# test-labels.sh - Verify label assignments and source selection

echo "=== Kernel label table ==="
ip addrlabel list

echo ""
echo "=== Source selection for various destinations ==="

DESTS=(
    "2001:db8::1"      # global
    "fd00::1"          # ULA
    "2001:db8:cdn::1"  # CDN prefix (if custom label added)
    "::1"              # loopback
)

for dest in "${DESTS[@]}"; do
    src=$(python3 -c "
import socket
try:
    s = socket.socket(socket.AF_INET6, socket.SOCK_DGRAM)
    s.connect(('${dest}', 80))
    print(s.getsockname()[0])
    s.close()
except Exception as e:
    print(f'error: {e}')
")
    echo "  Dest: ${dest:<30} → Source: ${src}"
done

echo ""
echo "=== gai.conf active labels ==="
grep "^label" /etc/gai.conf 2>/dev/null || echo "(using built-in defaults)"
```

## Conclusion

Label manipulation is the most powerful aspect of RFC 6724 policy control. By assigning the same label to a source prefix and destination prefix, you force traffic to prefer that source for those destinations. Use `ip addrlabel add prefix <prefix> label <N>` for kernel-level source selection, and add matching `label` entries to `/etc/gai.conf` for userspace `getaddrinfo()` destination sorting. Precedence values in `gai.conf` control absolute destination ordering - raise a prefix's precedence to make it always tried first. Persist kernel label rules across reboots with a systemd oneshot service that runs `ip addrlabel` commands at startup.
