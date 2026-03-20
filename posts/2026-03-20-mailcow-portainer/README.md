# How to Deploy Mailcow via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Mailcow, Email, Self-Hosted, Mail Server

Description: Deploy Mailcow, the full-featured self-hosted email server suite, using Docker and manage it via Portainer for a complete self-hosted mail solution.

## Introduction

Mailcow is a Docker-based mail server suite that includes Postfix, Dovecot, SOGo webmail, antispam, and antivirus - all in one package. This guide shows you how to install Mailcow using its official installer and then import it into Portainer for ongoing management.

## Prerequisites

- A fresh server with a public IP address
- A fully qualified domain name (FQDN) like `mail.yourdomain.com`
- Port 25, 80, 443, 465, 587, 993, 995, 4190 available
- At least 3 GB RAM (6 GB recommended)
- Portainer installed on the same host

## Step 1: Configure DNS Records

Before installing, set up these DNS records:

```text
# A record

mail.yourdomain.com  A  your.server.ip

# MX record
yourdomain.com  MX  10  mail.yourdomain.com

# PTR record (reverse DNS - set with your server provider)
your.server.ip  PTR  mail.yourdomain.com

# SPF record
yourdomain.com  TXT  "v=spf1 mx ~all"

# DMARC record
_dmarc.yourdomain.com  TXT  "v=DMARC1; p=quarantine; rua=mailto:dmarc@yourdomain.com"
```

## Step 2: Install Mailcow

Mailcow provides an official installation script:

```bash
# Clone the Mailcow repository
cd /opt
git clone https://github.com/mailcow/mailcow-dockerized
cd mailcow-dockerized

# Run the generate config script
./generate_config.sh
# When prompted, enter your FQDN: mail.yourdomain.com
# Select timezone
```

This creates `/opt/mailcow-dockerized/mailcow.conf` with your configuration.

## Step 3: Review Generated docker-compose.yml

Mailcow generates a detailed compose file. Key services include:

```yaml
# /opt/mailcow-dockerized/docker-compose.yml (excerpt)
services:
  # SMTP server
  postfix-mailcow:
    image: mailcow/postfix:1.71
    # ...

  # IMAP/POP3 server
  dovecot-mailcow:
    image: mailcow/dovecot:1.8
    # ...

  # Webmail client
  sogo-mailcow:
    image: mailcow/sogo:1.128
    # ...

  # Antispam
  rspamd-mailcow:
    image: mailcow/rspamd:1.99
    # ...

  # Database
  mysql-mailcow:
    image: mariadb:10.5
    # ...
```

## Step 4: Start Mailcow

```bash
# Pull all images first
docker compose pull

# Start Mailcow
docker compose up -d

# Check all services are healthy
docker compose ps
```

## Step 5: Import Mailcow Stack into Portainer

To manage Mailcow from Portainer:

1. Go to **Stacks** → **Add Stack** → **Repository**
2. Enter the Git URL: `https://github.com/mailcow/mailcow-dockerized`
3. Set the **Compose path**: `docker-compose.yml`

Alternatively, use **Upload** and upload the generated compose file from `/opt/mailcow-dockerized/`.

> **Important**: Mailcow relies on environment variables from `mailcow.conf`. If using Portainer's stack import, you'll need to manually set those env vars in the Portainer stack editor.

## Step 6: Access the Mailcow Admin Panel

1. Open `https://mail.yourdomain.com`
2. Log in with `admin` / `moohoo` (default credentials)
3. **Immediately change the admin password**

## Step 7: Add a Mail Domain and Mailbox

In the Mailcow admin panel:
1. **Mail Setup** → **Domains** → **Add Domain**
2. Enter `yourdomain.com` and click **Add**
3. **Mail Setup** → **Mailboxes** → **Add Mailbox**
4. Create `user@yourdomain.com`

## Step 8: Get DKIM Keys

1. In the admin panel, go to **Mail Setup** → **Configuration** → **ARC/DKIM Keys**
2. Generate a new DKIM key for your domain
3. Add the displayed TXT record to your DNS

## Monitoring via Portainer

After setup, use Portainer to:
- View logs for each Mailcow container
- Monitor resource usage (CPU/RAM per service)
- Restart individual services (e.g., just Rspamd after config changes)
- Inspect container health checks

## Conclusion

Mailcow provides a production-ready, self-hosted email platform and Portainer makes it easy to monitor and manage the ~15 containers it runs. With proper DNS configuration, you'll have a fully functional mail server that rivals hosted email services - all under your control.
