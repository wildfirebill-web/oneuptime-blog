# How to Configure Kamailio SIP Proxy with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kamailio, IPv6, SIP Proxy, VoIP, Telephony, Load Balancing

Description: Configure Kamailio SIP proxy server to handle SIP requests over IPv6, including dual-stack support, IPv6 routing logic, and NAT handling considerations.

---

Kamailio is a high-performance SIP server used as a proxy, registrar, and load balancer. It supports IPv6 through listener configuration and provides modules for handling IPv6-specific SIP routing requirements.

## Kamailio IPv6 Listener Configuration

```bash
# /etc/kamailio/kamailio.cfg

# Basic parameters
#!define DBURL "mysql://kamailio:password@localhost/kamailio"

# Listen on IPv6 (UDP and TCP)
listen=udp:[::]:5060
listen=tcp:[::]:5060
listen=tls:[::]:5061

# Also listen on IPv4 for dual-stack
listen=udp:0.0.0.0:5060
listen=tcp:0.0.0.0:5060

# Define aliases for this host
alias=sip.example.com

# IPv6 address definition
#!define MY_IPV6_ADDR "2001:db8::kamailio"
```

## Kamailio Routing for IPv6

```bash
# /etc/kamailio/kamailio.cfg - Request routing

request_route {
    # Handle REGISTER
    if (is_method("REGISTER")) {
        route(REGISTRAR);
    }

    # Route based on IP version
    if (af == INET6) {
        route(IPV6_ROUTING);
    } else {
        route(IPV4_ROUTING);
    }
}

route[REGISTRAR] {
    if (!save("location")) {
        sl_reply_error();
    }
    exit;
}

route[IPV6_ROUTING] {
    # Fix the contact header for IPv6 clients
    if (is_method("INVITE")) {
        # Add Record-Route with IPv6 address
        record_route_preset("[MY_IPV6_ADDR]:5060;transport=udp");
    }

    # Lookup in location database
    if (!lookup("location")) {
        sl_send_reply("404", "Not Found");
        exit;
    }

    t_relay();
    exit;
}

route[IPV4_ROUTING] {
    if (!lookup("location")) {
        sl_send_reply("404", "Not Found");
        exit;
    }
    t_relay();
    exit;
}
```

## IPv6/IPv4 Interworking in Kamailio

```bash
# Handle calls between IPv4 and IPv6 clients
# Kamailio acts as B2BUA or uses RTPEngine for media

# Load B2B User Agent module
loadmodule "b2b_entities.so"
loadmodule "b2b_logic.so"

# Or use RTPEngine for IPv6-IPv4 media transcoding
loadmodule "rtpengine.so"
modparam("rtpengine", "rtpengine_sock", "udp:[2001:db8::rtpengine]:22222")

request_route {
    if (src_ip == "0.0.0.0" && af == INET) {
        # IPv4 source calling IPv6 destination
        rtpengine_offer("force-relay");
    } else if (af == INET6) {
        rtpengine_offer("force-relay");
    }
}
```

## Kamailio IPv6 Registration Handling

```bash
# usrloc module for IPv6 registrations
loadmodule "usrloc.so"
modparam("usrloc", "db_mode", 2)
modparam("usrloc", "use_domain", 1)

# Track IPv6 contacts
route[REGISTRAR] {
    if (!save("location")) {
        sl_reply_error();
    }

    # Log IPv6 registration
    if (af == INET6) {
        xlog("L_INFO", "IPv6 REGISTER from $si:$sp\n");
    }
    exit;
}

# Lookup including IPv6 contacts
route[INVITE] {
    if (!lookup("location")) {
        # Try IPv6 alternative
        sl_send_reply("404", "Not Found");
        exit;
    }
}
```

## Firewall Rules for Kamailio IPv6

```bash
# SIP over IPv6
sudo ip6tables -A INPUT -p udp --dport 5060 -j ACCEPT
sudo ip6tables -A INPUT -p tcp --dport 5060 -j ACCEPT
sudo ip6tables -A INPUT -p tcp --dport 5061 -j ACCEPT  # TLS

# If using RTPEngine
sudo ip6tables -A INPUT -p udp --dport 10000:60000 -j ACCEPT

sudo ip6tables-save > /etc/ip6tables/rules.v6
```

## Testing Kamailio IPv6

```bash
# Check Kamailio is listening on IPv6
sudo kamailio -f /etc/kamailio/kamailio.cfg -l

# Verify listeners
kamcmd> cfg.get_list | grep listen

# Test with SIPp
sudo apt install sipp -y

# Send OPTIONS over IPv6
sipp [2001:db8::kamailio]:5060 \
  -sf OPTIONS.xml \
  -i [2001:db8::test-client] \
  -t un \
  -m 1

# Check Kamailio stats
kamctl stats

# Check registered IPv6 contacts
kamctl ul show | grep "2001:"

# Debug SIP over IPv6
kamailio -E -l [::]:5060
```

Kamailio's `listen=udp:[::]:5060` configuration enables IPv6 SIP proxy operation, with the `af == INET6` condition in routing scripts enabling IPv6-specific logic for record-routing, NAT handling, and media relay decisions for calls between IPv6 and IPv4 SIP clients.
