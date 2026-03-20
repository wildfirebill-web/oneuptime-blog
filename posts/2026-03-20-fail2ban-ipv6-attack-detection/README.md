# How to Configure Fail2Ban for IPv6 Attack Detection

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Fail2Ban, Security, Brute Force, Linux

Description: Configure Fail2Ban to detect and block brute force attacks from IPv6 addresses by using ip6tables actions, writing IPv6-compatible filters, and tuning ban policies for IPv6 subnets.

## Introduction

Fail2Ban monitors log files for failed authentication attempts and blocks source IPs using firewall rules. IPv6 support requires ip6tables actions (or nftables) since the default `iptables` action only handles IPv4. This guide covers enabling IPv6 support, configuring jails for IPv6, and writing IPv6-aware filters.

## Step 1: Check Fail2Ban IPv6 Readiness

```bash
# Check current Fail2Ban version (>= 0.10 for IPv6 support)
fail2ban-client version

# Check available actions
ls /etc/fail2ban/action.d/ | grep -E "ip6tables|nftables"

# Verify ip6tables is available
which ip6tables && ip6tables -L -n | head -5
```

## Step 2: Configure ip6tables Action

```ini
# /etc/fail2ban/action.d/ip6tables-multiport.local
# Override default to add IPv6 support

[Init]
# ip6tables action uses ip6tables command
```

The built-in `ip6tables-multiport` action in Fail2Ban 0.10+ handles IPv6. Verify:

```bash
# Check the action handles IPv6
cat /etc/fail2ban/action.d/ip6tables-multiport.conf | head -20
```

## Step 3: Configure Jails for IPv6

```ini
# /etc/fail2ban/jail.local

[DEFAULT]
# Use bantime, findtime, maxretry defaults
bantime  = 3600
findtime = 600
maxretry = 5

# Enable both IPv4 and IPv6 ban actions
banaction = iptables-multiport
banaction_allports = iptables-allports

# For IPv6 support, use dual-stack actions
# Fail2Ban >= 0.10 auto-detects IP version

[sshd]
enabled  = true
port     = ssh
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 3
# Ban for 24 hours after 3 failed attempts
bantime  = 86400

[nginx-http-auth]
enabled  = true
port     = http,https
filter   = nginx-http-auth
logpath  = /var/log/nginx/error.log
maxretry = 5

[nginx-botsearch]
enabled  = true
port     = http,https
filter   = nginx-botsearch
logpath  = /var/log/nginx/access.log
maxretry = 2
```

## Step 4: Write IPv6-Compatible Filters

Fail2Ban filters use Python regex patterns. IPv6 addresses contain colons, so patterns need updating:

```ini
# /etc/fail2ban/filter.d/sshd-ipv6.conf
# Extended sshd filter that explicitly handles IPv6

[Definition]
# Matches both IPv4 and IPv6 failed SSH attempts
_daemon = sshd

# IPv4: standard dotted-decimal
# IPv6: hex groups separated by colons
failregex = ^%(__prefix_line)s(?:error: PAM: )?[aA]uthentication (?:failure|error|failed) for .* from <HOST>\s*$
            ^%(__prefix_line)sFailed \S+ for (?:invalid user )?(?P<user>.+?) from <HOST>(?: port \d+)?(?: ssh\d*)?\s*$
            ^%(__prefix_line)sUser .+ from <HOST> not allowed because not listed in AllowUsers\s*$

# <HOST> in Fail2Ban matches both IPv4 and IPv6 addresses

ignoreregex =
```

```ini
# /etc/fail2ban/filter.d/nginx-ipv6.conf
# Nginx 4xx error filter for IPv6

[Definition]
failregex = ^<HOST> .+"(GET|POST|HEAD|PUT|DELETE) .+\s+(400|401|403|404|429)\s

ignoreregex =
```

## Step 5: Whitelist IPv6 Prefixes

```ini
# /etc/fail2ban/jail.local — add IPv6 whitelist

[DEFAULT]
ignoreip = 127.0.0.1/8 ::1
           10.0.0.0/8 172.16.0.0/12 192.168.0.0/16
           # ULA (internal)
           fc00::/7
           # Your management subnet
           2001:db8:admin::/48
```

## Step 6: Test and Monitor IPv6 Bans

```bash
# Test a filter against a log file
fail2ban-regex /var/log/auth.log /etc/fail2ban/filter.d/sshd.conf

# Manually test an IPv6 log line
fail2ban-regex \
    "Mar 20 10:00:00 host sshd[1234]: Failed password for root from 2001:db8::1 port 54321 ssh2" \
    /etc/fail2ban/filter.d/sshd.conf

# Check current bans (includes IPv6)
fail2ban-client status sshd

# Manually ban an IPv6 address
fail2ban-client set sshd banip 2001:db8::1

# Unban an IPv6 address
fail2ban-client set sshd unbanip 2001:db8::1

# Check ip6tables ban rules
ip6tables -L f2b-sshd -n -v
```

## Step 7: nftables Action (Modern Alternative)

```ini
# Use nftables instead of ip6tables for unified IPv4/IPv6 banning
# /etc/fail2ban/jail.local

[DEFAULT]
banaction = nftables-multiport
banaction_allports = nftables-allports
```

nftables handles IPv4 and IPv6 in the same ruleset, simplifying dual-stack ban management.

## Conclusion

Fail2Ban supports IPv6 brute force detection through its `<HOST>` placeholder in filter regexes, which matches both IPv4 and IPv6 addresses. Use `ip6tables-multiport` or `nftables` ban actions for IPv6 blocking. Whitelist internal IPv6 prefixes (ULA `fc00::/7`, loopback `::1`) in `ignoreip` to prevent accidental self-blocking. Test filters with `fail2ban-regex` against real log lines containing IPv6 addresses to verify detection before enabling in production.
