# How to Use IPv6 with Rust Actix-Web

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rust, IPv6, Actix-Web, HTTP, Web Framework, Networking

Description: Build IPv6-capable web applications with Rust's Actix-Web framework including binding, client IP extraction, and rate limiting by IPv6 prefix.

## Binding Actix-Web to IPv6

```toml
# Cargo.toml
[dependencies]
actix-web = "4"
tokio = { version = "1", features = ["full"] }
```

```rust
use actix_web::{get, web, App, HttpServer, Responder};

#[get("/")]
async fn index() -> impl Responder {
    "Hello from IPv6 Actix-Web!"
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    // [::] listens on all IPv6 interfaces (dual-stack on Linux)
    HttpServer::new(|| App::new().service(index))
        .bind("[::]:8080")?
        .run()
        .await
}
```

To bind to a specific IPv6 address:

```rust
HttpServer::new(|| App::new().service(index))
    .bind("[2001:db8::1]:8080")?  // specific address
    .bind("[::1]:8080")?          // loopback only
    .run()
    .await
```

## Extracting Client IPv6 Address

```rust
use actix_web::{get, HttpRequest, HttpResponse};
use std::net::IpAddr;

fn get_client_ip(req: &HttpRequest) -> Option<IpAddr> {
    // Check X-Forwarded-For first (when behind a proxy)
    if let Some(forwarded) = req.headers().get("X-Forwarded-For") {
        if let Ok(val) = forwarded.to_str() {
            // Take the first (leftmost) address
            if let Ok(ip) = val.split(',').next()?.trim().parse() {
                return Some(ip);
            }
        }
    }

    // Fall back to direct connection address
    req.peer_addr().map(|a| a.ip())
}

#[get("/info")]
async fn client_info(req: HttpRequest) -> HttpResponse {
    let ip = get_client_ip(&req);

    let body = match ip {
        Some(IpAddr::V6(v6)) => format!("Your IPv6 address: {}", v6),
        Some(IpAddr::V4(v4)) => format!("Your IPv4 address: {}", v4),
        None => "Could not determine client IP".to_string(),
    };

    HttpResponse::Ok().body(body)
}
```

## Middleware: IPv6 Request Logging

```rust
use actix_web::{
    dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
    Error,
};
use futures_util::future::{LocalBoxFuture, Ready, ready};
use std::net::IpAddr;

pub struct IPv6Logger;

impl<S, B> Transform<S, ServiceRequest> for IPv6Logger
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Transform = IPv6LoggerMiddleware<S>;
    type InitError = ();
    type Future = Ready<Result<Self::Transform, Self::InitError>>;

    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(IPv6LoggerMiddleware { service }))
    }
}

pub struct IPv6LoggerMiddleware<S> {
    service: S,
}

impl<S, B> Service<ServiceRequest> for IPv6LoggerMiddleware<S>
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Future = LocalBoxFuture<'static, Result<Self::Response, Self::Error>>;

    forward_ready!(service);

    fn call(&self, req: ServiceRequest) -> Self::Future {
        let ip = req.peer_addr().map(|a| a.ip());
        let method = req.method().to_string();
        let path = req.path().to_string();

        match ip {
            Some(IpAddr::V6(v6)) => println!("IPv6 {} {} from {}", method, path, v6),
            Some(IpAddr::V4(v4)) => println!("IPv4 {} {} from {}", method, path, v4),
            None => println!("{} {}", method, path),
        }

        let fut = self.service.call(req);
        Box::pin(async move { fut.await })
    }
}
```

## Shared State with IPv6 Addresses

```rust
use actix_web::{get, web, App, HttpServer, HttpResponse};
use std::net::Ipv6Addr;
use std::sync::Mutex;

struct AppState {
    allowed_prefixes: Mutex<Vec<Ipv6Addr>>,
}

#[get("/allowed")]
async fn list_allowed(data: web::Data<AppState>) -> HttpResponse {
    let prefixes = data.allowed_prefixes.lock().unwrap();
    let list: Vec<String> = prefixes.iter().map(|a| a.to_string()).collect();
    HttpResponse::Ok().json(list)
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let state = web::Data::new(AppState {
        allowed_prefixes: Mutex::new(vec![
            "2001:db8::1".parse().unwrap(),
            "2001:db8::2".parse().unwrap(),
        ]),
    });

    HttpServer::new(move || {
        App::new()
            .app_data(state.clone())
            .service(list_allowed)
    })
    .bind("[::]:8080")?
    .run()
    .await
}
```

## TLS/HTTPS on IPv6

```toml
# Cargo.toml
[dependencies]
actix-web = { version = "4", features = ["rustls-0_23"] }
rustls = "0.23"
rustls-pemfile = "2"
```

```rust
use actix_web::{get, App, HttpServer, Responder};
use rustls::ServerConfig;
use rustls_pemfile::{certs, pkcs8_private_keys};
use std::io::BufReader;
use std::fs::File;

fn load_tls_config() -> ServerConfig {
    let cert_file = &mut BufReader::new(File::open("cert.pem").unwrap());
    let key_file  = &mut BufReader::new(File::open("key.pem").unwrap());

    let cert_chain = certs(cert_file).map(|r| r.unwrap()).collect();
    let mut keys: Vec<_> = pkcs8_private_keys(key_file).map(|r| r.unwrap()).collect();

    ServerConfig::builder()
        .with_no_client_auth()
        .with_single_cert(cert_chain, rustls::pki_types::PrivateKeyDer::Pkcs8(keys.remove(0)))
        .unwrap()
}

#[get("/")]
async fn index() -> impl Responder { "HTTPS over IPv6!" }

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let tls_config = load_tls_config();

    HttpServer::new(|| App::new().service(index))
        .bind_rustls_0_23("[::]:443", tls_config)?
        .run()
        .await
}
```

## Conclusion

Actix-Web supports IPv6 by binding to `[::]:port` or specific IPv6 addresses. Use `req.peer_addr()` to get the client's socket address, checking `X-Forwarded-For` for proxied environments. Middleware provides per-request IPv6 logging and access control. TLS over IPv6 uses `bind_rustls` with a standard Rustls config. Actix-Web's dual-stack support means a single bind to `[::]:port` serves both IPv4 and IPv6 clients on Linux.
