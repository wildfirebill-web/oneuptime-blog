# How to Add a Persistent Static Route on Windows

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Windows, Static Routes, route add, PowerShell, Networking, Routing

Description: Add persistent static routes on Windows using the route add command with the -p flag or New-NetRoute PowerShell cmdlet to survive reboots.

## Introduction

Windows supports static routes through the `route` command and PowerShell's `New-NetRoute` cmdlet. Routes added without the `-p` flag are temporary and lost on reboot. The `-p` flag makes routes persistent, storing them in the Windows registry.

## Add a Temporary Static Route

```cmd
REM Syntax: route add <destination> mask <subnet> <gateway>
route add 192.168.2.0 mask 255.255.255.0 10.0.0.1
```

## Add a Persistent Static Route

```cmd
REM Add -p flag for persistent route (survives reboot)
route add 192.168.2.0 mask 255.255.255.0 10.0.0.1 -p
```

## Add a Route with a Specific Metric

```cmd
REM Lower metric = preferred route
route add 192.168.2.0 mask 255.255.255.0 10.0.0.1 metric 10 -p
```

## Add a Route on a Specific Interface

```cmd
REM Get interface index first
route print

REM Add route on specific interface (IF = interface index)
route add 192.168.2.0 mask 255.255.255.0 10.0.0.1 IF 5 -p
```

## View the Routing Table

```cmd
REM Show all routes
route print

REM Show only IPv4 routes
route print -4
```

## Delete a Static Route

```cmd
REM Delete a specific route
route delete 192.168.2.0 mask 255.255.255.0 10.0.0.1
```

## Using PowerShell New-NetRoute

```powershell
# Get the interface index
Get-NetAdapter | Select-Object Name, ifIndex

# Add a persistent static route
New-NetRoute -DestinationPrefix "192.168.2.0/24" `
             -NextHop "10.0.0.1" `
             -InterfaceIndex 5 `
             -RouteMetric 10

# View routes
Get-NetRoute | Where-Object { $_.DestinationPrefix -ne "0.0.0.0/0" }

# Remove a route
Remove-NetRoute -DestinationPrefix "192.168.2.0/24" -NextHop "10.0.0.1"
```

## Add Default Route

```cmd
REM Set a default gateway
route add 0.0.0.0 mask 0.0.0.0 10.0.0.1 -p
```

## Verify the Route Works

```cmd
REM Trace the route to the destination
tracert 192.168.2.100

REM Ping using the static route
ping 192.168.2.100
```

## Conclusion

Windows persistent static routes require the `-p` flag with `route add` or use PowerShell's `New-NetRoute`. Without `-p`, routes are lost on reboot. Use `route print` to verify and `route delete` to remove routes. PowerShell's `New-NetRoute` provides more modern management with tab completion and better integration with Windows networking stack.
