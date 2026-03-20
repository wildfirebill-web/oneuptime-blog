# How to Configure DHCP Option 66 and Option 67 for TFTP Boot

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCP, TFTP, PXE Boot, Option 66, Option 67, sysadmin

Description: DHCP option 66 delivers the TFTP server hostname or IP and option 67 delivers the boot filename to network-booting clients, together enabling PXE and VoIP phone provisioning workflows.

## What Are Options 66 and 67?

| Option | RFC | Purpose | Data Type |
|--------|-----|---------|-----------|
| 66 | RFC 2132 | TFTP server name/IP | String |
| 67 | RFC 2132 | Bootfile name | String |

These options inform DHCP clients where to find their boot file and what file to load. Combined with `siaddr` (TFTP server IP in the DHCP header), they form the basis of PXE and phone provisioning.

## ISC dhcpd Configuration

```
# /etc/dhcp/dhcpd.conf

# For BIOS PXE boot
subnet 10.0.0.0 netmask 255.255.255.0 {
    range 10.0.0.100 10.0.0.200;
    option routers 10.0.0.1;

    # Option 66: TFTP server address
    option tftp-server-name "10.0.0.10";

    # Option 67: Boot filename
    filename "pxelinux.0";

    # Also set next-server (siaddr) for compatibility
    next-server 10.0.0.10;
}

# Differentiate UEFI vs BIOS using vendor class
class "UEFI-64" {
    match if option vendor-class-identifier = "PXEClient:Arch:00007";
    filename "shimx64.efi";                 # UEFI boot file
    option tftp-server-name "10.0.0.10";
}

class "BIOS" {
    match if option vendor-class-identifier = "PXEClient:Arch:00000";
    filename "pxelinux.0";                  # BIOS boot file
    option tftp-server-name "10.0.0.10";
}
```

## dnsmasq Configuration

```
# /etc/dnsmasq.conf

# Enable TFTP server built into dnsmasq
enable-tftp
tftp-root=/var/lib/tftpboot

# Option 66 and 67 via dhcp-boot
# dhcp-boot=[filename],[tftp-hostname],[tftp-ip]
dhcp-boot=pxelinux.0,tftpserver,10.0.0.10

# UEFI support with multiple tags
dhcp-match=set:efi-x86_64,option:client-arch,7
dhcp-boot=tag:efi-x86_64,shimx64.efi,tftpserver,10.0.0.10
dhcp-boot=tag:!efi-x86_64,pxelinux.0,tftpserver,10.0.0.10
```

## For VoIP Phones (Generic)

```
# Phones needing firmware from TFTP
subnet 10.0.30.0 netmask 255.255.255.0 {
    range 10.0.30.10 10.0.30.250;
    option routers 10.0.30.1;
    next-server 10.0.0.100;                 # TFTP/provisioning server
    option tftp-server-name "10.0.0.100";  # Option 66
    filename "phone_config.xml";            # Option 67 (phone config)
    default-lease-time 3600;
}
```

## Testing Option 66 and 67 Delivery

```bash
# Check which options a client received
# Method 1: verbose dhclient
sudo dhclient -v eth0 2>&1 | grep -E "option|tftp|boot"

# Method 2: tshark
tshark -i eth0 -Y "bootp.option.type == 66 || bootp.option.type == 67" \
    -T fields -e bootp.option.tftp_server_name \
               -e bootp.option.bootfile_name
```

## Key Takeaways

- Option 66 (string) and `siaddr` (binary IP in header) are redundant; use both for compatibility.
- Option 67 defines the boot filename; combine with option 66 for full PXE provisioning.
- dnsmasq has a built-in TFTP server (`enable-tftp`) that simplifies PXE setups.
- For UEFI boot, use vendor class matching to serve `shimx64.efi`; for BIOS, serve `pxelinux.0`.
