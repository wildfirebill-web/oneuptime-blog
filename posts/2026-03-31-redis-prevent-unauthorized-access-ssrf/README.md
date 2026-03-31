# How to Prevent Redis Unauthorized Access and SSRF Attacks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Security, SSRF, Network, Authorization

Description: Learn how to prevent unauthorized Redis access and SSRF attacks by combining network binding, authentication, and firewall rules for a layered defense.

---

Redis has been compromised in many real-world attacks through Server-Side Request Forgery (SSRF) and misconfigured network exposure. An attacker who can reach your Redis port can read all data, write arbitrary keys, and in some cases achieve remote code execution.

## Why Redis Is Vulnerable to SSRF

If an application accepts a URL and fetches it server-side, an attacker can point it to `http://redis-host:6379/` and inject Redis commands via HTTP-like payloads. Since Redis uses a simple line-based protocol, HTTP request headers can accidentally execute commands.

## Layer 1: Bind to Private Interfaces Only

Never let Redis listen on `0.0.0.0` in production:

```text
# /etc/redis/redis.conf
bind 127.0.0.1 10.0.1.5
protected-mode yes
```

## Layer 2: Require Authentication

Require a strong password for all connections:

```bash
ACL SETUSER default on >strongrandompassword ~* &* allcommands
```

Or in `redis.conf`:

```text
requirepass strongrandompassword
```

## Layer 3: Firewall Rules

Block the Redis port at the OS level:

```bash
# Allow only app servers
sudo ufw allow from 10.0.1.0/24 to any port 6379
sudo ufw deny 6379
sudo ufw reload
```

## Layer 4: Disable Dangerous Commands

Remove commands that can be abused via SSRF:

```bash
ACL SETUSER default -CONFIG -DEBUG -SLAVEOF -REPLICAOF -MODULE
```

## Layer 5: Block Redis in Application Egress

If your application uses an HTTP client that could be redirected, block outbound connections to the Redis port from the app tier:

```bash
# Block app servers from reaching Redis directly (force use of proper clients)
iptables -A OUTPUT -d 10.0.1.5 -p tcp --dport 6379 -m owner ! --uid-owner redis-app -j DROP
```

## Detecting SSRF Attempts via Redis Logs

Enable verbose logging to spot unusual connection patterns:

```text
loglevel verbose
logfile /var/log/redis/redis-server.log
```

Then monitor for unexpected client IPs:

```bash
grep "Accepted" /var/log/redis/redis-server.log | \
  awk '{print $NF}' | sort | uniq -c | sort -rn
```

## Test Your Exposure

Check if Redis is accidentally public:

```bash
nmap -p 6379 your-public-ip
# Port should show as filtered, not open
```

## Summary

Prevent Redis unauthorized access and SSRF attacks with multiple layers: bind to private IPs, require strong authentication via ACL, apply OS firewall rules, and disable dangerous commands. Monitor connection logs for unexpected source IPs. No single measure is sufficient - all layers together create a secure configuration.
