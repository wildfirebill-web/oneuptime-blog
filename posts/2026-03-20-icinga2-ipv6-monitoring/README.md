# How to Configure Icinga2 for IPv6 Monitoring

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Icinga2, IPv6, Monitoring, Networks, ICMP, Service Checks

Description: A guide to configuring Icinga2 to monitor IPv6 hosts and services, including host object definitions, check commands, and apply rules.

Icinga2 provides full IPv6 support through its plugin check commands and host/service configuration DSL. IPv6 addresses can be used directly as `address6` attributes on host objects, enabling precise address family selection for checks.

## Step 1: Enable Icinga2 to Listen on IPv6

```bash
# Icinga2 listens on all interfaces by default

# Verify in /etc/icinga2/features-enabled/api.conf
cat /etc/icinga2/features-enabled/api.conf
```

```text
# /etc/icinga2/features-enabled/api.conf
object ApiListener "api" {
  # Listen on all interfaces (IPv4 and IPv6)
  bind_host = "::"
  bind_port = 5665
}
```

## Step 2: Define IPv6 Hosts

```icinga2
# /etc/icinga2/conf.d/ipv6-hosts.conf - Host objects with IPv6 addresses

object Host "web-01" {
  display_name = "Web Server 01"
  # IPv4 address (primary)
  address  = "10.0.1.10"
  # IPv6 address (for IPv6 checks)
  address6 = "2001:db8::10"

  vars.os = "Linux"
  vars.http_uri = "/"
  vars.check_ipv6 = true

  check_command = "hostalive"  # Default check uses address (IPv4)
}

object Host "ipv6-only-host" {
  display_name = "IPv6-Only Server"
  # Only IPv6 address defined
  address6 = "2001:db8::20"

  # Use hostalive6 to check via IPv6
  check_command = "hostalive6"
}
```

## Step 3: Configure IPv6 Check Commands

```icinga2
# /etc/icinga2/conf.d/commands-ipv6.conf - IPv6-specific check commands

# ICMP ping over IPv6
object CheckCommand "hostalive6" {
  import "plugin-check-command"
  command = [ PluginDir + "/check_ping", "-6",
              "-H", "$address6$",
              "-w", "3000.0,80%",
              "-c", "5000.0,100%",
              "-p", "1" ]
  vars.address6 = "$host.address6$"
}

# HTTP check over IPv6
object CheckCommand "http-ipv6" {
  import "plugin-check-command"
  command = [ PluginDir + "/check_http",
              "-H", "$http_vhost$",
              "-I", "$address6$",
              "-u", "$http_uri$" ]
  vars.http_vhost = "$host.name$"
  vars.http_uri = "$host.vars.http_uri$"
  vars.address6 = "$host.address6$"
}
```

## Step 4: Apply Service Rules for IPv6 Hosts

```icinga2
# /etc/icinga2/conf.d/ipv6-services.conf - Apply rules for IPv6 monitoring

# Apply HTTP check to hosts with IPv6 address and http_uri defined
apply Service "HTTP IPv6" {
  import "generic-service"
  check_command = "http-ipv6"
  assign where host.address6 && host.vars.http_uri
}

# Apply ICMP IPv6 check to all hosts with an IPv6 address
apply Service "ICMP IPv6 Ping" {
  import "generic-service"
  check_command = "hostalive6"
  check_interval = 1m
  assign where host.address6
}

# Apply SSH check to Linux IPv6 hosts
apply Service "SSH IPv6" {
  import "generic-service"
  check_command = "ssh"
  vars.ssh_address = host.address6
  assign where host.address6 && host.vars.os == "Linux"
}
```

## Step 5: Dual-Stack Monitoring

Monitor both IPv4 and IPv6 independently to detect address-family-specific issues:

```icinga2
# Monitor HTTP over both IPv4 and IPv6
apply Service "HTTP IPv4" {
  import "generic-service"
  check_command = "http"
  assign where host.address && host.vars.http_uri
}

apply Service "HTTP IPv6" {
  import "generic-service"
  check_command = "http-ipv6"
  assign where host.address6 && host.vars.http_uri
}
```

## Step 6: Validate and Reload

```bash
# Check configuration syntax
sudo icinga2 daemon -C

# Reload Icinga2
sudo systemctl reload icinga2

# Verify IPv6 hosts appear in Icinga2 Web
curl -u root:icinga "https://localhost/icingaweb2/monitoring/list/hosts?host=*ipv6*"
```

## Step 7: Add Notification for IPv6 Failures

```icinga2
apply Notification "ipv6-alert" to Host {
  import "mail-host-notification"
  user_groups = ["admins"]
  assign where host.address6
}
```

Icinga2's `address6` host attribute and `apply` rules make it straightforward to build comprehensive dual-stack monitoring that tracks IPv4 and IPv6 independently, surfacing address-family-specific failures clearly.
