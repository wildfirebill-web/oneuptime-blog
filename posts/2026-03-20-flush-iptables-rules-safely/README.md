# How to Flush All iptables Rules Without Locking Yourself Out

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: iptables, Linux, Firewall, Security, Safety

Description: Safely flush all iptables rules on a remote server without losing SSH access, using the correct sequence of commands to avoid lockout.

Flushing iptables rules (`iptables -F`) without changing the default policy first can permanently lock you out of a remote server. The safe approach sets policies to ACCEPT before flushing.

## The Dangerous Way (DON'T DO THIS REMOTELY)

```bash
# DANGEROUS if default policy is DROP:

sudo iptables -F     # Flushes rules
# At this point, if policy was DROP, all new connections are blocked
# If your SSH connection closes, you're locked out!

# Also dangerous:
sudo iptables -P INPUT DROP   # Without first ensuring SSH is allowed
```

## The Safe Flush Sequence

```bash
#!/bin/bash
# safe-flush.sh - Flush iptables safely on remote servers

# Step 1: Set all policies to ACCEPT FIRST (before flushing)
# This ensures traffic flows even with no rules
sudo iptables -P INPUT ACCEPT
sudo iptables -P FORWARD ACCEPT
sudo iptables -P OUTPUT ACCEPT

# Step 2: Now safe to flush all chains
sudo iptables -F         # Flush all rules in filter table
sudo iptables -X         # Delete custom chains
sudo iptables -Z         # Zero packet/byte counters

# Step 3: Flush NAT table too if needed
sudo iptables -t nat -F
sudo iptables -t nat -X

# Step 4: Flush mangle table
sudo iptables -t mangle -F
sudo iptables -t mangle -X

echo "All iptables rules flushed safely"
sudo iptables -L -n -v
```

## Using a Timed Reset (Emergency Escape Hatch)

For risky changes, schedule a flush-and-reset that runs in 5 minutes:

```bash
# Create an escape hatch: if something goes wrong, this will restore access
echo "sudo iptables-restore < /etc/iptables/rules.v4.backup" | sudo at now + 5 minutes

# Or simpler: schedule a flush to run in 5 minutes
echo "sudo iptables -P INPUT ACCEPT; sudo iptables -F" | sudo at now + 5 minutes

# Now make your changes
# If you get locked out, wait 5 minutes - the scheduled job will restore access

# If everything works, cancel the scheduled job
sudo atq                     # List scheduled jobs
sudo atrm <job-number>       # Cancel specific job
```

## Flush Only Specific Tables

```bash
# Flush only filter table (most common)
sudo iptables -F INPUT
sudo iptables -F OUTPUT
sudo iptables -F FORWARD

# Flush only NAT rules (doesn't affect packet filtering)
sudo iptables -t nat -F

# Flush only a specific chain
sudo iptables -F INPUT    # Keep OUTPUT and FORWARD rules intact
```

## Complete Reset to Default State

To fully reset iptables to defaults (as if freshly installed):

```bash
#!/bin/bash
# reset-iptables.sh - Complete reset to open state

# Set policies to ACCEPT first
sudo iptables -P INPUT ACCEPT
sudo iptables -P FORWARD ACCEPT
sudo iptables -P OUTPUT ACCEPT

# Flush filter table
sudo iptables -F
sudo iptables -X

# Flush NAT table
sudo iptables -t nat -F
sudo iptables -t nat -X

# Flush mangle table
sudo iptables -t mangle -F
sudo iptables -t mangle -X

# Flush raw table
sudo iptables -t raw -F
sudo iptables -t raw -X

# Clear ipsets if used
command -v ipset >/dev/null && sudo ipset flush

echo "iptables completely reset to open state"
```

## Verify the Result

```bash
# After flush, verify all chains are empty with ACCEPT policy
sudo iptables -L -n -v

# Expected output:
# Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
# target prot opt source destination
# (empty)
#
# Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
# (empty)
#
# Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
# (empty)
```

The golden rule: always set policies to ACCEPT before flushing rules - this one habit prevents the most common and most damaging iptables mistakes on remote servers.
