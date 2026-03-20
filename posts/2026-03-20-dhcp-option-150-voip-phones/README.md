# How to Understand DHCP Option 150 for VoIP Phones

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCP, VoIP, Option 150, TFTP, Cisco, Sysadmin

Description: DHCP option 150 is a Cisco-proprietary option that delivers the TFTP server address to Cisco IP phones, enabling them to download their firmware and configuration automatically during boot.

## What Is DHCP Option 150?

Option 150 is a Cisco proprietary extension to DHCP that provides the IP address of a TFTP server to Cisco IP phones (and other Cisco VoIP equipment). Unlike standard option 66 (which carries a server name as a string), option 150 carries an IP address directly, which Cisco phones prefer.

## ISC dhcpd Configuration

```text
# /etc/dhcp/dhcpd.conf

# Define option 150 as a TFTP server IP address

option tftp-server-address code 150 = ip-address;

subnet 10.0.30.0 netmask 255.255.255.0 {
    range 10.0.30.10 10.0.30.250;
    option routers 10.0.30.1;
    option domain-name-servers 10.0.0.53;

    # Cisco CUCM/TFTP server IP
    option tftp-server-address 10.0.0.100;

    # Also set standard option 66 for non-Cisco phones
    option tftp-server-name "10.0.0.100";

    default-lease-time 3600;    # 1-hour leases for phones
}
```

## Serving Option 150 to Specific Devices Only

Use vendor class matching to send option 150 only to Cisco phones:

```text
# /etc/dhcp/dhcpd.conf
class "cisco-phone" {
    match if substring(option vendor-class-identifier, 0, 5) = "Cisco";
    option tftp-server-address 10.0.0.100;
    option tftp-server-name "10.0.0.100";
}

subnet 10.0.30.0 netmask 255.255.255.0 {
    range 10.0.30.10 10.0.30.250;
    option routers 10.0.30.1;
    allow members of "cisco-phone";
}
```

## dnsmasq Configuration

```text
# /etc/dnsmasq.conf

# Option 150 as hex (0x96 = 150 in decimal)
# Format: dhcp-option=<interface>,<option-number>,<value>
dhcp-option=eth0.30,150,10.0.0.100

# Also set standard option 66 (TFTP server name)
dhcp-option=eth0.30,66,10.0.0.100

# Option 67: boot filename (optional, for PXE-style booting)
# dhcp-option=eth0.30,67,SEPdefault.cnf
```

## Option 150 vs Option 66

| Option | Number | Format | Standard |
|--------|--------|--------|----------|
| TFTP Server Name | 66 | String (hostname/IP) | RFC 2132 |
| Cisco TFTP Server | 150 | IP address | Cisco proprietary |
| Next Server | `siaddr` in header | IP address | Part of DHCP header |

Cisco phones check option 150 first, then option 66, then the `siaddr` field.

## Verifying Option 150 Delivery

```bash
# Check what DHCP options a client receives
sudo tcpdump -i eth0 'port 67 or port 68' -v -n | grep -A5 "DHCPACK"

# tshark: show option 150
tshark -i eth0 -Y "bootp.option.type == 150" \
    -T fields -e bootp.option.tftp_server_address
```

## Key Takeaways

- Option 150 carries the TFTP server IP address as binary data (not a string like option 66).
- Cisco IP phones prioritize option 150 over option 66 for TFTP server discovery.
- Define option 150 in ISC dhcpd as `option tftp-server-address code 150 = ip-address;`.
- Deploy both option 150 and option 66 for maximum compatibility across phone vendors.
