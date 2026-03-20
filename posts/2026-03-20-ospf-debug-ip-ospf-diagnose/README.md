# How to Use debug ip ospf to Diagnose OSPF Problems

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OSPF, Debug, Troubleshooting, FRR, Cisco, Routing, IPv4, Diagnostics

Description: Learn how to use OSPF debug commands to diagnose adjacency failures, LSA flooding issues, and SPF recalculation problems on FRR and Cisco routers.

---

When `show` commands don't reveal the cause of an OSPF problem, debug commands provide real-time visibility into OSPF packet processing, neighbor state transitions, and database changes. Use them judiciously in production - high OSPF activity can generate large volumes of debug output.

## Debug Safety Guidelines

- Always redirect debug output to a log file or buffer, not just the terminal.
- Enable debugging at the most specific level possible.
- Disable debugging immediately after capturing the relevant output.
- On busy routers, use ACLs or filters to limit debug output to specific neighbors.

## FRR Debug Commands

```bash
# Access FRR CLI

vtysh

# Enable OSPF Hello debugging (neighbor state transitions)
conf t
  debug ospf hello
  debug ospf event

# Or use the shell command directly
vtysh -c "debug ospf packet hello recv detail"

# Disable all OSPF debugging
vtysh -c "no debug ospf all"
```

### FRR Debug Options

| Command | What It Shows |
|---------|--------------|
| `debug ospf hello` | Hello packet transmit/receive |
| `debug ospf event` | State machine transitions |
| `debug ospf packet all recv` | All received OSPF packets |
| `debug ospf packet lsa` | LSA flooding events |
| `debug ospf lsa install` | LSA installation in LSDB |
| `debug ospf spf` | SPF calculation triggers |
| `debug ospf nsm` | Neighbor state machine |

### Viewing FRR Debug Output

```bash
# FRR logs to syslog or a dedicated file
tail -f /var/log/frr/frr.log | grep -i ospf

# Or view in syslog
journalctl -f -u frr | grep -i ospf
```

## Common OSPF Problems and Debug Commands

### Problem 1: Neighbors Won't Form (stuck in Init)

```bash
# Enable hello debugging to see if hellos are being received
debug ospf hello
debug ospf packet hello recv detail

# Look for:
# - "Packet from X.X.X.X network address mismatch" → subnet mask issue
# - "Packet from X.X.X.X mismatched hello/dead interval" → timer mismatch
# - No output at all → connectivity or ACL issue
```

### Problem 2: Neighbor Drops Repeatedly

```bash
debug ospf nsm    # Neighbor State Machine
# Look for repeated: NSM[X.X.X.X:Y] event KillNbr
# This indicates dead timer expiry → check interface keepalives
```

### Problem 3: Routes Not Appearing

```bash
debug ospf lsa install
# Verify LSAs are being installed into the LSDB

debug ospf spf
# Verify SPF is recalculating when LSAs change
```

## Cisco IOS Debug Commands

```text
! Enable OSPF debugging
debug ip ospf hello
debug ip ospf events
debug ip ospf database
debug ip ospf adj

! Filter to a specific neighbor (reduces output volume)
debug ip ospf hello detail

! Disable all OSPF debug
no debug ip ospf all
undebug all
```

### Log to Buffer Instead of Terminal

```text
! Set a large buffer for debug output
logging buffered 1048576 debugging

! View the buffer
show logging
```

## Key Takeaways

- Use `debug ospf hello` first - most adjacency problems are visible in hello packet processing.
- On FRR, check `/var/log/frr/frr.log` for debug output; on Cisco, use `show logging` after `logging buffered`.
- Always disable debugging with `no debug ospf all` / `undebug all` after diagnosis.
- For production routers, consider using `show` commands and OSPF event logging instead of interactive debug to minimize CPU impact.
