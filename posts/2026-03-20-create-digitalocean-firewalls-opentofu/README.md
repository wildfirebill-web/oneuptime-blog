# How to Create DigitalOcean Firewalls with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, DigitalOcean, Firewalls, Security, Infrastructure as Code

Description: Learn how to create DigitalOcean cloud firewalls with OpenTofu to control inbound and outbound traffic to your Droplets.

DigitalOcean Cloud Firewalls are stateful, cloud-level firewalls applied to Droplets. Unlike OS-level firewalls, they filter traffic before it reaches the Droplet's network interface. OpenTofu lets you define firewall rules as code and apply them to Droplets by tag or direct reference.

## Creating a Basic Web Server Firewall

```hcl
resource "digitalocean_firewall" "web" {
  name = "web-firewall"

  # Apply to all Droplets tagged "web"
  tags = ["web"]

  # Allow SSH from a specific management IP
  inbound_rule {
    protocol         = "tcp"
    port_range       = "22"
    source_addresses = ["203.0.113.10/32"]  # Replace with your IP
  }

  # Allow HTTP from anywhere
  inbound_rule {
    protocol         = "tcp"
    port_range       = "80"
    source_addresses = ["0.0.0.0/0", "::/0"]
  }

  # Allow HTTPS from anywhere
  inbound_rule {
    protocol         = "tcp"
    port_range       = "443"
    source_addresses = ["0.0.0.0/0", "::/0"]
  }

  # Allow all outbound traffic
  outbound_rule {
    protocol              = "tcp"
    port_range            = "all"
    destination_addresses = ["0.0.0.0/0", "::/0"]
  }

  outbound_rule {
    protocol              = "udp"
    port_range            = "all"
    destination_addresses = ["0.0.0.0/0", "::/0"]
  }

  outbound_rule {
    protocol              = "icmp"
    destination_addresses = ["0.0.0.0/0", "::/0"]
  }
}
```

## Creating a Database Firewall

```hcl
resource "digitalocean_firewall" "database" {
  name = "database-firewall"

  # Apply only to database Droplets (not the managed DB — use database_firewall for that)
  tags = ["database"]

  # Allow PostgreSQL only from app Droplets by tag
  inbound_rule {
    protocol    = "tcp"
    port_range  = "5432"
    source_tags = ["app"]
  }

  # Allow Redis only from app Droplets
  inbound_rule {
    protocol    = "tcp"
    port_range  = "6379"
    source_tags = ["app"]
  }

  outbound_rule {
    protocol              = "tcp"
    port_range            = "all"
    destination_addresses = ["0.0.0.0/0", "::/0"]
  }
}
```

## Applying Firewalls to Specific Droplets

```hcl
resource "digitalocean_droplet" "bastion" {
  name   = "bastion"
  region = "nyc3"
  size   = "s-1vcpu-1gb"
  image  = "ubuntu-22-04-x64"
}

resource "digitalocean_firewall" "bastion" {
  name = "bastion-firewall"

  # Apply to a specific Droplet by ID (not by tag)
  droplet_ids = [digitalocean_droplet.bastion.id]

  inbound_rule {
    protocol         = "tcp"
    port_range       = "22"
    source_addresses = ["0.0.0.0/0", "::/0"]  # Public SSH for bastion
  }

  outbound_rule {
    protocol              = "tcp"
    port_range            = "all"
    destination_addresses = ["0.0.0.0/0", "::/0"]
  }
}
```

## Port Range Syntax

- Single port: `"22"`, `"80"`, `"443"`
- Port range: `"8000-9000"`
- All ports: `"all"` or `"1-65535"`

## Conclusion

DigitalOcean cloud firewalls in OpenTofu are defined with inbound and outbound rules using protocol, port range, and source/destination filters. Apply firewalls to Droplets by tag for scalable, automatic rule application as new Droplets are added to a tag, or by Droplet ID for targeted rules on specific instances.
