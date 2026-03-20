# How to Add a Static Route on Windows

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, Routing, Windows, IPv4

Description: Learn how to add static IPv4 routes on Windows using the route add command and PowerShell for both temporary and persistent routing.

## Adding a Temporary Route with `route add`

```cmd
route add DESTINATION MASK GATEWAY [METRIC n] [IF interface_index]
```

### Examples

```cmd
# Route to 10.0.0.0/24 via 192.168.1.254

route add 10.0.0.0 mask 255.255.255.0 192.168.1.254

# Route with specific metric
route add 10.0.0.0 mask 255.255.255.0 192.168.1.254 metric 5

# Route via specific interface (use route print to find interface index)
route add 10.0.0.0 mask 255.255.255.0 192.168.1.254 if 4

# Add default gateway
route add 0.0.0.0 mask 0.0.0.0 192.168.1.1
```

## Adding a Persistent Route

Without the `-p` flag, routes are removed at reboot. Use `-p` to make permanent:

```cmd
# Persistent route (survives reboot)
route add -p 10.0.0.0 mask 255.255.255.0 192.168.1.254

# Persistent with metric
route add -p 172.16.0.0 mask 255.240.0.0 192.168.1.254 metric 10
```

Persistent routes are stored in the registry:
`HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\PersistentRoutes`

## Adding Routes with PowerShell

```powershell
# Add temporary route
New-NetRoute -DestinationPrefix "10.0.0.0/24" -NextHop "192.168.1.254" -RouteMetric 100

# Specify interface
New-NetRoute -DestinationPrefix "10.0.0.0/24" -NextHop "192.168.1.254" `
             -InterfaceAlias "Ethernet" -RouteMetric 100

# Persistent is the default in PowerShell (unlike route add)
# To remove when done: Remove-NetRoute
```

## Verifying the Route

```cmd
# Show full routing table
route print

# Show only the new route
route print | findstr "10.0.0"

# Test which route is used
# (no equivalent of 'ip route get' on Windows, but tracert works)
tracert -d 10.0.0.1
```

## Modifying an Existing Route

```cmd
# The route change command modifies existing routes
route change 10.0.0.0 mask 255.255.255.0 192.168.1.1

# PowerShell: modify metric
Set-NetRoute -DestinationPrefix "10.0.0.0/24" -RouteMetric 200
```

## Script: Add Multiple Static Routes

```cmd
@echo off
REM Add corporate network routes
set GATEWAY=192.168.1.254

route add -p 10.10.0.0 mask 255.255.255.0 %GATEWAY%
route add -p 10.20.0.0 mask 255.255.255.0 %GATEWAY%
route add -p 172.16.5.0 mask 255.255.255.0 %GATEWAY%
echo Routes added successfully
```

PowerShell version:

```powershell
$gateway = "192.168.1.254"
$routes = @("10.10.0.0/24", "10.20.0.0/24", "172.16.5.0/24")

foreach ($route in $routes) {
    New-NetRoute -DestinationPrefix $route -NextHop $gateway -RouteMetric 100
    Write-Host "Added route: $route via $gateway"
}
```

## Key Takeaways

- `route add DEST mask MASK GATEWAY` adds a temporary route on Windows.
- Use `-p` flag with `route add` to make the route persistent across reboots.
- PowerShell's `New-NetRoute` adds persistent routes by default.
- Use `route print` to verify routes; `route change` to modify existing ones.

**Related Reading:**

- [How to View the Routing Table on Windows](https://oneuptime.com/blog/post/2026-03-20-view-routing-table-windows/view)
- [How to Add a Static Route on Linux](https://oneuptime.com/blog/post/2026-03-20-add-static-route-linux/view)
- [How to Configure a Default Gateway on Linux](https://oneuptime.com/blog/post/2026-03-20-configure-default-gateway-linux/view)
