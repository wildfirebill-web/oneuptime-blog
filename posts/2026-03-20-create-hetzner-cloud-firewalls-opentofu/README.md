# How to Create Hetzner Cloud Firewalls with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Hetzner Cloud, Firewall, Security, Infrastructure as Code

Description: Learn how to create Hetzner Cloud firewalls with OpenTofu to control inbound and outbound traffic to your servers.

Hetzner Cloud Firewalls are stateful, cloud-level firewalls that filter traffic before it reaches your server's network interface. OpenTofu lets you define firewall rules and attach them to servers by label selector or direct ID.

## Creating a Web Server Firewall

```hcl
resource "hcloud_firewall" "web" {
  name = "web-firewall"

  # Allow SSH from a management IP
  rule {
    direction  = "in"
    protocol   = "tcp"
    port       = "22"
    source_ips = ["203.0.113.10/32"]  # Replace with your IP
  }

  # Allow HTTP from anywhere
  rule {
    direction  = "in"
    protocol   = "tcp"
    port       = "80"
    source_ips = ["0.0.0.0/0", "::/0"]
  }

  # Allow HTTPS from anywhere
  rule {
    direction  = "in"
    protocol   = "tcp"
    port       = "443"
    source_ips = ["0.0.0.0/0", "::/0"]
  }

  # Allow ICMP (ping) for monitoring
  rule {
    direction  = "in"
    protocol   = "icmp"
    source_ips = ["0.0.0.0/0", "::/0"]
  }

  labels = {
    role       = "web"
    managed_by = "opentofu"
  }
}
```

## Attaching a Firewall to Servers by Label

```hcl
resource "hcloud_firewall_attachment" "web" {
  firewall_id = hcloud_firewall.web.id

  # Apply to all servers with label role=web
  label_selectors = ["role=web"]
}
```

## Attaching a Firewall to Specific Servers

```hcl
resource "hcloud_firewall_attachment" "specific" {
  firewall_id = hcloud_firewall.web.id
  server_ids  = [
    hcloud_server.web1.id,
    hcloud_server.web2.id,
  ]
}
```

## Database Server Firewall

```hcl
resource "hcloud_firewall" "database" {
  name = "database-firewall"

  # Allow PostgreSQL only from private network
  rule {
    direction  = "in"
    protocol   = "tcp"
    port       = "5432"
    source_ips = ["10.0.1.0/24"]  # Private network CIDR
  }

  # Allow Redis from private network
  rule {
    direction  = "in"
    protocol   = "tcp"
    port       = "6379"
    source_ips = ["10.0.1.0/24"]
  }
}
```

## Port Range Rules

```hcl
resource "hcloud_firewall" "app" {
  name = "app-firewall"

  # Allow a port range for application services
  rule {
    direction  = "in"
    protocol   = "tcp"
    port       = "8000-9000"
    source_ips = ["10.0.0.0/8"]
  }
}
```

## Outbound Rules

Hetzner firewalls allow all outbound traffic by default. If you need to restrict outbound:

```hcl
resource "hcloud_firewall" "restricted" {
  name = "restricted-firewall"

  # Only allow outbound HTTPS and DNS
  rule {
    direction         = "out"
    protocol          = "tcp"
    port              = "443"
    destination_ips   = ["0.0.0.0/0", "::/0"]
  }

  rule {
    direction         = "out"
    protocol          = "udp"
    port              = "53"
    destination_ips   = ["0.0.0.0/0", "::/0"]
  }
}
```

## Conclusion

Hetzner Cloud Firewalls are lightweight and free - there is no extra charge for using them. Define rules with protocol, port, and IP range filters, then attach firewalls to servers using label selectors for scalable, automatic application to new servers as they are created with matching labels.
