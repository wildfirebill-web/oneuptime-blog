# How to Set Up Corporate Proxy Environment Variables for IPv4 on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Proxy, IPv4, Environment Variables, Corporate, HTTP

Description: Configure http_proxy, https_proxy, and no_proxy environment variables on Linux for IPv4 corporate proxy access, covering shell sessions, system-wide configuration, and tool-specific settings.

## Introduction

A corporate proxy intercepts and controls internet traffic. Most Linux command-line tools (curl, wget, apt, pip, npm, git) respect `http_proxy`, `https_proxy`, and `no_proxy` environment variables. Configuring these correctly ensures tools work through the proxy without bypassing internal services.

## Setting Proxy for the Current Shell Session

```bash
# Lowercase (most tools)
export http_proxy="http://proxy.corp.example.com:3128"
export https_proxy="http://proxy.corp.example.com:3128"
export ftp_proxy="http://proxy.corp.example.com:3128"

# Uppercase (some tools require UPPERCASE)
export HTTP_PROXY="http://proxy.corp.example.com:3128"
export HTTPS_PROXY="http://proxy.corp.example.com:3128"

# Bypass proxy for internal addresses
export no_proxy="localhost,127.0.0.1,10.0.0.0/8,192.168.0.0/16,.corp.example.com"
export NO_PROXY="localhost,127.0.0.1,10.0.0.0/8,192.168.0.0/16,.corp.example.com"
```

## Persistent User-Level Configuration

```bash
# Add to ~/.bashrc or ~/.zshrc
cat >> ~/.bashrc << 'EOF'

# Corporate proxy settings
export http_proxy="http://proxy.corp.example.com:3128"
export https_proxy="http://proxy.corp.example.com:3128"
export HTTP_PROXY="$http_proxy"
export HTTPS_PROXY="$https_proxy"
export no_proxy="localhost,127.0.0.1,10.0.0.0/8,192.168.0.0/16,.corp.example.com"
export NO_PROXY="$no_proxy"
EOF

source ~/.bashrc
```

## System-Wide Configuration (All Users)

```bash
# /etc/environment — read at login for all users
sudo tee -a /etc/environment << 'EOF'
http_proxy="http://proxy.corp.example.com:3128"
https_proxy="http://proxy.corp.example.com:3128"
HTTP_PROXY="http://proxy.corp.example.com:3128"
HTTPS_PROXY="http://proxy.corp.example.com:3128"
no_proxy="localhost,127.0.0.1,10.0.0.0/8,192.168.0.0/16"
NO_PROXY="localhost,127.0.0.1,10.0.0.0/8,192.168.0.0/16"
EOF
```

Note: `/etc/environment` uses key=value format (no `export`).

## Proxy with Username/Password

```bash
# URL-encode special characters in passwords
# @ → %40, : → %3A, # → %23

export http_proxy="http://username:p%40ssw0rd@proxy.corp.example.com:3128"
export https_proxy="http://username:p%40ssw0rd@proxy.corp.example.com:3128"
```

## Tool-Specific Configuration

### apt (Debian/Ubuntu)

```bash
sudo tee /etc/apt/apt.conf.d/95proxy << 'EOF'
Acquire::http::Proxy "http://proxy.corp.example.com:3128";
Acquire::https::Proxy "http://proxy.corp.example.com:3128";
EOF
```

### pip (Python)

```bash
# Temporary
pip install package --proxy http://proxy.corp.example.com:3128

# Persistent (~/.pip/pip.conf)
mkdir -p ~/.pip
tee ~/.pip/pip.conf << 'EOF'
[global]
proxy = http://proxy.corp.example.com:3128
EOF
```

### git

```bash
git config --global http.proxy "http://proxy.corp.example.com:3128"
git config --global https.proxy "http://proxy.corp.example.com:3128"

# For self-signed proxy certificates
git config --global http.sslVerify false  # Not recommended for production
```

### curl

```bash
# One-off
curl --proxy http://proxy.corp.example.com:3128 https://example.com

# Persistent (~/.curlrc)
echo 'proxy = "http://proxy.corp.example.com:3128"' >> ~/.curlrc
```

### wget

```bash
# Persistent (~/.wgetrc)
echo "http_proxy = http://proxy.corp.example.com:3128" >> ~/.wgetrc
echo "https_proxy = http://proxy.corp.example.com:3128" >> ~/.wgetrc
echo "use_proxy = on" >> ~/.wgetrc
```

## Testing Proxy Configuration

```bash
# Verify proxy variable is set
echo $http_proxy

# Test HTTP through proxy
curl -v http://example.com 2>&1 | grep -E "(< HTTP|Connected|Trying)"

# Test HTTPS through proxy (CONNECT tunnel)
curl -v https://example.com 2>&1 | grep -E "(< HTTP|CONNECT|SSL)"

# Check which IP is seen by the server
curl https://icanhazip.com
# Should return the proxy's IP, not your local IP
```

## Removing Proxy for a Single Command

```bash
# Unset proxy for one command
env -u http_proxy -u https_proxy curl https://internal-service.corp.example.com
```

## Conclusion

Set `http_proxy`, `https_proxy`, and `no_proxy` (both lowercase and uppercase) for comprehensive coverage. Add to `~/.bashrc` for user-level or `/etc/environment` for system-wide persistence. Configure tool-specific files (`/etc/apt/apt.conf.d/`, `~/.pip/pip.conf`) for package managers. Always populate `no_proxy` with internal CIDR ranges and domain suffixes to prevent routing internal traffic through the corporate proxy.
