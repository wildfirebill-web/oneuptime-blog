# How to View the ARP Table on macOS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, macOS, ARP, IPv4

Description: Learn how to view and interpret the ARP cache on macOS using arp, ndp, and related tools.

## Viewing the ARP Table with `arp -a`

On macOS, use the `arp` command to display the ARP cache:

```bash
arp -a
```

Sample output:

```text
gateway (192.168.1.1) at aa:bb:cc:dd:ee:ff on en0 ifscope [ethernet]
myhost.local (192.168.1.20) at 00:11:22:33:44:55 on en0 ifscope [ethernet]
? (192.168.1.255) at ff:ff:ff:ff:ff:ff on en0 ifscope permanent [ethernet]
```

### Output Format

macOS uses BSD-style `arp` output:
- `hostname (IP)` - resolved hostname and IP
- `at MAC` - the associated MAC address
- `on interface` - network interface
- `ifscope` - entry is scoped to that interface
- `permanent` - static entry

## Numerical Output (No DNS Lookups)

```bash
# Skip hostname resolution for faster output

arp -an

# Show entries for a specific interface
arp -an -i en0

# Show all interfaces
arp -an -i any 2>/dev/null || arp -an
```

## Using `ndp` for IPv6 Neighbor Cache

On macOS, `ndp` shows the IPv6 neighbor cache (equivalent to ARP for IPv6):

```bash
ndp -an
```

## Filtering ARP Entries

```bash
# Show only complete (resolved) entries
arp -an | grep -v incomplete

# Show entry for a specific IP
arp -n 192.168.1.1

# Show only entries on en0
arp -an -i en0
```

## Python: Parsing macOS ARP Output

```python
import subprocess
import re

def get_macos_arp():
    result = subprocess.run(['arp', '-an'], capture_output=True, text=True)
    entries = []
    # Pattern: ? (IP) at MAC on iface
    pattern = re.compile(r'\((\d+\.\d+\.\d+\.\d+)\) at ([0-9a-f:]+) on (\S+)', re.IGNORECASE)
    for line in result.stdout.strip().split('\n'):
        m = pattern.search(line)
        if m:
            entries.append({'ip': m.group(1), 'mac': m.group(2), 'iface': m.group(3)})
    return entries

for e in get_macos_arp():
    print(f"IP: {e['ip']:18} MAC: {e['mac']:20} Interface: {e['iface']}")
```

## Monitoring ARP Changes

macOS does not have `ip monitor` built in, but you can use `watch` (if installed via Homebrew):

```bash
# Install watch with Homebrew
brew install watch

# Watch ARP table every second
watch -n 1 'arp -an'
```

## Key Differences Between macOS, Linux, and Windows

| Feature | macOS | Linux | Windows |
|---------|-------|-------|---------|
| Primary command | `arp -an` | `ip neigh show` | `arp -a` |
| Legacy command | `arp -a` | `arp -n` | `arp -a` |
| Live monitoring | `watch arp -an` | `ip monitor neigh` | `watch` via tools |
| Static entries | `arp -s IP MAC` | `ip neigh add` | `arp -s IP MAC` |

## Key Takeaways

- Use `arp -an` on macOS to display all ARP entries numerically.
- The `permanent` flag indicates static entries (like broadcast).
- Use `arp -n IP` to look up a specific address.
- macOS ARP cache entries expire based on the OS-managed timeout.

**Related Reading:**

- [How to View the ARP Table on Linux](https://oneuptime.com/blog/post/2026-03-20-view-arp-table-linux/view)
- [How to View the ARP Table on Windows](https://oneuptime.com/blog/post/2026-03-20-view-arp-table-windows/view)
- [How to Clear the ARP Cache](https://oneuptime.com/blog/post/2026-03-20-clear-arp-cache-linux/view)
