# How to Understand IPv6 Route Metrics and Preferences

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Routing, Metrics, Administrative Distance, Linux

Description: Understand how IPv6 route metrics and administrative distances control which route is selected when multiple paths exist to the same destination.

## Overview

When multiple routes exist for the same IPv6 prefix, the router must choose one. The selection process uses **metric** (also called administrative distance or preference) — a numeric value where lower numbers indicate more preferred routes.

## Route Metrics in Linux

On Linux, each route has a `metric` field. When multiple routes match a destination, the one with the **lowest metric** wins.

```bash
# Show routes with their metrics
ip -6 route show
# 2001:db8:1::/48 via fe80::1 dev eth0 proto static metric 100
# 2001:db8:1::/48 via fe80::2 dev eth1 proto static metric 200
# The /48 via fe80::1 (metric 100) is preferred

# Verify which route is actually used
ip -6 route get 2001:db8:1::1
# 2001:db8:1::1 via fe80::1 dev eth0  ← Uses metric 100 route
```

## Setting Metrics When Adding Routes

```bash
# Add a primary route with low metric
sudo ip -6 route add 2001:db8:1::/48 via fe80::1 dev eth0 metric 100

# Add a backup (floating) route with high metric
sudo ip -6 route add 2001:db8:1::/48 via fe80::2 dev eth1 metric 500

# The backup only activates if the primary route is removed
# (e.g., if the interface eth0 goes down)
```

## Floating Static Routes (High-Availability)

A floating static route is a backup route with a deliberately high metric that only becomes active when the primary route disappears:

```bash
# Primary route via ISP1 (metric 100)
sudo ip -6 route add default via fe80::isp1 dev eth0 metric 100

# Backup route via ISP2 (metric 500 — floating static)
sudo ip -6 route add default via fe80::isp2 dev eth1 metric 500

# If eth0 goes down, kernel removes the metric-100 route
# The metric-500 backup becomes the active default route
```

## Administrative Distance vs Metric

In Cisco terminology, **administrative distance** determines preference between route sources (not within the same source), while **metric** ranks routes within the same protocol.

| Source | Cisco AD | Linux Equivalent |
|--------|----------|-----------------|
| Connected | 0 | proto kernel, metric 256 |
| Static | 1 | proto static, user-defined |
| OSPF | 110 | proto ospf, metric varies |
| BGP iBGP | 200 | proto bgp |
| RIPng | 120 | proto ripng |

In Linux, the `metric` field plays both roles — it ranks routes regardless of their source.

## Routing Protocol Metrics

Different protocols calculate metrics differently:

```bash
# OSPFv3: metric based on interface cost (default: 10^8 / bandwidth)
# FRRouting: view ospf metrics
vtysh -c "show ipv6 ospf route"
# N 2001:db8:1::/48  [10]  area: 0.0.0.0    ← cost=10

# BGP: metric is the MED (Multi-Exit Discriminator) attribute
vtysh -c "show bgp ipv6 unicast 2001:db8:1::/48"
# MED: 100
```

## Modifying Interface Metrics

```bash
# Change the metric for a Router Advertisement-learned route
# (modify the RA handling metric via interface sysctl)
sudo sysctl -w net.ipv6.conf.eth0.accept_ra_rt_info_max_plen=64
sudo sysctl -w net.ipv6.conf.eth0.router_solicitations=3

# Or set default metric for all RA-learned routes
sudo ip -6 route change default metric 2000
```

## Checking for Multiple Equal-Cost Routes (ECMP)

When two routes have identical prefix length AND metric, Linux installs both and load-balances:

```bash
# Add two equal-cost routes
sudo ip -6 route add 2001:db8:1::/48 via fe80::1 dev eth0 metric 100
sudo ip -6 route add 2001:db8:1::/48 via fe80::2 dev eth1 metric 100

# Both appear in the routing table as ECMP
ip -6 route show 2001:db8:1::/48
# 2001:db8:1::/48 metric 100
#     nexthop via fe80::1 dev eth0 weight 1
#     nexthop via fe80::2 dev eth1 weight 1
```

## Summary

IPv6 route selection on Linux is controlled by the `metric` field — lower values are preferred. Use low metrics for primary paths and high metrics for floating static backup routes. When multiple routes have the same prefix and metric, ECMP load-balancing applies. Always verify route selection with `ip -6 route get <destination>`.
