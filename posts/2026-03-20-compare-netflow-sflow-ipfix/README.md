# How to Compare NetFlow vs sFlow vs IPFIX for Your Network

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: NetFlow, sFlow, IPFIX, Traffic Analysis, Network Monitoring, Comparison

Description: Compare NetFlow, sFlow, and IPFIX to understand their technical differences, vendor support, accuracy trade-offs, and which is best suited for your monitoring needs.

## The Three Flow Protocols

| Feature | NetFlow v5/v9 | sFlow | IPFIX |
|---|---|---|---|
| Origin | Cisco (proprietary) | InMon Corp (open) | IETF Standard (RFC 7011) |
| Architecture | Flow cache + export | Packet sampling | Template-based |
| Accuracy | High (all flows) | Statistical (sampled) | High (all flows) |
| CPU overhead | Medium-High | Low | Medium |
| Link speed limit | ~10G (cache-limited) | Any speed | ~10G+ |
| Vendor support | Cisco, others | Most vendors | All modern vendors |
| Port | 2055 (UDP) | 6343 (UDP) | 4739 (UDP/TCP) |
| IPv6 support | v9+ only | Yes | Yes |
| MPLS visibility | v9 only | Yes | Yes |
| Customizable fields | v9/FNF | Packet header | Yes (templates) |

## NetFlow: Best for Deep Cisco Visibility

NetFlow tracks every flow in hardware or software. The router maintains a flow cache and exports records when flows expire.

**Strengths:**
- Complete per-flow data (every packet counted)
- Rich metadata: AS numbers, BGP nexthop, MPLS labels (v9)
- Exact bandwidth accounting

**Weaknesses:**
- Higher CPU/memory on router due to flow cache
- Doesn't scale well above 10G+ without sampling
- v5 fixed format, v9 better but still Cisco-centric

```bash
# NetFlow v5 verification

snmpget -v2c -c public router.example.com \
  .1.3.6.1.4.1.9.9.387.1.1.1.0   # ciscoIpCef OID - shows cache stats
```

## sFlow: Best for High-Speed Links

sFlow samples 1 in N packets and immediately forwards the sample-no flow cache needed. This makes it scale to any line rate.

**Strengths:**
- Works at 40G, 100G, 400G interfaces
- Low router CPU overhead
- Includes raw packet headers (can see application data in samples)
- Wide vendor support (Arista, Juniper, HP, Brocade, Linux)

**Weaknesses:**
- Statistical-accuracy depends on sampling rate
- Small flows may be missed at low sampling rates
- Less detailed than NetFlow at equivalent data rates

```bash
# Verify sFlow is sending on port 6343
sudo tcpdump -i eth0 udp port 6343 -n -c 3

# Check sample rate (1 in X packets) is appropriate
# For 10G: sample 1:2000 gives ~5000 samples/sec at 10Gbps line rate
```

## IPFIX: Best for Vendor-Neutral Environments

IPFIX is the IETF-standardized version of NetFlow v9. It uses the same template mechanism but with IANA-assigned element IDs.

**Strengths:**
- Open standard-works across Cisco, Juniper, Palo Alto, F5, OVS
- Enterprise-specific extensions possible
- TCP transport option for reliable delivery
- All modern platforms support it

**Weaknesses:**
- Similar CPU overhead to NetFlow (flow cache required)
- Collector must handle template negotiation

```bash
# IPFIX export uses port 4739 by default (IANA-assigned)
sudo tcpdump -i eth0 udp port 4739 -n -c 3
```

## When to Use Each Protocol

### Use NetFlow v9/FNF when:
- Your environment is predominantly Cisco
- You need exact flow accounting (billing, SLA verification)
- Links are under 10 Gbps
- You need BGP AS-path or MPLS label visibility

### Use sFlow when:
- You have mixed-vendor switches (HP, Arista, Cumulus)
- Links exceed 10 Gbps (40G, 100G data center)
- You need raw packet header data in samples
- Low router CPU overhead is critical

### Use IPFIX when:
- Multi-vendor environment (Cisco + Juniper + Palo Alto)
- You need standards compliance and future-proofing
- You want TCP transport for reliable delivery
- You're exporting from Linux (Open vSwitch, Linux TC)

## Collector Compatibility Matrix

| Collector | NetFlow v5 | NetFlow v9 | IPFIX | sFlow |
|---|---|---|---|---|
| nfdump/nfcapd | Yes | Yes | Yes | No |
| ElastiFlow | Yes | Yes | Yes | Yes |
| ntopng | Yes | Yes | Yes | Yes |
| Grafana (via Telegraf) | Yes | Yes | Yes | Yes |
| PMacct | Yes | Yes | Yes | Yes |

## Conclusion

Choose NetFlow for Cisco-centric environments where per-flow accuracy matters; choose sFlow for high-speed links or mixed-vendor networks; choose IPFIX for standards-based multi-vendor deployments. In practice, most modern monitoring platforms (ElastiFlow, ntopng) support all three, so you can collect all types and let the collector normalize them into a unified dataset.
