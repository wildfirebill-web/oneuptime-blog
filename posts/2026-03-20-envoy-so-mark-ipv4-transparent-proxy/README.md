# How to Use SO_MARK Socket Option in Envoy for IPv4 Transparent Proxying

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Envoy, SO_MARK, IPv4, Transparent Proxy, iptables, Linux, Networking

Description: Learn how to configure Envoy's SO_MARK socket option to mark outbound IPv4 packets for policy-based routing in transparent proxy deployments.

---

`SO_MARK` sets a mark (an integer tag) on packets originating from a socket. Combined with Linux policy routing (`ip rule`) and `iptables`, this allows Envoy to route its own outbound packets differently from other processes — a key technique for transparent proxy deployments.

## Why SO_MARK Matters for Transparent Proxying

In a transparent proxy, intercepted traffic must not be re-intercepted by the same iptables rules that captured it in the first place (causing an infinite loop). `SO_MARK` solves this:

1. `iptables` redirects all traffic to Envoy.
2. Envoy sets `SO_MARK` on its outbound sockets.
3. A policy routing rule routes marked packets directly, bypassing the iptables redirect.

## Configuring SO_MARK in Envoy

```yaml
# envoy-config.yaml
static_resources:
  clusters:
    - name: original_dst_cluster
      type: ORIGINAL_DST       # Use the original destination IP (before interception)
      connect_timeout: 5s
      upstream_bind_config:
        socket_options:
          - description: "SO_MARK for transparent proxy bypass"
            level: 1           # SOL_SOCKET = 1
            name: 36           # SO_MARK = 36
            int_value: 100     # Mark value (must match ip rule fwmark)
            state: STATE_PREBIND

  listeners:
    - name: transparent_listener
      address:
        socket_address:
          address: 0.0.0.0
          port_value: 15001
      listener_filters:
        # Recover the original destination address before interception
        - name: envoy.filters.listener.original_dst
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.filters.listener.original_dst.v3.OriginalDst
      filter_chains:
        - filters:
            - name: envoy.filters.network.tcp_proxy
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy
                stat_prefix: transparent
                cluster: original_dst_cluster

admin:
  address:
    socket_address: { address: 127.0.0.1, port_value: 9901 }
```

## iptables Rules to Redirect Traffic to Envoy

```bash
# Redirect all outbound IPv4 TCP traffic to Envoy's listener port
# except packets already marked with fwmark 100 (Envoy's own traffic)
iptables -t nat -A OUTPUT -p tcp -m mark ! --mark 100 -j REDIRECT --to-port 15001

# For inbound traffic interception (in-pod transparent proxy like Istio)
iptables -t nat -A PREROUTING -p tcp -m mark ! --mark 100 -j REDIRECT --to-port 15001
```

## Policy Routing to Bypass Re-Interception

```bash
# Route packets marked with 100 via a different routing table (table 100)
ip rule add fwmark 100 table 100

# Table 100: send marked packets directly to the default gateway
ip route add default via 192.168.1.1 table 100
```

## Granting Envoy the NET_ADMIN Capability

`SO_MARK` requires the `CAP_NET_ADMIN` capability.

```bash
# For a system Envoy binary
setcap cap_net_admin+ep /usr/local/bin/envoy

# For Kubernetes (add to the pod's securityContext)
# securityContext:
#   capabilities:
#     add: ["NET_ADMIN"]
```

## Key Takeaways

- `SO_MARK` sets a per-socket packet mark; use mark value `100` (or any non-zero integer) and reference it in `ip rule`.
- The mark bypasses iptables redirect rules for Envoy's own outbound connections, preventing routing loops.
- Use `ORIGINAL_DST` cluster type with `original_dst` listener filter to forward to the true destination.
- Envoy requires `CAP_NET_ADMIN` to set `SO_MARK`.
