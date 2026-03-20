# How to Set Up GoAccess for Real-Time IPv4 Web Traffic Analysis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GoAccess, IPv4, Log Analysis, Real-Time, Web Traffic, Nginx, Apache

Description: Set up GoAccess to analyze Nginx and Apache access logs in real time, generate HTML reports with IPv4 traffic statistics, and use WebSocket streaming for live dashboard updates.

## Introduction

GoAccess is a fast, terminal-based and browser-based log analyzer. It parses access logs in real time, displaying IPv4 visitor statistics, top pages, status code distribution, and bandwidth usage with no database required.

## Install GoAccess

```bash
# Ubuntu/Debian

sudo apt install goaccess

# CentOS/RHEL
sudo yum install goaccess

# macOS
brew install goaccess
```

## Analyze Nginx Log in Terminal

```bash
# Interactive terminal dashboard (Nginx combined format)
goaccess /var/log/nginx/access.log \
  --log-format=COMBINED \
  -c   # Select log format interactively on first run
```

## Non-Interactive with Explicit Format

```bash
goaccess /var/log/nginx/access.log \
  --log-format=COMBINED \
  --no-query-string  # Merge URLs with different query strings
```

## Generate Static HTML Report

```bash
goaccess /var/log/nginx/access.log \
  --log-format=COMBINED \
  -o /var/www/html/report.html \
  --html-prefs='{"layout":"horizontal"}'

# Multiple log files (rotated logs)
cat /var/log/nginx/access.log* | goaccess \
  --log-format=COMBINED \
  -o /var/www/html/report.html
```

## Real-Time HTML Dashboard with WebSocket

```bash
# Start GoAccess as a daemon with WebSocket push updates
goaccess /var/log/nginx/access.log \
  --log-format=COMBINED \
  --real-time-html \
  -o /var/www/html/live-report.html \
  --daemonize \
  --ws-url=ws://your-server.com:7890 \
  --port=7890 \
  --persist \
  --restore
```

Nginx config to expose the report:

```nginx
server {
    listen 443 ssl;
    server_name analytics.example.com;

    location / {
        root /var/www/html;
        index live-report.html;
    }

    location /ws {
        proxy_pass http://127.0.0.1:7890;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

## Custom Log Format (with Real IP Header)

```bash
# If using X-Forwarded-For or X-Real-IP
goaccess /var/log/nginx/access.log \
  --log-format='%^ - %^ [%d:%t %^] "%r" %s %b "%R" "%u" %h' \
  --date-format='%d/%b/%Y' \
  --time-format='%H:%M:%S'
```

## Useful GoAccess Panels

```text
Visitors     - Unique IPv4 addresses per day
Requests     - Most requested URLs
Static Files - .css, .js, .png requests
404s         - Not found URLs
Hosts        - Full IPv4 address breakdown with OS/browser
Status Codes - 2xx/3xx/4xx/5xx distribution
Bandwidth    - Data transferred per IP
```

## Cron Job for Daily HTML Reports

```bash
# Generate report every hour
0 * * * * goaccess /var/log/nginx/access.log \
  --log-format=COMBINED \
  -o /var/www/html/hourly-report.html \
  >> /var/log/goaccess.log 2>&1
```

## Conclusion

GoAccess provides instant IPv4 web traffic analytics with zero infrastructure - just a binary and your log files. Use `--real-time-html` with WebSocket for a live browser dashboard, static HTML generation for periodic reports, and `cat access.log*` to include rotated logs. The **Hosts** panel gives a complete breakdown of every IPv4 address accessing your server.
