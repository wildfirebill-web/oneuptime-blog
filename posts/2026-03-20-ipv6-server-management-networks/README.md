# How to Configure IPv6 for Server Management Networks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Server Management, Management Network, SNMP, SSH, Monitoring, Out-of-Band

Description: Configure IPv6 on server management networks including dedicated management VLANs with IPv6, SSH access via IPv6, SNMP monitoring, and management plane security.

---

Server management networks provide out-of-band access to server operating systems for configuration, monitoring, and recovery. IPv6 on management networks enables dual-stack or IPv6-only management access while maintaining strict security policies to protect management plane traffic.

## Management Network IPv6 Design

```text
Server Management IPv6 Architecture:

Management Network: 2001:db8:mgmt::/48
  In-Band Management VLAN: 2001:db8:mgmt:0010::/64
  Out-of-Band (OOB):        2001:db8:mgmt:0020::/64
  IPMI/BMC:                 2001:db8:mgmt:0030::/64

Server Management Addressing:
  Server-01 eth0 (OS): 2001:db8:mgmt:10::101/64
  Server-01 IPMI/BMC:  2001:db8:mgmt:30::101/64
  Server-01 OOB NIC:   2001:db8:mgmt:20::101/64

Encoding: .:VVVV::NNNN where VVVV=VLAN, NNNN=server number
```

## Configure IPv6 on Linux Management Interface

```bash
# Server in-band management IPv6 configuration

# Ubuntu/Debian - /etc/netplan/01-mgmt.yaml

network:
  ethernets:
    eth0:
      dhcp4: false
      dhcp6: false
      addresses:
        - "192.168.10.101/24"
        - "2001:db8:mgmt:10::101/64"
      gateway6: "2001:db8:mgmt:10::1"
      nameservers:
        addresses: [2001:4860:4860::8888, 8.8.8.8]

# Apply
sudo netplan apply

# Verify
ip -6 addr show eth0
ip -6 route show default

# RHEL/CentOS - /etc/sysconfig/network-scripts/ifcfg-eth0
# IPV6INIT=yes
# IPV6ADDR=2001:db8:mgmt:10::101/64
# IPV6_DEFAULTGW=2001:db8:mgmt:10::1
```

## SSH via IPv6 Management

```bash
# /etc/ssh/sshd_config - Allow SSH on IPv6 management address
ListenAddress 2001:db8:mgmt:10::101
ListenAddress 192.168.10.101
# Or listen on all:
# ListenAddress ::

# Restrict SSH to management subnet only
AllowUsers admin@2001:db8:mgmt::/48
PermitRootLogin no

# Restart SSH
sudo systemctl restart sshd

# Test SSH via IPv6
ssh -6 admin@2001:db8:mgmt:10::101
# Or via hostname (with AAAA DNS record)
ssh admin@server-01.mgmt.example.com
```

## Management Firewall (ip6tables)

```bash
#!/bin/bash
# ipv6-mgmt-firewall.sh - Secure management network

MGMT_NETWORK="2001:db8:mgmt::/48"
MGMT_IFACE="eth0"
ALLOW_MANAGEMENT_HOSTS="2001:db8:mgmt:10::10"  # Jump server

# Flush existing
ip6tables -F INPUT
ip6tables -F OUTPUT
ip6tables -F FORWARD

# Default policy: drop
ip6tables -P INPUT DROP
ip6tables -P FORWARD DROP
ip6tables -P OUTPUT ACCEPT

# Allow loopback
ip6tables -A INPUT -i lo -j ACCEPT

# Allow ICMPv6 (NDP, ping - required)
ip6tables -A INPUT -p icmpv6 -j ACCEPT

# Allow established/related
ip6tables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow SSH only from management jump server
ip6tables -A INPUT -p tcp --dport 22 \
    -s $ALLOW_MANAGEMENT_HOSTS -j ACCEPT

# Allow SNMP from monitoring server
ip6tables -A INPUT -p udp --dport 161 \
    -s 2001:db8:mgmt:10::50 -j ACCEPT

# Allow HTTPS management (web UI)
ip6tables -A INPUT -p tcp --dport 443 \
    -s $MGMT_NETWORK -j ACCEPT

# Log and drop everything else
ip6tables -A INPUT -j LOG --log-prefix "MGMT-DROP: " --log-level 4
ip6tables -A INPUT -j DROP

# Save rules
ip6tables-save > /etc/ip6tables/rules.v6
echo "Management IPv6 firewall configured"
```

## SNMP Monitoring via IPv6

```bash
# /etc/snmp/snmpd.conf - SNMP listening on IPv6

agentAddress udp6:[2001:db8:mgmt:10::101]:161
agentAddress udp:[0.0.0.0]:161

# SNMPv3 user for monitoring
createUser MONITOR SHA "authpass123" AES "privpass456"
rouser MONITOR priv

# Trap destination (IPv6 monitoring server)
trap6sink 2001:db8:mgmt:10::50 MONITOR 162

# System info
sysLocation "DC1-Rack1-U10"
sysContact "noc@example.com"
sysName "server-01.dc1.example.com"
```

## Centralized Management via IPv6

```yaml
# Ansible inventory with IPv6 management addresses
# /etc/ansible/hosts

[servers]
server-01 ansible_host=2001:db8:mgmt:10::101 ansible_user=admin
server-02 ansible_host=2001:db8:mgmt:10::102 ansible_user=admin

[all:vars]
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_extra_args='-6'
```

```bash
# Ansible IPv6 playbook example
cat > /tmp/ipv6-mgmt-test.yaml << 'EOF'
---
- hosts: servers
  tasks:
    - name: Check IPv6 management interface
      command: ip -6 addr show eth0
      register: ipv6_result

    - name: Show IPv6 addresses
      debug:
        msg: "{{ ipv6_result.stdout_lines }}"

    - name: Test IPv6 DNS resolution
      command: dig AAAA example.com
      register: dns_result
EOF

ansible-playbook -i /etc/ansible/hosts /tmp/ipv6-mgmt-test.yaml
```

## DNS for IPv6 Management

```bash
# /etc/bind/zones/mgmt.db - AAAA records for management hosts
$ORIGIN mgmt.example.com.
server-01  IN  AAAA  2001:db8:mgmt:10::101
server-02  IN  AAAA  2001:db8:mgmt:10::102
gateway    IN  AAAA  2001:db8:mgmt:10::1
monitor    IN  AAAA  2001:db8:mgmt:10::50

# Reverse DNS (/etc/bind/zones/ipv6-mgmt.rev)
# 1.0.1.0.0.0.0.0.0.1.0.t.g.m.8.b.d.0.1.0.0.2.ip6.arpa.
# → server-01.mgmt.example.com.
```

IPv6 server management networks benefit from structured addressing that embeds server numbers into the IPv6 address for predictability, strict firewall rules allowing SSH and SNMP only from authorized jump hosts and monitoring systems, and DNS AAAA records for all management interfaces to enable hostname-based access and monitoring tool configuration.
