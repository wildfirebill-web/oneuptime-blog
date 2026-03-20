# How to Set Up OpenVPN with LDAP Authentication for IPv4 Access

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenVPN, LDAP, Authentication, IPv4, VPN, Active Directory

Description: Integrate OpenVPN with LDAP or Active Directory for username/password authentication, enabling centralized user management for IPv4 VPN access.

## Introduction

OpenVPN supports plugin-based authentication, allowing you to validate VPN credentials against an LDAP directory (Active Directory, OpenLDAP). This means you can use existing corporate credentials for VPN access and revoke access by disabling the user's directory account.

## Installing the openvpn-auth-ldap Plugin

```bash
# Ubuntu/Debian
sudo apt-get install openvpn-auth-ldap

# The plugin library will be installed to:
# /usr/lib/openvpn/openvpn-auth-ldap.so
```

## Creating the LDAP Configuration

Create `/etc/openvpn/auth/ldap.conf`:

```xml
<!-- /etc/openvpn/auth/ldap.conf -->
<LDAP>
    <!-- LDAP server URL (use ldaps:// for encrypted) -->
    URL             ldaps://ldap.example.com:636
    
    <!-- Bind credentials for reading the directory -->
    BindDN          cn=vpn-service,ou=Service Accounts,dc=example,dc=com
    Password        your-service-account-password
    
    <!-- Connection timeout -->
    Timeout         15
    
    <!-- LDAP TLS settings -->
    TLSEnable       yes
    TLSCACertFile   /etc/ssl/certs/ca-certificates.crt
</LDAP>

<Authorization>
    <!-- Base DN to search for users -->
    BaseDN          ou=Users,dc=example,dc=com
    
    <!-- Search filter: %u is replaced with the username -->
    SearchFilter    (&(sAMAccountName=%u)(objectClass=user))
    
    <!-- Require users to be in this group to connect -->
    RequireGroup    yes
    
    <Group>
        BaseDN      ou=Groups,dc=example,dc=com
        SearchFilter (&(cn=VPN-Users)(objectClass=group))
        MemberAttribute member
    </Group>
</Authorization>
```

## Configuring the OpenVPN Server

Edit `/etc/openvpn/server.conf` to enable the LDAP plugin:

```bash
# /etc/openvpn/server.conf

proto udp
dev tun
port 1194

ca   /etc/openvpn/ca.crt
cert /etc/openvpn/server.crt
key  /etc/openvpn/server.key
dh   /etc/openvpn/dh.pem
tls-auth /etc/openvpn/ta.key 0

# IPv4 VPN subnet
server 10.8.0.0 255.255.255.0
push "redirect-gateway def1"
push "dhcp-option DNS 8.8.8.8"

# Load the LDAP authentication plugin
plugin /usr/lib/openvpn/openvpn-auth-ldap.so /etc/openvpn/auth/ldap.conf

# Require both certificate AND LDAP username/password (two-factor)
# If you want password-only (no cert), use: verify-client-cert none
# verify-client-cert none

keepalive 10 120
cipher AES-256-CBC
```

## Client Configuration

The client `.ovpn` file needs `auth-user-pass` to prompt for credentials:

```ovpn
# client.ovpn
client
dev tun
proto udp
remote vpn.example.com 1194

ca ca.crt
cert client.crt
key client.key
tls-auth ta.key 1

# Prompt for username and password on connection
auth-user-pass

cipher AES-256-CBC
verb 3
```

## Using a Credentials File (for Automation)

```bash
# Store credentials in a file
cat > /etc/openvpn/credentials.txt << 'EOF'
vpnuser
StrongPassword123
EOF
chmod 600 /etc/openvpn/credentials.txt

# Reference in client config
# auth-user-pass /etc/openvpn/credentials.txt
```

## Testing the LDAP Connection

```bash
# Test LDAP bind manually before starting OpenVPN
ldapsearch -x \
  -H ldaps://ldap.example.com:636 \
  -D "cn=vpn-service,ou=Service Accounts,dc=example,dc=com" \
  -w "your-service-account-password" \
  -b "ou=Users,dc=example,dc=com" \
  "(sAMAccountName=testuser)"
```

## Troubleshooting

```bash
# Check OpenVPN logs for LDAP auth failures
sudo journalctl -u openvpn@server -f

# Common errors:
# "LDAP bind failed" — wrong BindDN or password
# "User not found" — wrong BaseDN or SearchFilter
# "Group check failed" — user not in required group
```

## Conclusion

LDAP authentication for OpenVPN centralizes access control in your existing directory service. Users connect with corporate credentials, and access is revoked automatically when their account is disabled — eliminating the need to manually revoke VPN certificates when employees leave.
