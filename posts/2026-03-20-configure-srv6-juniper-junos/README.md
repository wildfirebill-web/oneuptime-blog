# How to Configure SRv6 on Juniper Junos

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SRv6, Juniper, Junos, Segment Routing, Traffic Engineering

Description: Configure SRv6 segment routing on Juniper Junos routers, including locator blocks, IS-IS SRv6 extensions, and traffic engineering using SRv6 policies.

## Introduction

Juniper Junos supports SRv6 on MX, PTX, and ACX platforms. Configuration uses the familiar Junos hierarchical CLI. This guide covers locator blocks, IS-IS advertisement, and verifying SRv6 forwarding.

## SRv6 Locator Configuration

```
# Junos SRv6 configuration
set routing-options source-routing
set protocols source-packet-routing srv6
set protocols source-packet-routing srv6 locator MAIN prefix 5f00:2::/48
set protocols source-packet-routing srv6 locator MAIN micro-sid
set protocols source-packet-routing srv6 no-reduced-srh

# Alternatively in full hierarchy:
protocols {
    source-packet-routing {
        srv6 {
            locator MAIN {
                prefix 5f00:2::/48;
                micro-sid;
            }
        }
    }
}
```

## Loopback and Interface Configuration

```
# Assign locator to loopback
set interfaces lo0 unit 0 family inet6 address 5f00:2::/128

# Enable IPv6 on interfaces
set interfaces ge-0/0/0 unit 0 family inet6 address fd00:12::2/64

interfaces {
    ge-0/0/0 {
        unit 0 {
            family inet6 {
                address fd00:12::2/64;
            }
        }
    }
    ge-0/0/1 {
        unit 0 {
            family inet6 {
                address fd00:23::2/64;
            }
        }
    }
}
```

## IS-IS with SRv6 Extensions

```
protocols {
    isis {
        level 2 wide-metrics-only;
        interface ge-0/0/0.0 {
            level 2 metric 10;
        }
        interface ge-0/0/1.0 {
            level 2 metric 10;
        }
        interface lo0.0 {
            passive;
        }
        source-packet-routing {
            srv6 {
                locator MAIN;
                node-segment ipv6-index 2;
            }
        }
    }
}
```

## Verification Commands

```bash
# Show SRv6 locators
show spring-traffic-engineering lsp detail | grep -A5 "SRv6"

# Show SRv6 SIDs
show protocols source-packet-routing srv6 sid

# Show IS-IS SRv6 database
show isis database extensive | grep -A10 "SRv6"

# Show forwarding
show route 5f00:2::/48 detail
show route 5f00:1:0:e001:: detail

# Ping using SRv6 SID
ping 5f00:3:: routing-instance default source 5f00:2::
```

## SRv6 Traffic Engineering Policy

```
# SRv6 TE policy via Spring Traffic Engineering
protocols {
    spring-traffic-engineering {
        srv6 {
            encapsulation-type srv6;
        }
        lsp R2-to-R1-via-R3 {
            to 5f00:1::;
            label-switched-path {
                primary via-R3;
            }
            paths {
                via-R3 {
                    explicit-path {
                        path-segments {
                            segment 5f00:3:0:e001::;
                            segment 5f00:1:0:e000::;
                        }
                    }
                }
            }
        }
    }
}
```

## BGP with SRv6 for L3VPN

```
routing-instances {
    CUSTOMER-A {
        instance-type vrf;
        route-distinguisher 65002:100;
        vrf-target target:65002:100;
        protocols {
            bgp {
                family inet6 {
                    unicast {
                        srv6 {
                            locator MAIN;
                        }
                    }
                }
            }
        }
    }
}
```

## Conclusion

Juniper Junos SRv6 configuration uses the `source-packet-routing` and `spring-traffic-engineering` hierarchies. IS-IS advertises locators and node segments. Use `show spring-traffic-engineering lsp detail` to verify TE policies. Monitor SRv6 path status and BGP session health with OneUptime.
