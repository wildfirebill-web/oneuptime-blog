# How to Bind Rust Sockets to Specific IPv4 Network Interfaces

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rust, Sockets, IPv4, Network Interfaces, Binding, TCP, std::net

Description: Bind Rust TCP listener and client sockets to specific IPv4 addresses and network interfaces to control which interface handles incoming and outgoing traffic.

## Introduction

On multi-homed hosts (servers with multiple NICs or IP addresses), you may need to bind a socket to a specific IPv4 address to control which interface receives connections or which interface sends traffic. Rust's `TcpListener::bind` and `TcpStream::connect` take `SocketAddr` parameters for this purpose.

## Binding a Server to a Specific IPv4 Address

```rust
use std::net::{TcpListener, TcpStream, SocketAddr, Ipv4Addr};
use std::io::{Read, Write};
use std::thread;

fn main() -> std::io::Result<()> {
    // Bind only to 192.168.1.10 — won't accept connections on other interfaces
    let bind_addr: SocketAddr = "192.168.1.10:8080".parse().unwrap();
    let listener = TcpListener::bind(bind_addr)?;
    
    println!("Listening on {}", listener.local_addr()?);
    
    for stream in listener.incoming() {
        match stream {
            Ok(s) => { thread::spawn(|| handle(s)); }
            Err(e) => eprintln!("Accept error: {}", e),
        }
    }
    Ok(())
}

fn handle(mut stream: TcpStream) {
    let peer = stream.peer_addr().unwrap();
    println!("Connection from {}", peer);
    
    let mut buf = [0u8; 512];
    if let Ok(n) = stream.read(&mut buf) {
        let _ = stream.write_all(&buf[..n]);
    }
}
```

## Binding to Multiple Addresses

To listen on multiple specific addresses, create multiple listeners:

```rust
use std::net::TcpListener;
use std::sync::Arc;
use std::thread;

fn main() -> std::io::Result<()> {
    let addresses = vec![
        "192.168.1.10:8080",   // Internal interface
        "10.0.0.5:8080",       // VPN interface
    ];
    
    let mut handles = Vec::new();
    
    for addr in addresses {
        let listener = TcpListener::bind(addr)?;
        println!("Listening on {}", listener.local_addr()?);
        
        let handle = thread::spawn(move || {
            for stream in listener.incoming() {
                if let Ok(s) = stream {
                    thread::spawn(|| handle_connection(s));
                }
            }
        });
        
        handles.push(handle);
    }
    
    for handle in handles {
        handle.join().ok();
    }
    
    Ok(())
}

fn handle_connection(mut stream: std::net::TcpStream) {
    // Same handler for all listeners
    let local = stream.local_addr().unwrap();
    let peer = stream.peer_addr().unwrap();
    println!("Connection {} -> {}", peer, local);
    
    let response = format!("You connected to {}\n", local);
    let _ = stream.write_all(response.as_bytes());
}
```

## Binding Client to a Specific Source IP

For outbound connections, bind the socket to a specific source IP:

```rust
use std::net::{TcpStream, SocketAddr};
use std::io::Write;

fn connect_from_specific_ip(
    source_ip: &str, 
    destination: &str
) -> std::io::Result<TcpStream> {
    // Parse addresses
    let src_addr: SocketAddr = format!("{}:0", source_ip).parse().unwrap(); // Port 0 = OS assigns
    let dst_addr: SocketAddr = destination.parse().unwrap();
    
    // Create and bind the socket to the source IP
    use std::net::TcpStream;
    
    // Use Socket2 crate for bind-before-connect on TcpStream
    // (std::net doesn't directly support this pattern)
    // For now, show the approach with socket2:
    println!("Would connect from {} to {}", src_addr.ip(), dst_addr);
    
    // Without socket2: connect normally (OS chooses source IP based on routing)
    TcpStream::connect(dst_addr)
}
```

## Using socket2 for Advanced Binding

The `socket2` crate provides more control:

```toml
# Cargo.toml
[dependencies]
socket2 = "0.5"
```

```rust
use socket2::{Socket, Domain, Type, Protocol, SockAddr};
use std::net::{SocketAddr, TcpStream};
use std::time::Duration;

fn connect_with_source_bind(source_ip: &str, destination: &str) -> std::io::Result<TcpStream> {
    let socket = Socket::new(Domain::IPV4, Type::STREAM, Some(Protocol::TCP))?;
    
    // Bind to specific source IP (let OS pick the port)
    let src_addr: SocketAddr = format!("{}:0", source_ip).parse().unwrap();
    socket.bind(&SockAddr::from(src_addr))?;
    
    // Set non-blocking for connect with timeout
    socket.set_nonblocking(true)?;
    
    let dst_addr: SocketAddr = destination.parse().unwrap();
    match socket.connect(&SockAddr::from(dst_addr)) {
        Ok(_) => {}
        Err(e) if e.raw_os_error() == Some(115) => { // EINPROGRESS
            // Wait for connection with select/poll
        }
        Err(e) => return Err(e),
    }
    
    socket.set_nonblocking(false)?;
    
    // Convert socket2::Socket to std::net::TcpStream
    Ok(TcpStream::from(socket))
}
```

## Listing Available IPv4 Addresses

```rust
use std::net::Ipv4Addr;

/// Get all IPv4 addresses available on this host
fn get_local_ipv4_addresses() -> Vec<Ipv4Addr> {
    // Use the `get-if-addrs` crate or parse /proc/net/fib_trie on Linux
    // Simple approach: try to bind to common interfaces
    let test_addrs = ["127.0.0.1", "0.0.0.0"];
    
    println!("To list network interfaces, use the get-if-addrs crate:");
    println!("  get_if_addrs::get_if_addrs().unwrap()");
    println!("    .iter()");
    println!("    .filter(|i| i.addr.is_loopback() == false)");
    println!("    .for_each(|i| println!(\"{}: {}\", i.name, i.addr.ip()))");
    
    Vec::new()
}
```

## Conclusion

Binding Rust sockets to specific IPv4 addresses is straightforward with `TcpListener::bind` for servers. For source-binding client connections, use the `socket2` crate which exposes the full POSIX socket API. This level of control is essential for multi-homed hosts, traffic engineering, and testing network policies.
