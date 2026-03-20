# How to Set Up Squid Proxy on pfSense for IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: pfSense, Squid, Proxy, IPv4, Web Filtering, Content Caching

Description: Install and configure the Squid proxy package on pfSense for IPv4 web traffic caching and filtering, including transparent proxy mode, SSL inspection, and SquidGuard URL filtering.

## Introduction

Squid on pfSense provides HTTP/HTTPS caching and filtering for LAN clients. In transparent mode, clients need no manual proxy configuration - pfSense intercepts port 80/443 traffic automatically.

## Install Squid Package

Navigate to **System > Package Manager > Available Packages**:
- Install: `squid`
- Install: `squidGuard` (optional - for URL filtering)

## Basic Squid Configuration

Navigate to **Services > Squid Proxy Server > General**:

```text
Enable Squid Proxy:     checked
Proxy Interface(s):     LAN, OPT1 (interfaces to listen on)
Proxy Port:             3128
Allow Users on Interface: checked (allow LAN subnet)

Transparent HTTP Proxy: checked
Transparent Proxy Interface: LAN

Logging:
  Enable Access Logging: checked
  Log Store Directory:   /var/squid/logs
```

## Cache Configuration

Navigate to **Services > Squid Proxy Server > Cache Mgmt**:

```text
Hard Disk Cache Size:    2048 MB
Hard Disk Cache System:  ufs
Level 1 Subdirs:         16
Level 2 Subdirs:         256
Memory Cache Size:       256 MB
Maximum Object Size:     512 MB
```

## SSL/HTTPS Interception (SSL Bump)

```text
WARNING: SSL interception requires distributing pfSense CA cert to clients.

Navigate to: Services > Squid Proxy Server > General
  Enable SSL Interception:    checked
  SSL Interception Interface: LAN
  CA Certificate:             pfSense-CA (create in Cert Manager)
  SSL/TLS Method:             Modern
```

Deploy the CA cert to clients via GPO or MDM.

## Transparent Proxy Firewall Rules

Navigate to **Firewall > NAT > Port Forward > Add**:
```text
Interface:   LAN
Protocol:    TCP
Source:      LAN net
Dest port:   80
Redirect to: 127.0.0.1:3128
Description: Transparent proxy redirect

```

## SquidGuard URL Filtering

Navigate to **Services > SquidGuard Proxy Filter > General**:
- Enable: checked

Navigate to **Target categories**:
- Create category: `BLOCKED-SITES`
- Domain List: `facebook.com twitter.com tiktok.com`

Navigate to **Common ACL**:
- Target Rules: `BLOCKED-SITES → deny`
- Default access: `allow`

## Monitor Squid

Navigate to **Status > Squid Proxy Stats**:
- Cache hit rate, request counts, bandwidth saved

```bash
# pfSense CLI

squid -k check
tail -f /var/squid/logs/access.log
```

## Conclusion

Squid on pfSense enables transparent IPv4 web caching and filtering with minimal client configuration. Enable the transparent proxy, configure cache storage, optionally enable SSL interception for HTTPS filtering, and use SquidGuard for URL blocklists. The firewall NAT rule silently redirects port 80 traffic to Squid without requiring client proxy settings.
