# How to Create a UDP Server and Client in Rust with UdpSocket for IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rust, UDP, UdpSocket, IPv4, Networking, Datagram

Description: Learn how to create a UDP server and client in Rust using std::net::UdpSocket for IPv4 datagram communication.

## UDP Server

```rust
use std::net::UdpSocket;
use std::io;

fn main() -> io::Result<()> {
    // Bind to all IPv4 interfaces on port 9001
    let socket = UdpSocket::bind("0.0.0.0:9001")?;
    println!("UDP server listening on {}", socket.local_addr()?);

    let mut buf = [0u8; 65535];

    loop {
        // recv_from blocks until a datagram arrives
        // Returns (bytes_received, sender_address)
        let (n, src) = socket.recv_from(&mut buf)?;
        let message = std::str::from_utf8(&buf[..n]).unwrap_or("<invalid utf-8>");
        println!("Received {} bytes from {}: {}", n, src, message);

        // Send echo reply back to the sender
        let reply = format!("Echo: {}", message);
        socket.send_to(reply.as_bytes(), src)?;
    }
}
```

## UDP Client

```rust
use std::net::UdpSocket;
use std::time::Duration;
use std::io;

fn main() -> io::Result<()> {
    // Bind to any available local port (OS chooses the port)
    let socket = UdpSocket::bind("0.0.0.0:0")?;

    // Set a receive timeout to avoid blocking forever
    socket.set_read_timeout(Some(Duration::from_secs(3)))?;

    let server_addr = "127.0.0.1:9001";
    let message = b"Hello, UDP server!";

    // send_to sends a datagram to the specified address
    socket.send_to(message, server_addr)?;
    println!("Sent {} bytes to {}", message.len(), server_addr);

    // Receive the reply
    let mut buf = [0u8; 4096];
    match socket.recv_from(&mut buf) {
        Ok((n, from)) => {
            let reply = std::str::from_utf8(&buf[..n]).unwrap_or("<invalid>");
            println!("Reply from {}: {}", from, reply);
        }
        Err(e) if e.kind() == io::ErrorKind::WouldBlock || e.kind() == io::ErrorKind::TimedOut => {
            eprintln!("No reply within timeout");
        }
        Err(e) => eprintln!("Receive error: {}", e),
    }

    Ok(())
}
```

## Using connect() for Fixed Remote Address

```rust
use std::net::UdpSocket;
use std::io;

fn main() -> io::Result<()> {
    let socket = UdpSocket::bind("0.0.0.0:0")?;

    // connect() sets the default remote address; no actual handshake
    socket.connect("127.0.0.1:9001")?;

    // After connect(), use send() and recv() instead of send_to()/recv_from()
    socket.send(b"Hello connected UDP!")?;

    let mut buf = [0u8; 4096];
    let n = socket.recv(&mut buf)?;
    println!("Reply: {}", std::str::from_utf8(&buf[..n]).unwrap_or("?"));

    Ok(())
}
```

## Concurrent UDP Server with Threads

```rust
use std::net::UdpSocket;
use std::sync::Arc;
use std::thread;

fn main() -> std::io::Result<()> {
    let socket = Arc::new(UdpSocket::bind("0.0.0.0:9001")?);
    println!("Concurrent UDP server on port 9001");

    let mut buf = [0u8; 65535];

    loop {
        let (n, src) = socket.recv_from(&mut buf)?;
        let data: Vec<u8> = buf[..n].to_vec();  // Copy before spawning thread
        let sock = Arc::clone(&socket);

        thread::spawn(move || {
            let msg = std::str::from_utf8(&data).unwrap_or("?");
            println!("Processing from {}: {}", src, msg);
            let reply = format!("Processed: {}", msg);
            if let Err(e) = sock.send_to(reply.as_bytes(), src) {
                eprintln!("Reply error: {}", e);
            }
        });
    }
}
```

## Error Handling for WouldBlock

```rust
use std::io;
use std::net::UdpSocket;

// WouldBlock means the operation couldn't complete without blocking.
// On timeout sockets it indicates the timeout expired.
fn try_recv(socket: &UdpSocket, buf: &mut [u8]) -> Option<(usize, std::net::SocketAddr)> {
    match socket.recv_from(buf) {
        Ok(result) => Some(result),
        Err(e) if e.kind() == io::ErrorKind::WouldBlock
                || e.kind() == io::ErrorKind::TimedOut => None,
        Err(e) => {
            eprintln!("Unexpected recv error: {}", e);
            None
        }
    }
}
```

## Conclusion

Rust UDP sockets use `UdpSocket::bind()` for both servers and clients (servers use a fixed port; clients use `"0.0.0.0:0"` for any ephemeral port). Use `recv_from()`/`send_to()` for connectionless communication, or `connect()` + `recv()`/`send()` for a fixed remote. Always clone buffer data before spawning threads since Rust's ownership rules prevent shared mutation.
