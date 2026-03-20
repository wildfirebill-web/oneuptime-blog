# How to Work with IPv6 Link-Local Addresses in Go

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Go, Link-Local, Zone ID, Networking

Description: Handle IPv6 link-local addresses in Go including zone IDs, binding to specific interfaces, using link-local addresses for discovery, and connecting to link-local endpoints.

## Understanding Link-Local and Zone IDs

```go
package main

import (
    "fmt"
    "net"
    "net/netip"
)

func main() {
    // Link-local addresses: fe80::/10
    // REQUIRE a zone ID (interface name) to be usable

    // Parse link-local — valid without zone but not usable for sockets
    addr, _ := netip.ParseAddr("fe80::1")
    fmt.Println("Is link-local:", addr.IsLinkLocalUnicast())  // true

    // With zone ID — for socket operations
    addrWithZone, _ := netip.ParseAddrPort("[fe80::1%eth0]:8080")
    fmt.Println("Addr:", addrWithZone.Addr())   // fe80::1%eth0
    fmt.Println("Zone:", addrWithZone.Addr().Zone())  // eth0

    // net.IP approach
    ip := net.ParseIP("fe80::1")
    fmt.Println("net.IP link-local:", ip.IsLinkLocalUnicast())  // true
    // net.IP has no zone ID — must use net.IPAddr or net.UDPAddr

    // For binding, always specify interface
    ipAddr := &net.IPAddr{IP: ip, Zone: "eth0"}
    fmt.Println("With zone:", ipAddr)  // fe80::1%eth0
}
```

## List Link-Local Addresses on All Interfaces

```go
package main

import (
    "fmt"
    "net"
)

func getLinkLocalAddresses() []struct {
    Iface string
    Addr  net.IP
} {
    var results []struct {
        Iface string
        Addr  net.IP
    }

    ifaces, err := net.Interfaces()
    if err != nil {
        return results
    }

    for _, iface := range ifaces {
        if iface.Flags&net.FlagLoopback != 0 {
            continue  // skip loopback
        }

        addrs, _ := iface.Addrs()
        for _, addr := range addrs {
            var ip net.IP
            switch v := addr.(type) {
            case *net.IPNet:
                ip = v.IP
            }
            if ip != nil && ip.IsLinkLocalUnicast() {
                results = append(results, struct {
                    Iface string
                    Addr  net.IP
                }{iface.Name, ip})
            }
        }
    }
    return results
}

func main() {
    linkLocals := getLinkLocalAddresses()
    fmt.Printf("Found %d link-local addresses:\n", len(linkLocals))
    for _, ll := range linkLocals {
        fmt.Printf("  %s%%%-12s  (fe80:: range)\n", ll.Addr, ll.Iface)
    }
}
```

## Dial to Link-Local Address with Zone ID

```go
package main

import (
    "fmt"
    "net"
    "time"
)

func dialLinkLocal(ipv6LinkLocal, zone, port string) (net.Conn, error) {
    // Zone ID is required for link-local addresses
    // Combine as: "fe80::1%eth0"
    addr := fmt.Sprintf("[%s%%%s]:%s", ipv6LinkLocal, zone, port)
    fmt.Printf("Dialing: %s\n", addr)

    conn, err := net.DialTimeout("tcp6", addr, 5*time.Second)
    if err != nil {
        return nil, fmt.Errorf("dial failed: %w", err)
    }
    return conn, nil
}

func main() {
    // Connect to a router's link-local SSH or management port
    conn, err := dialLinkLocal("fe80::1", "eth0", "22")
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    defer conn.Close()

    local := conn.LocalAddr()
    remote := conn.RemoteAddr()
    fmt.Printf("Connected: %s → %s\n", local, remote)
}
```

## Listen on Link-Local Address

```go
package main

import (
    "fmt"
    "net"
)

func listenLinkLocal(iface string, port int) (net.Listener, error) {
    // Bind to link-local address on specific interface
    // Use "%iface" zone suffix
    bindAddr := fmt.Sprintf("[fe80::%%%s]:%d", iface, port)

    ln, err := net.Listen("tcp6", bindAddr)
    if err != nil {
        // Fallback: listen on all link-local addresses
        fmt.Printf("Specific bind failed (%v), trying all link-locals\n", err)
        ln, err = net.Listen("tcp6", fmt.Sprintf("[::%%%s]:%d", iface, port))
        if err != nil {
            return nil, err
        }
    }
    return ln, nil
}

func main() {
    ln, err := listenLinkLocal("eth0", 8080)
    if err != nil {
        fmt.Println("Listen error:", err)
        return
    }
    defer ln.Close()
    fmt.Printf("Listening on: %s\n", ln.Addr())

    for {
        conn, err := ln.Accept()
        if err != nil {
            break
        }
        fmt.Printf("  Connection from: %s\n", conn.RemoteAddr())
        conn.Close()
    }
}
```

## UDP with Link-Local Multicast

```go
package main

import (
    "fmt"
    "net"
)

func sendMulticast(iface, message string) error {
    // ff02::1 = all nodes on link
    // ff02::2 = all routers on link
    multicastAddr := &net.UDPAddr{
        IP:   net.ParseIP("ff02::1"),
        Port: 5353,
        Zone: iface,  // Zone ID for link-scoped multicast
    }

    conn, err := net.ListenUDP("udp6", &net.UDPAddr{
        IP:   net.IPv6zero,
        Zone: iface,
    })
    if err != nil {
        return err
    }
    defer conn.Close()

    _, err = conn.WriteToUDP([]byte(message), multicastAddr)
    return err
}

func receiveMulticast(iface string) error {
    // Join all-nodes multicast group on interface
    groupAddr := &net.UDPAddr{
        IP:   net.ParseIP("ff02::1"),
        Port: 5353,
        Zone: iface,
    }

    conn, err := net.ListenMulticastUDP("udp6", &net.Interface{Name: iface}, groupAddr)
    if err != nil {
        return err
    }
    defer conn.Close()

    buf := make([]byte, 1024)
    for {
        n, addr, err := conn.ReadFromUDP(buf)
        if err != nil {
            return err
        }
        fmt.Printf("Multicast from [%s%%%s]: %s\n", addr.IP, addr.Zone, buf[:n])
    }
}

func main() {
    // Example: send multicast discovery message
    if err := sendMulticast("eth0", "IPv6 discovery probe"); err != nil {
        fmt.Println("Send error:", err)
    }
}
```

## Conclusion

IPv6 link-local addresses in Go require a zone ID (interface name) for all socket operations — without it, the OS cannot determine which interface to use. Always specify zone IDs: in `net.IPAddr{Zone: "eth0"}`, in `net.UDPAddr{Zone: "eth0"}`, or by appending `%iface` to the address string in dial/listen calls (`"[fe80::1%eth0]:8080"`). Use `net/netip.Addr.Zone()` to extract the zone from a parsed address. For link-local multicast (service discovery protocols like mDNS), use `net.ListenMulticastUDP` with the interface's `Zone` field set. Always check `ip.IsLinkLocalUnicast()` before using a link-local address and ensure your application handles the zone ID requirement gracefully.
