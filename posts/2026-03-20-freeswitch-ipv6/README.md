# How to Configure FreeSWITCH with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: FreeSWITCH, IPv6, PBX, VoIP, SIP, Telephony, Linux

Description: Configure FreeSWITCH telephony platform to accept SIP connections over IPv6, including SIP profile IPv6 binding, media transport configuration, and firewall setup.

---

FreeSWITCH is a scalable open-source PBX/VoIP platform. Enabling IPv6 support requires configuring SIP profiles to bind to IPv6 addresses and ensuring the media (RTP) subsystem uses IPv6 for media streams.

## FreeSWITCH SIP Profile for IPv6

```xml
<!-- /etc/freeswitch/sip_profiles/internal-ipv6.xml -->

<profile name="internal-ipv6">
  <aliases></aliases>

  <gateways>
  </gateways>

  <domains>
    <domain name="all" alias="true" parse="true"/>
  </domains>

  <settings>
    <!-- Bind to all IPv6 interfaces -->
    <param name="sip-ip" value="::"/>
    <param name="sip-port" value="5060"/>

    <!-- External IPv6 address (for SDP) -->
    <param name="ext-sip-ip" value="2001:db8::freeswitch"/>
    <param name="ext-rtp-ip" value="2001:db8::freeswitch"/>

    <!-- RTP settings -->
    <param name="rtp-ip" value="::"/>
    <param name="rtp-port-min" value="16384"/>
    <param name="rtp-port-max" value="32768"/>

    <!-- Enable IPv6 -->
    <param name="enable-ipv6" value="true"/>
    <param name="force-register-domain" value="$${domain}"/>

    <!-- SIP options -->
    <param name="apply-nat-acl" value="nat.auto"/>
    <param name="user-agent-string" value="FreeSWITCH-IPv6"/>
  </settings>
</profile>
```

## Dual-Stack FreeSWITCH Profile

```xml
<!-- Configure both IPv4 and IPv6 profiles -->

<!-- IPv4 profile: /etc/freeswitch/sip_profiles/internal.xml -->
<param name="sip-ip" value="$${local_ip_v4}"/>
<param name="rtp-ip" value="$${local_ip_v4}"/>

<!-- IPv6 profile: /etc/freeswitch/sip_profiles/internal-ipv6.xml -->
<param name="sip-ip" value="::"/>
<param name="rtp-ip" value="::"/>
<param name="ext-sip-ip" value="2001:db8::freeswitch"/>
```

## FreeSWITCH vars.xml for IPv6

```xml
<!-- /etc/freeswitch/vars.xml -->

<!-- Define IPv6 address variables -->
<X-PRE-PROCESS cmd="set" data="local_ipv6=2001:db8::freeswitch"/>
<X-PRE-PROCESS cmd="set" data="public_ipv6=2001:db8::freeswitch"/>

<!-- Use in profiles -->
<!-- <param name="ext-sip-ip" value="$${local_ipv6}"/> -->
```

## Dialplan for IPv6 Routing

```xml
<!-- /etc/freeswitch/dialplan/default.xml -->

<extension name="route-to-ipv6-gateway">
  <condition field="destination_number" expression="^(\d+)$">
    <action application="bridge"
      data="sofia/internal-ipv6/sip:$1@[2001:db8::sip-gateway]"/>
  </condition>
</extension>

<extension name="receive-ipv6-calls">
  <condition field="${sip_received_ip}" expression="^2001:db8:">
    <action application="answer"/>
    <action application="bridge" data="user/1001@$${domain}"/>
  </condition>
</extension>
```

## ACL Configuration for IPv6

```xml
<!-- /etc/freeswitch/autoload_configs/acl.conf.xml -->

<configuration name="acl.conf" description="Network Lists">
  <network-lists>

    <!-- Allow IPv6 internal network -->
    <list name="ipv6-internal" default="deny">
      <node type="allow" cidr="2001:db8:internal::/48"/>
      <node type="allow" cidr="::1/128"/>
    </list>

    <!-- Trusted SIP IPv6 peers -->
    <list name="ipv6-trusted-peers" default="deny">
      <node type="allow" cidr="2001:db8::sip-gateway/128"/>
      <node type="allow" cidr="2001:db8:carriers::/48"/>
    </list>

  </network-lists>
</configuration>
```

## Firewall Rules for FreeSWITCH IPv6

```bash
# SIP over IPv6

sudo ip6tables -A INPUT -p udp --dport 5060 -j ACCEPT
sudo ip6tables -A INPUT -p tcp --dport 5060 -j ACCEPT
sudo ip6tables -A INPUT -p tcp --dport 5061 -j ACCEPT  # SIP TLS

# RTP media range
sudo ip6tables -A INPUT -p udp --dport 16384:32768 -j ACCEPT

# FreeSWITCH Event Socket (restrict to localhost)
sudo ip6tables -A INPUT -p tcp -s ::1 --dport 8021 -j ACCEPT

sudo ip6tables-save > /etc/ip6tables/rules.v6
```

## Testing FreeSWITCH IPv6

```bash
# Verify SIP profile is listening on IPv6
fs_cli -x "sofia status"

# Check IPv6 profile status
fs_cli -x "sofia status profile internal-ipv6"

# Check registrations from IPv6 clients
fs_cli -x "sofia status profile internal-ipv6 reg"

# Monitor SIP over IPv6
fs_cli -x "sofia global siptrace on"
sudo tail -f /var/log/freeswitch/freeswitch.log | grep "2001:"

# Test call from IPv6 endpoint
fs_cli -x "originate sofia/internal-ipv6/sip:1001@[2001:db8::phone] &echo()"
```

FreeSWITCH's IPv6 support via SIP profile `sip-ip=::` binding enables both SIP signaling and RTP media over IPv6, with the `ext-sip-ip` and `ext-rtp-ip` parameters being critical for correct SDP generation when the server is behind a firewall or NAT-like device.
