# How to Configure SNMPv3 over IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SNMPv3, IPv6, Network Monitoring, Security, Authentication, Encryption

Description: Configure SNMPv3 with authentication and privacy over IPv6 transport, providing secure SNMP monitoring of network devices and servers on IPv6 networks.

---

SNMPv3 adds authentication and encryption to SNMP, addressing the security weaknesses of community-based SNMPv1 and SNMPv2c. Deploying SNMPv3 over IPv6 transport provides both secure SNMP communication and IPv6 network compatibility.

## SNMPv3 Security Models

SNMPv3 defines three security levels:
- **noAuthNoPriv** - No authentication, no encryption (not recommended)
- **authNoPriv** - Authentication (MD5/SHA), no encryption
- **authPriv** - Authentication + encryption (AES/DES)

## Configuring SNMPv3 Agent on Linux

```bash
# Stop snmpd before creating users
sudo systemctl stop snmpd

# Create SNMPv3 user with SHA authentication and AES privacy
sudo net-snmp-create-v3-user \
  -ro \
  -A "SecureAuthPass123!" \
  -a SHA \
  -X "SecurePrivPass123!" \
  -x AES \
  ipv6monitor

# Verify user was added to /var/lib/snmp/snmpd.conf
sudo cat /var/lib/snmp/snmpd.conf | grep ipv6monitor

# Configure /etc/snmp/snmpd.conf
cat >> /etc/snmp/snmpd.conf << 'EOF'
# SNMPv3 user with read-only access
rouser ipv6monitor priv

# Listen on IPv6
agentAddress udp:161,udp6:161
EOF

sudo systemctl start snmpd
```

## SNMPv3 User Configuration File

```bash
# /etc/snmp/snmpd.conf - complete SNMPv3 over IPv6 config

# Agent binding
agentAddress udp6:161

# SNMPv3 user definitions (auto-generated in /var/lib/snmp/snmpd.conf)
# createUser ipv6monitor SHA "AuthPassword" AES "PrivPassword"

# Access control
rouser ipv6monitor priv 1.3.6.1   # Read-only access to full MIB tree
rwuser ipv6admin  priv             # Read-write for admin user

# System location
sysLocation     "Server Room A"
sysContact      noc@example.com

# View definition for restricted access
view   systemonly  included   .1.3.6.1.2.1.1
view   systemonly  included   .1.3.6.1.2.1.25.1
rouser ipv6readonly priv systemonly
```

## Querying with SNMPv3 over IPv6

```bash
# SNMPv3 GET over IPv6
snmpget \
  -v3 \
  -l authPriv \
  -u ipv6monitor \
  -a SHA \
  -A "SecureAuthPass123!" \
  -x AES \
  -X "SecurePrivPass123!" \
  udp6:[2001:db8::server]:161 \
  sysDescr.0

# SNMPv3 WALK over IPv6
snmpwalk \
  -v3 \
  -l authPriv \
  -u ipv6monitor \
  -a SHA -A "SecureAuthPass123!" \
  -x AES -X "SecurePrivPass123!" \
  udp6:[2001:db8::server]:161 \
  interfaces

# Save SNMPv3 credentials to ~/.snmp/snmp.conf for convenience
cat > ~/.snmp/snmp.conf << 'EOF'
defSecurityName ipv6monitor
defAuthType SHA
defAuthPassphrase SecureAuthPass123!
defPrivType AES
defPrivPassphrase SecurePrivPass123!
defSecurityLevel authPriv
defVersion 3
EOF

# Now queries are simpler
snmpget udp6:[2001:db8::server]:161 sysDescr.0
```

## SNMPv3 on Cisco Devices over IPv6

```
! Cisco IOS - SNMPv3 over IPv6

! Create SNMPv3 user group
snmp-server group MONITORGROUP v3 priv

! Create SNMPv3 user
snmp-server user ipv6monitor MONITORGROUP v3 \
  auth sha AuthPassword123 \
  priv aes 128 PrivPassword123

! Configure SNMP trap to IPv6 NMS
snmp-server host 2001:db8::nms version 3 priv ipv6monitor

! Verify
show snmp user
show snmp group
```

## SNMPv3 on Juniper Devices over IPv6

```
# Juniper JunOS - SNMPv3

set snmp v3 usm local-engine user ipv6monitor \
  authentication-sha authentication-password "AuthPassword123"
set snmp v3 usm local-engine user ipv6monitor \
  privacy-aes128 privacy-password "PrivPassword123"

set snmp v3 vacm access group READONLY \
  default-context-prefix security-model usm \
  security-level privacy read-view all

set snmp trap-group monitors targets 2001:db8::nms
set snmp trap-group monitors version v3

commit
```

## Monitoring SNMPv3 Activity

```bash
# Capture SNMPv3 traffic (encrypted but visible structure)
sudo tcpdump -i eth0 -nn "udp port 161" -v

# Log SNMPv3 queries in snmpd
# Add to /etc/snmp/snmpd.conf:
# log_in_msg 3

sudo journalctl -u snmpd -f

# Audit SNMPv3 access
grep "ipv6monitor" /var/log/syslog | tail -20
```

SNMPv3 over IPv6 combines the security benefits of message authentication and encryption with IPv6's larger address space and improved routing, providing a secure monitoring channel for network infrastructure as it transitions to IPv6.
