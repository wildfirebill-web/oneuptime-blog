# How to Use pathping for IPv4 Network Diagnostics on Windows

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Windows, Networking, Pathping, IPv4, Diagnostics, Network Troubleshooting

Description: Use pathping on Windows to combine the hop-by-hop path discovery of tracert with the packet loss statistics of ping, identifying which specific hops have loss or delay.

## Introduction

`pathping` is a Windows network diagnostic tool that combines the route-tracing capability of `tracert` with the statistical analysis of `ping`. It identifies not just where the path goes, but which specific hops are dropping or delaying packets.

## Running pathping

```cmd
:: Run pathping to a destination
pathping 8.8.8.8

:: Use -n to suppress DNS lookups (faster)
pathping -n 8.8.8.8

:: Force IPv4
pathping -4 google.com
```

## How pathping Works

pathping runs in two phases:
1. **Discovery phase**: traces the route (same as tracert), shows each hop's IP
2. **Statistics phase**: sends 100 pings to each hop and measures loss and latency

## Reading pathping Output

```text
Tracing route to 8.8.8.8
  0  MY-PC [192.168.1.100]
  1  192.168.1.1
  2  10.100.0.1
  3  8.8.8.8

Computing statistics for 75 seconds...
            Source to Here   This Node/Link
Hop  RTT    Lost/Sent = Pct  Lost/Sent = Pct  Address
  0                                            MY-PC [192.168.1.100]
                                0/ 100 =  0%   |
  1    1ms     0/ 100 =  0%     0/ 100 =  0%  192.168.1.1
                                0/ 100 =  0%   |
  2    8ms     0/ 100 =  0%     0/ 100 =  0%  10.100.0.1
                                0/ 100 =  0%   |
  3   15ms     0/ 100 =  0%     0/ 100 =  0%  8.8.8.8
```

| Column | Meaning |
|---|---|
| RTT | Round-trip time from source to this hop |
| Lost/Sent (Source to Here) | Cumulative packet loss from source to this hop |
| Lost/Sent (This Node/Link) | Loss attributable to this specific hop or link |

## Interpreting Loss

- **High "This Node/Link" loss at a specific hop**: that router or the link entering it is dropping packets
- **High "Source to Here" loss that stays constant hop-to-hop**: loss happened earlier, not at this hop
- **`*` in RTT**: that hop doesn't respond to ICMP (but check if path continues)

## Useful pathping Options

```cmd
:: Increase the number of queries per hop (default 100)
pathping -q 200 8.8.8.8

:: Set maximum hop count
pathping -h 20 8.8.8.8

:: Set timeout per probe (milliseconds, default 3000)
pathping -w 1000 8.8.8.8

:: Set probe interval (milliseconds, default 250)
pathping -p 500 8.8.8.8
```

## pathping vs. tracert vs. ping

| Tool | Discovers Path | Measures Loss | Per-Hop Statistics |
|---|---|---|---|
| ping | No | Global only | No |
| tracert | Yes | No | Latency only |
| pathping | Yes | Per-hop | Yes |
| mtr (Linux) | Yes | Per-hop | Yes (continuously) |

## Linux Equivalent

`mtr` provides similar functionality on Linux with a continuous, updating display:

```bash
mtr -n 8.8.8.8
```

## Conclusion

`pathping -n` is the most informative single command for diagnosing Windows network issues that involve packet loss at specific hops. The statistics phase takes about 75 seconds but provides definitive per-hop loss data that neither `ping` nor `tracert` can supply.
