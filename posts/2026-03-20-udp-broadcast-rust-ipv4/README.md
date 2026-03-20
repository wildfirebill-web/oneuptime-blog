# How to Build a UDP Broadcast Application in Rust for IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rust, UDP, Broadcast, IPv4, UdpSocket, Networking

Description: Learn how to send and receive UDP broadcast messages over IPv4 in Rust using UdpSocket with the set_broadcast option.

## UDP Broadcast Sender

```rust
use std::net::UdpSocket;
use std::time::Duration;
use std::thread;

fn main() -> std::io::Result<()> {
    let socket = UdpSocket::bind("0.0.0.0:0")?;

    // set_broadcast(true) is required before sending to broadcast addresses
    socket.set_broadcast(true)?;

    let broadcast_addr = "255.255.255.255:41234";  // Limited broadcast

    for i in 0..5 {
        let msg = format!("{{\"type\":\"DISCOVERY\",\"seq\":{},\"service\":\"my-service\"}}", i + 1);
        socket.send_to(msg.as_bytes(), broadcast_addr)?;
        println!("Broadcast #{}: {}", i + 1, msg);
        thread::sleep(Duration::from_secs(1));
    }

    println!("Done broadcasting");
    Ok(())
}
```

## UDP Broadcast Receiver

```rust
use std::net::UdpSocket;
use std::time::Duration;

fn main() -> std::io::Result<()> {
    let socket = UdpSocket::bind("0.0.0.0:41234")?;
    socket.set_broadcast(true)?;

    // 10-second receive timeout
    socket.set_read_timeout(Some(Duration::from_secs(10)))?;

    println!("Listening for broadcasts on port 41234...");

    let mut buf = [0u8; 4096];

    loop {
        match socket.recv_from(&mut buf) {
            Ok((n, src)) => {
                let msg = std::str::from_utf8(&buf[..n]).unwrap_or("<invalid>");
                println!("From {}: {}", src, msg);

                // Send unicast reply back to the sender
                let reply = format!("{{\"type\":\"DISCOVERY_REPLY\",\"port\":8080}}");
                socket.send_to(reply.as_bytes(), src)?;
            }
            Err(e) if e.kind() == std::io::ErrorKind::WouldBlock
                   || e.kind() == std::io::ErrorKind::TimedOut => {
                println!("Timeout: no broadcasts received for 10 seconds");
                break;
            }
            Err(e) => {
                eprintln!("Error: {}", e);
                break;
            }
        }
    }

    Ok(())
}
```

## Subnet-Directed Broadcast

```rust
use std::net::{UdpSocket, Ipv4Addr};

fn get_broadcast_address(ip: Ipv4Addr, prefix_len: u8) -> Ipv4Addr {
    let ip_u32 = u32::from(ip);
    let mask = if prefix_len == 0 { 0 } else { !0u32 << (32 - prefix_len) };
    Ipv4Addr::from(ip_u32 | !mask)
}

fn main() -> std::io::Result<()> {
    let my_ip: Ipv4Addr = "192.168.1.50".parse().unwrap();
    let broadcast = get_broadcast_address(my_ip, 24);
    println!("Subnet broadcast: {}", broadcast);  // 192.168.1.255

    let socket = UdpSocket::bind("0.0.0.0:0")?;
    socket.set_broadcast(true)?;

    let dest = format!("{}:41234", broadcast);
    socket.send_to(b"Hello local subnet!", dest.as_str())?;
    println!("Sent directed broadcast to {}", dest);

    Ok(())
}
```

## Combined Sender/Receiver

```rust
use std::net::UdpSocket;
use std::sync::Arc;
use std::thread;
use std::time::Duration;

fn main() -> std::io::Result<()> {
    // Create socket for both sending and receiving
    let socket = Arc::new(UdpSocket::bind("0.0.0.0:41234")?);
    socket.set_broadcast(true)?;

    let recv_socket = Arc::clone(&socket);

    // Receiver thread
    let recv_handle = thread::spawn(move || {
        recv_socket.set_read_timeout(Some(Duration::from_secs(15))).unwrap();
        let mut buf = [0u8; 4096];

        loop {
            match recv_socket.recv_from(&mut buf) {
                Ok((n, src)) => {
                    println!("[recv] From {}: {}", src,
                        std::str::from_utf8(&buf[..n]).unwrap_or("?"));
                }
                Err(_) => break,
            }
        }
    });

    // Sender: broadcast every second for 5 seconds
    for i in 0..5 {
        let msg = format!("Broadcast #{}", i);
        socket.send_to(msg.as_bytes(), "255.255.255.255:41234")?;
        thread::sleep(Duration::from_secs(1));
    }

    recv_handle.join().unwrap();
    Ok(())
}
```

## Conclusion

UDP broadcast in Rust requires calling `socket.set_broadcast(true)` before sending to broadcast addresses. Both `255.255.255.255` (limited broadcast) and subnet broadcast addresses (e.g., `192.168.1.255`) work after enabling this option. Receivers bind to the broadcast port and listen. This mechanism is perfect for local service discovery, peer detection, and network-wide announcements.
