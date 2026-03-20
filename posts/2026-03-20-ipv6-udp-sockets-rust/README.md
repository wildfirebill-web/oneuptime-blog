# How to Use IPv6 UDP Sockets in Rust

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rust, IPv6, UDP, Networking, Tokio, Multicast

Description: Use IPv6 UDP sockets in Rust for unicast messaging, multicast groups, and async UDP with Tokio.

## Basic IPv6 UDP Socket

```rust
use std::net::UdpSocket;

fn main() -> std::io::Result<()> {
    // Bind to IPv6 any address
    let socket = UdpSocket::bind("[::]:9000")?;
    println!("UDP server on {}", socket.local_addr()?);

    let mut buf = [0u8; 1500];
    loop {
        let (len, src) = socket.recv_from(&mut buf)?;
        let msg = std::str::from_utf8(&buf[..len]).unwrap_or("<binary>");
        println!("From {}: {}", src, msg);

        // Echo back
        socket.send_to(&buf[..len], src)?;
    }
}
```

## IPv6 UDP Client

```rust
use std::net::UdpSocket;

fn main() -> std::io::Result<()> {
    // Bind to ephemeral port on any IPv6 interface
    let socket = UdpSocket::bind("[::]:0")?;

    let server = "[2001:db8::1]:9000";
    let msg = b"Hello over IPv6 UDP";

    socket.send_to(msg, server)?;
    println!("Sent {} bytes to {}", msg.len(), server);

    let mut buf = [0u8; 1500];
    let (len, from) = socket.recv_from(&mut buf)?;
    println!("Response from {}: {}", from, std::str::from_utf8(&buf[..len]).unwrap());

    Ok(())
}
```

## IPv6 Multicast

IPv6 multicast uses `ff00::/8` addresses. Join a multicast group to receive packets sent to that group:

```rust
use std::net::{Ipv6Addr, UdpSocket};

fn main() -> std::io::Result<()> {
    let socket = UdpSocket::bind("[::]:5353")?;

    // Join ff02::fb (mDNS multicast) on interface index 0 (any)
    let multicast_addr: Ipv6Addr = "ff02::fb".parse().unwrap();
    socket.join_multicast_v6(&multicast_addr, 0)?;

    println!("Joined multicast group ff02::fb");

    let mut buf = [0u8; 1500];
    loop {
        let (len, src) = socket.recv_from(&mut buf)?;
        println!("Multicast from {}: {} bytes", src, len);
    }
}
```

To send multicast, set the outgoing interface:

```rust
use std::net::UdpSocket;

fn send_multicast(message: &[u8], iface_index: u32) -> std::io::Result<()> {
    let socket = UdpSocket::bind("[::]:0")?;

    // Set outgoing multicast interface
    socket.set_multicast_if_v6(iface_index)?;

    // Send to all-nodes on link scope
    socket.send_to(message, "[ff02::1]:5000")?;
    println!("Sent {} bytes to ff02::1", message.len());

    Ok(())
}

fn main() -> std::io::Result<()> {
    send_multicast(b"announcement", 1) // interface index 1 = typically eth0
}
```

## Async UDP with Tokio

```toml
# Cargo.toml

[dependencies]
tokio = { version = "1", features = ["full"] }
```

```rust
use tokio::net::UdpSocket;

#[tokio::main]
async fn main() -> tokio::io::Result<()> {
    let socket = UdpSocket::bind("[::]:9000").await?;
    println!("Async UDP server on {}", socket.local_addr()?);

    let mut buf = vec![0u8; 4096];
    loop {
        let (len, src) = socket.recv_from(&mut buf).await?;
        let msg = String::from_utf8_lossy(&buf[..len]);
        println!("From {}: {}", src, msg);
        socket.send_to(&buf[..len], src).await?;
    }
}
```

## Async UDP with Shared Socket (Arc)

When multiple tasks need to send from the same socket, wrap it in `Arc`:

```rust
use std::sync::Arc;
use tokio::net::UdpSocket;

#[tokio::main]
async fn main() -> tokio::io::Result<()> {
    let socket = Arc::new(UdpSocket::bind("[::]:9000").await?);

    // Receiver task
    let recv_sock = socket.clone();
    let send_sock = socket.clone();

    let recv_handle = tokio::spawn(async move {
        let mut buf = vec![0u8; 4096];
        loop {
            let (len, src) = recv_sock.recv_from(&mut buf).await.unwrap();
            println!("Received {} bytes from {}", len, src);
        }
    });

    // Sender task
    let send_handle = tokio::spawn(async move {
        let targets = ["[2001:db8::1]:9001", "[2001:db8::2]:9001"];
        for target in targets {
            send_sock.send_to(b"ping", target).await.unwrap();
        }
    });

    tokio::try_join!(recv_handle, send_handle).unwrap();
    Ok(())
}
```

## Setting Socket Options

```rust
use std::net::UdpSocket;
use std::time::Duration;

fn configured_udp_socket() -> std::io::Result<UdpSocket> {
    let socket = UdpSocket::bind("[::]:0")?;

    // Set read/write timeouts
    socket.set_read_timeout(Some(Duration::from_secs(5)))?;
    socket.set_write_timeout(Some(Duration::from_secs(5)))?;

    // Allow multiple sockets on the same port
    socket.set_reuse_address(true)?;

    // Set TTL / hop limit for outgoing packets
    socket.set_multicast_ttl_v6(16)?;  // max 16 hops for multicast

    Ok(socket)
}

fn main() -> std::io::Result<()> {
    let sock = configured_udp_socket()?;
    println!("Socket ready: {}", sock.local_addr()?);
    Ok(())
}
```

## Conclusion

Rust's `std::net::UdpSocket` handles IPv6 by binding to `[::]:port` or a specific IPv6 address. Multicast group membership uses `join_multicast_v6()` with an interface index. For async applications, Tokio's `UdpSocket` mirrors the API with `.await` semantics. Wrap shared send sockets in `Arc` to allow multiple tasks to send concurrently.
