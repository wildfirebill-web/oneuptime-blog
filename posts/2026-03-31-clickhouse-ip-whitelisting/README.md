# How to Set Up IP Whitelisting in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Security, IP Whitelisting, Access Control, Network Security

Description: Learn how to restrict ClickHouse access to trusted IP addresses using host-based restrictions and network-level controls.

---

ClickHouse does not have a built-in firewall, but you can enforce IP whitelisting at the user level and via network configuration. This prevents unauthorized hosts from connecting even with valid credentials.

## Configure User-Level Host Restrictions

In `users.xml`, use the `<networks>` section to restrict which IPs can authenticate as a given user:

```xml
<users>
  <analytics_user>
    <password>secret</password>
    <networks>
      <ip>10.0.1.0/24</ip>
      <ip>192.168.5.10</ip>
    </networks>
    <profile>default</profile>
    <quota>default</quota>
  </analytics_user>
</users>
```

Reload configuration with:

```bash
sudo systemctl reload clickhouse-server
```

## Use SQL-Based User Creation

For ClickHouse 20.4+, you can define host restrictions via SQL:

```sql
CREATE USER analytics_user
  IDENTIFIED WITH plaintext_password BY 'secret'
  HOST IP '10.0.1.0/24', IP '192.168.5.10';
```

To update an existing user:

```sql
ALTER USER analytics_user HOST IP '10.0.2.0/24';
```

## Verify Restrictions

Check current user host settings in the system tables:

```sql
SELECT name, host_ip, host_names
FROM system.users
WHERE name = 'analytics_user';
```

## Combine with OS-Level Firewall

For defense in depth, also restrict the ClickHouse ports (9000/TCP for native, 8123/TCP for HTTP) at the OS level:

```bash
sudo ufw allow from 10.0.1.0/24 to any port 9000
sudo ufw allow from 10.0.1.0/24 to any port 8123
sudo ufw deny 9000
sudo ufw deny 8123
sudo ufw enable
```

## Use a Reverse Proxy for HTTP Connections

If clients connect over HTTP, place nginx or HAProxy in front of ClickHouse and enforce IP restrictions there:

```text
upstream clickhouse {
    server 127.0.0.1:8123;
}

server {
    listen 8443 ssl;
    allow 10.0.1.0/24;
    deny all;
    location / {
        proxy_pass http://clickhouse;
    }
}
```

## Testing Access

Test that an unauthorized IP is blocked by using `clickhouse-client` from a different host:

```bash
clickhouse-client --host clickhouse.example.com --user analytics_user --password secret
# Expected: Code: 516. DB::Exception: analytics_user: Authentication failed
```

## Summary

IP whitelisting in ClickHouse is achieved by combining user-level host restrictions in `users.xml` or via SQL `CREATE USER ... HOST IP` clauses, OS-level firewall rules, and optionally a reverse proxy for HTTP clients. This layered approach ensures only trusted network ranges can reach your ClickHouse cluster.
