# How to Configure IPv6 for Constrained IoT Devices

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, IoT, Constrained Devices, 6LoWPAN, Embedded, Networking

Description: Configure IPv6 on constrained IoT devices using lightweight stacks like Contiki-NG and RIOT OS, including address configuration and CoAP-based communication.

## Introduction

Constrained IoT devices - microcontrollers with kilobytes of RAM and limited CPU - cannot run full Linux network stacks. Lightweight IPv6 implementations like those in Contiki-NG, RIOT OS, and Zephyr RTOS bring IPv6 to these devices through optimized stacks and 6LoWPAN compression.

## What "Constrained" Means

RFC 7228 defines three device classes:
- **Class 0**: < 10KB RAM, < 100KB flash (no IP stack possible, gateway translates)
- **Class 1**: ~10KB RAM, ~100KB flash (minimal IPv6 with 6LoWPAN)
- **Class 2**: ~50KB RAM, ~250KB flash (full IPv6 with TLS)

## Contiki-NG IPv6 Configuration

Contiki-NG is one of the most popular IoT operating systems with native 6LoWPAN and IPv6:

```c
// project-conf.h - Contiki-NG IPv6 configuration

// Enable 6LoWPAN
#define NETSTACK_CONF_NETWORK sicslowpan_driver

// Enable IPv6
#define UIP_CONF_IPV6 1

// Reduce TCP/IP stack size for constrained devices
#define UIP_CONF_BUFFER_SIZE 140
#define UIP_CONF_RECEIVE_WINDOW 60
#define NBR_TABLE_CONF_MAX_NEIGHBORS 8
#define UIP_CONF_MAX_ROUTES 8

// Enable RPL routing (for mesh networks)
#define UIP_CONF_IPV6_RPL 1
```

## RIOT OS IPv6 Configuration

```c
// Makefile for RIOT OS application with IPv6

BOARD = samr21-xpro    // IEEE 802.15.4 board
USEMODULE += gnrc_ipv6_default
USEMODULE += gnrc_udp
USEMODULE += gnrc_sixlowpan_full
USEMODULE += auto_init_gnrc_netif
USEMODULE += gnrc_rpl

// In application code:
#include "net/gnrc.h"
#include "net/ipv6/addr.h"

// Get interface and its IPv6 address
kernel_pid_t iface_pid = KERNEL_PID_UNDEF;
gnrc_netif_t *netif = gnrc_netif_iter(NULL);
ipv6_addr_t addr;
gnrc_netapi_get(netif->pid, NETOPT_IPV6_ADDR, 0, &addr, sizeof(addr));
```

## Zephyr RTOS IPv6 Configuration

```ini
# prj.conf - Zephyr IPv6 configuration

CONFIG_NETWORKING=y
CONFIG_NET_IPV6=y
CONFIG_NET_IPV6_NBR_CACHE=y
CONFIG_NET_IPV6_MLD=y
CONFIG_NET_UDP=y
CONFIG_NET_TCP=y
CONFIG_NET_SOCKETS=y
CONFIG_NET_SHELL=y

# 6LoWPAN for IEEE 802.15.4
CONFIG_NET_L2_IEEE802154=y
CONFIG_NET_L2_IEEE802154_SECURITY=y
CONFIG_IEEE802154_CC1200=y
```

```c
// Zephyr application code - get IPv6 address
#include <net/net_if.h>
#include <net/net_ip.h>

struct net_if *iface = net_if_get_default();
struct net_if_config *config = net_if_get_config(iface);

// Print global IPv6 address
for (int i = 0; i < NET_IF_MAX_IPV6_ADDR; i++) {
    struct net_if_addr *addr = &config->ip.ipv6->unicast[i];
    if (addr->is_used && addr->addr_type == NET_ADDR_AUTOCONF) {
        char buf[NET_IPV6_ADDR_LEN];
        net_addr_ntop(AF_INET6, &addr->address.in6_addr, buf, sizeof(buf));
        printk("IPv6: %s\n", buf);
    }
}
```

## Sending a CoAP Request from a Constrained Device

```c
// Example using libcoap on a constrained Zephyr device
#include <net/coap.h>

#define COAP_SERVER_ADDR "2001:db8:1:1::100"
#define COAP_SERVER_PORT 5683

int send_sensor_data(int temperature) {
    // Create a CoAP POST request
    struct coap_packet request;
    uint8_t data[128];

    coap_packet_init(&request, data, sizeof(data),
                     COAP_VERSION_1, COAP_TYPE_CON,
                     0, NULL, COAP_METHOD_POST, coap_next_id());

    coap_packet_append_option(&request, COAP_OPTION_URI_PATH, "sensor", 6);

    char payload[32];
    snprintf(payload, sizeof(payload), "{\"temp\":%d}", temperature);
    coap_packet_append_payload_marker(&request);
    coap_packet_append_payload(&request, payload, strlen(payload));

    // Send to server
    // (socket send to COAP_SERVER_ADDR:COAP_SERVER_PORT)
    return 0;
}
```

## Address Assignment on Constrained Devices

Constrained devices typically use one of:
1. **EUI-64 from MAC**: Auto-derived, no SLAAC needed (but privacy risk)
2. **SLAAC from RA**: Configured from router advertisement prefix
3. **DHCPv6**: Full address assignment (heavier stack)
4. **Static**: Hard-coded for very small class 0/1 devices

```c
// In RIOT OS: set a manual IPv6 address for class 0 devices
// (in application init code)
gnrc_netif_t *netif = gnrc_netif_iter(NULL);
ipv6_addr_t addr;
ipv6_addr_from_str(&addr, "2001:db8:1:1::sensor1");
gnrc_netif_ipv6_addr_add(netif, &addr, 64, GNRC_NETIF_IPV6_ADDRS_FLAGS_STATE_VALID);
```

## Conclusion

Configuring IPv6 on constrained IoT devices is handled by lightweight IPv6 stacks in embedded RTOS environments like RIOT OS, Contiki-NG, and Zephyr. The key configurations include enabling the IPv6 module, selecting 6LoWPAN as the network adaptation layer, and choosing an address assignment strategy appropriate for the device class. Higher-level protocols like CoAP then use IPv6/UDP for efficient, RESTful communication over the constrained network.
