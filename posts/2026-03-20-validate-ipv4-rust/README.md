# How to Validate IPv4 Address Strings in Rust

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rust, IPv4, Validation, Ipv4Addr, Regex, Networking

Description: Learn multiple techniques to validate IPv4 address strings in Rust, from simple parse-based validation to custom range checks and regex patterns.

## Method 1: Using Ipv4Addr::parse (Recommended)

```rust
use std::net::Ipv4Addr;

fn is_valid_ipv4(s: &str) -> bool {
    s.parse::<Ipv4Addr>().is_ok()
}

fn main() {
    let test_cases = [
        ("192.168.1.1", true),
        ("0.0.0.0", true),
        ("255.255.255.255", true),
        ("10.0.0.256", false),     // Octet > 255
        ("1.2.3", false),          // Missing octet
        ("1.2.3.4.5", false),      // Too many octets
        ("::1", false),            // IPv6
        ("192.168.01.1", false),   // Leading zero (rejected by Rust's parser)
        ("", false),
    ];

    for (ip, expected) in test_cases {
        let valid = is_valid_ipv4(ip);
        let status = if valid == expected { "PASS" } else { "FAIL" };
        println!("[{}] {} -> {}", status, ip, valid);
    }
}
```

## Method 2: Validate and Return Parsed Address

```rust
use std::net::Ipv4Addr;

fn parse_ipv4(s: &str) -> Option<Ipv4Addr> {
    s.parse().ok()
}

fn main() {
    match parse_ipv4("192.168.1.50") {
        Some(ip) => println!("Valid: {} (private: {})", ip, ip.is_private()),
        None => println!("Invalid IP address"),
    }
}
```

## Method 3: Custom Validator with Detailed Errors

```rust
#[derive(Debug)]
enum Ipv4Error {
    WrongPartCount(usize),
    InvalidOctet { octet: String, position: usize },
    OctetOutOfRange { value: u16, position: usize },
}

fn validate_ipv4_detailed(s: &str) -> Result<[u8; 4], Ipv4Error> {
    let parts: Vec<&str> = s.split('.').collect();

    if parts.len() != 4 {
        return Err(Ipv4Error::WrongPartCount(parts.len()));
    }

    let mut octets = [0u8; 4];
    for (i, part) in parts.iter().enumerate() {
        let n: u16 = part.parse().map_err(|_| Ipv4Error::InvalidOctet {
            octet: part.to_string(),
            position: i,
        })?;

        if n > 255 {
            return Err(Ipv4Error::OctetOutOfRange { value: n, position: i });
        }

        octets[i] = n as u8;
    }

    Ok(octets)
}

fn main() {
    let cases = ["192.168.1.1", "10.0.0.256", "1.2.3", "bad.ip.here.x"];
    for s in cases {
        match validate_ipv4_detailed(s) {
            Ok(octets) => println!("{} -> octets: {:?}", s, octets),
            Err(e) => println!("{} -> Error: {:?}", s, e),
        }
    }
}
```

## Method 4: Regex Validation

```toml
# Cargo.toml
[dependencies]
regex = "1"
```

```rust
use regex::Regex;
use std::sync::OnceLock;

static IPV4_REGEX: OnceLock<Regex> = OnceLock::new();

fn ipv4_regex() -> &'static Regex {
    IPV4_REGEX.get_or_init(|| {
        // Matches valid IPv4: each octet 0-255, no leading zeros
        Regex::new(
            r"^(?:(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)\.){3}(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)$"
        ).unwrap()
    })
}

fn is_valid_ipv4_regex(s: &str) -> bool {
    ipv4_regex().is_match(s)
}

// Note: prefer Ipv4Addr::parse() over regex for correctness
```

## Method 5: IP Range Validation

```rust
use std::net::Ipv4Addr;

fn is_in_cidr(ip_str: &str, network: Ipv4Addr, prefix_len: u8) -> bool {
    let ip: Ipv4Addr = match ip_str.parse() {
        Ok(ip) => ip,
        Err(_) => return false,
    };

    let ip_u32 = u32::from(ip);
    let net_u32 = u32::from(network);
    let mask = if prefix_len == 0 { 0 } else { !0u32 << (32 - prefix_len) };

    (ip_u32 & mask) == (net_u32 & mask)
}

fn main() {
    let network = Ipv4Addr::new(192, 168, 1, 0);
    println!("{}", is_in_cidr("192.168.1.50", network, 24));  // true
    println!("{}", is_in_cidr("10.0.0.1", network, 24));      // false
}
```

## Conclusion

The idiomatic Rust approach is to validate IPv4 addresses by attempting to parse them with `s.parse::<Ipv4Addr>()`. This handles all edge cases: octet range checking, correct separator count, and leading zero rejection. Avoid custom string parsing or regex when `Ipv4Addr`'s built-in parser is available—it's both faster and more correct.
