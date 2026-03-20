# How to Create a UDP Server and Client in Go Using IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Go, UDP, Sockets, IPv4, Networking, Datagram

Description: Learn how to create a UDP server and client in Go using IPv4 with the net package, including reading sender addresses and sending replies.

## UDP Server in Go

```go
package main

import (
    "fmt"
    "log"
    "net"
)

func main() {
    // Resolve the UDP address to listen on
    addr, err := net.ResolveUDPAddr("udp4", "0.0.0.0:9001")
    if err != nil {
        log.Fatalf("ResolveUDPAddr error: %v", err)
    }

    // Create the UDP socket and bind to the address
    conn, err := net.ListenUDP("udp4", addr)
    if err != nil {
        log.Fatalf("ListenUDP error: %v", err)
    }
    defer conn.Close()

    log.Printf("UDP server listening on %s", conn.LocalAddr())

    buf := make([]byte, 65535)

    for {
        // ReadFromUDP returns the number of bytes, sender address, and error
        n, clientAddr, err := conn.ReadFromUDP(buf)
        if err != nil {
            log.Printf("ReadFromUDP error: %v", err)
            continue
        }

        message := string(buf[:n])
        log.Printf("Received %d bytes from %s: %s", n, clientAddr, message)

        // Send a reply back to the sender
        reply := fmt.Sprintf("Echo: %s", message)
        _, err = conn.WriteToUDP([]byte(reply), clientAddr)
        if err != nil {
            log.Printf("WriteToUDP error: %v", err)
        }
    }
}
```

## UDP Client in Go

```go
package main

import (
    "fmt"
    "log"
    "net"
    "time"
)

func main() {
    // Resolve server address
    serverAddr, err := net.ResolveUDPAddr("udp4", "127.0.0.1:9001")
    if err != nil {
        log.Fatal(err)
    }

    // Dial UDP (this sets the default remote address for Send/Recv)
    conn, err := net.DialUDP("udp4", nil, serverAddr)
    if err != nil {
        log.Fatalf("DialUDP error: %v", err)
    }
    defer conn.Close()

    // Set a read deadline to avoid blocking forever
    conn.SetReadDeadline(time.Now().Add(3 * time.Second))

    // Send a datagram
    message := []byte("Hello, UDP server!")
    _, err = conn.Write(message)
    if err != nil {
        log.Fatalf("Write error: %v", err)
    }

    // Receive reply
    buf := make([]byte, 4096)
    n, err := conn.Read(buf)
    if err != nil {
        if netErr, ok := err.(net.Error); ok && netErr.Timeout() {
            log.Println("Read timeout: no response received")
        } else {
            log.Fatalf("Read error: %v", err)
        }
        return
    }

    fmt.Printf("Server reply: %s\n", buf[:n])
}
```

## Using WriteTo/ReadFrom (Without Dial)

For a client that doesn't use `DialUDP`:

```go
package main

import (
    "fmt"
    "log"
    "net"
    "time"
)

func main() {
    // Create a UDP socket without connecting to a specific server
    conn, err := net.ListenUDP("udp4", &net.UDPAddr{IP: net.IPv4zero, Port: 0})
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()

    serverAddr := &net.UDPAddr{IP: net.ParseIP("127.0.0.1"), Port: 9001}

    // Send using WriteToUDP
    conn.WriteToUDP([]byte("Hello!"), serverAddr)

    // Receive reply with timeout
    conn.SetReadDeadline(time.Now().Add(3 * time.Second))
    buf := make([]byte, 4096)
    n, from, err := conn.ReadFromUDP(buf)
    if err != nil {
        log.Printf("No reply: %v", err)
        return
    }

    fmt.Printf("Reply from %s: %s\n", from, buf[:n])
}
```

## Concurrent UDP Server

```go
package main

import (
    "log"
    "net"
    "sync"
)

func main() {
    addr, _ := net.ResolveUDPAddr("udp4", ":9001")
    conn, _ := net.ListenUDP("udp4", addr)
    defer conn.Close()

    var wg sync.WaitGroup
    buf := make([]byte, 65535)

    for {
        n, client, err := conn.ReadFromUDP(buf)
        if err != nil {
            break
        }

        data := make([]byte, n)
        copy(data, buf[:n])

        wg.Add(1)
        go func(payload []byte, from *net.UDPAddr) {
            defer wg.Done()
            log.Printf("Processing %d bytes from %s", len(payload), from)
            conn.WriteToUDP(payload, from)
        }(data, client)
    }

    wg.Wait()
}
```

## Conclusion

UDP in Go uses `net.ListenUDP` for servers and `net.DialUDP` (or `net.ListenUDP` with `WriteToUDP`) for clients. Always set a read deadline on clients to prevent indefinite blocking. For concurrent handling, copy the datagram buffer before spawning a goroutine since the buffer is reused.
