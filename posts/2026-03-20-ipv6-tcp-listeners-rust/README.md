# How to Create IPv6 TCP Listeners in Rust

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rust, IPv6, TCP, Networking, Tokio, Async

Description: Create IPv6 TCP listeners in Rust using std::net for synchronous servers and Tokio for high-performance async servers.

## Synchronous IPv6 TCP Listener

The `std::net::TcpListener` supports IPv6 when bound to a `[::]:port` address:

```rust
use std::io::{BufRead, BufReader, Write};
use std::net::{TcpListener, TcpStream};
use std::thread;

fn handle_client(mut stream: TcpStream) {
    let peer = stream.peer_addr().unwrap();
    println!("Connection from {}", peer);

    let reader = BufReader::new(stream.try_clone().unwrap());
    for line in reader.lines() {
        match line {
            Ok(l) => {
                println!("[{}] {}", peer, l);
                stream.write_all(format!("echo: {}\n", l).as_bytes()).ok();
            }
            Err(_) => break,
        }
    }
    println!("Disconnected: {}", peer);
}

fn main() -> std::io::Result<()> {
    // [::] listens on all IPv6 interfaces (and IPv4 on dual-stack systems)
    let listener = TcpListener::bind("[::]:8080")?;
    println!("Listening on {}", listener.local_addr()?);

    for stream in listener.incoming() {
        match stream {
            Ok(s) => {
                thread::spawn(|| handle_client(s));
            }
            Err(e) => eprintln!("Accept error: {}", e),
        }
    }

    Ok(())
}
```

## IPv6-Only Listener

To bind exclusively to IPv6 (disabling dual-stack), use `SocketAddrV6` and set `IPV6_V6ONLY`:

```rust
use std::net::{Ipv6Addr, SocketAddrV6, TcpListener};

#[cfg(unix)]
fn set_ipv6only(listener: &TcpListener) -> std::io::Result<()> {
    use std::os::unix::io::AsRawFd;
    let fd = listener.as_raw_fd();
    let one: libc::c_int = 1;
    unsafe {
        let ret = libc::setsockopt(
            fd,
            libc::IPPROTO_IPV6,
            libc::IPV6_V6ONLY,
            &one as *const _ as *const libc::c_void,
            std::mem::size_of::<libc::c_int>() as libc::socklen_t,
        );
        if ret != 0 {
            return Err(std::io::Error::last_os_error());
        }
    }
    Ok(())
}

fn main() -> std::io::Result<()> {
    let addr = SocketAddrV6::new(Ipv6Addr::UNSPECIFIED, 8080, 0, 0);
    let listener = TcpListener::bind(addr)?;

    #[cfg(unix)]
    set_ipv6only(&listener)?;

    println!("IPv6-only listener on {}", listener.local_addr()?);
    Ok(())
}
```

## Async IPv6 TCP Server with Tokio

```toml
# Cargo.toml

[dependencies]
tokio = { version = "1", features = ["full"] }
```

```rust
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::net::{TcpListener, TcpStream};

async fn handle_client(stream: TcpStream) {
    let peer = stream.peer_addr().unwrap();
    println!("Connected: {}", peer);

    let (reader, mut writer) = stream.into_split();
    let mut lines = BufReader::new(reader).lines();

    while let Ok(Some(line)) = lines.next_line().await {
        println!("[{}] {}", peer, line);
        if writer.write_all(format!("echo: {}\n", line).as_bytes()).await.is_err() {
            break;
        }
    }

    println!("Disconnected: {}", peer);
}

#[tokio::main]
async fn main() -> tokio::io::Result<()> {
    let listener = TcpListener::bind("[::]:8080").await?;
    println!("Async IPv6 server on {}", listener.local_addr()?);

    loop {
        let (socket, _addr) = listener.accept().await?;
        tokio::spawn(handle_client(socket));
    }
}
```

## Dual-Stack Listener Handling

When `[::]:port` accepts both IPv4 and IPv6 connections, the IPv4 addresses appear as IPv4-mapped IPv6 addresses:

```rust
use std::net::IpAddr;
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> tokio::io::Result<()> {
    let listener = TcpListener::bind("[::]:9090").await?;
    println!("Dual-stack server on {}", listener.local_addr()?);

    loop {
        let (socket, addr) = listener.accept().await?;

        let client_ip = match addr.ip() {
            IpAddr::V6(v6) => {
                // Convert IPv4-mapped to real IPv4
                v6.to_ipv4().map(IpAddr::V4).unwrap_or(IpAddr::V6(v6))
            }
            v4 => v4,
        };

        println!("Client IP: {} (real)", client_ip);
        drop(socket);
    }
}
```

## Graceful Shutdown

```rust
use tokio::net::TcpListener;
use tokio::signal;

#[tokio::main]
async fn main() -> tokio::io::Result<()> {
    let listener = TcpListener::bind("[::]:8080").await?;
    println!("Server on {}", listener.local_addr()?);

    loop {
        tokio::select! {
            result = listener.accept() => {
                match result {
                    Ok((socket, addr)) => {
                        println!("Accepted from {}", addr);
                        tokio::spawn(async move {
                            drop(socket); // handle connection here
                        });
                    }
                    Err(e) => eprintln!("Accept error: {}", e),
                }
            }
            _ = signal::ctrl_c() => {
                println!("Shutting down");
                break;
            }
        }
    }

    Ok(())
}
```

## Conclusion

Rust makes IPv6 TCP servers straightforward. Binding to `[::]:port` creates a dual-stack listener on most platforms. For async servers, Tokio's `TcpListener` mirrors the synchronous API. Spawning each connection into its own task provides high concurrency without blocking. Use `peer_addr()` to identify clients and handle IPv4-mapped addresses when running in dual-stack mode.
