# How to Create Linode Firewalls with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Linode, Firewalls, Security, Infrastructure as Code

Description: Learn how to create Linode Cloud Firewalls with OpenTofu to control inbound and outbound traffic to your Linode instances.

Linode Cloud Firewalls are stateful, cloud-level firewalls that filter traffic before it reaches your instances. OpenTofu lets you define firewall rules and assign firewalls to Linode instances as code.

## Creating a Web Server Firewall

```hcl
resource "linode_firewall" "web" {
  label   = "web-firewall"
  tags    = ["production", "web"]

  inbound_policy  = "DROP"   # Block all inbound by default
  outbound_policy = "ACCEPT" # Allow all outbound by default

  # Allow SSH from specific IP
  inbound {
    label    = "allow-ssh"
    action   = "ACCEPT"
    protocol = "TCP"
    ports    = "22"
    ipv4     = ["203.0.113.10/32"]
  }

  # Allow HTTP from anywhere
  inbound {
    label    = "allow-http"
    action   = "ACCEPT"
    protocol = "TCP"
    ports    = "80"
    ipv4     = ["0.0.0.0/0"]
    ipv6     = ["::/0"]
  }

  # Allow HTTPS from anywhere
  inbound {
    label    = "allow-https"
    action   = "ACCEPT"
    protocol = "TCP"
    ports    = "443"
    ipv4     = ["0.0.0.0/0"]
    ipv6     = ["::/0"]
  }

  # Allow ICMP for monitoring
  inbound {
    label    = "allow-icmp"
    action   = "ACCEPT"
    protocol = "ICMP"
    ipv4     = ["0.0.0.0/0"]
    ipv6     = ["::/0"]
  }

  # Assign to specific Linodes
  linodes = linode_instance.web[*].id
}
```

## Port Range Rules

```hcl
resource "linode_firewall" "app" {
  label           = "app-firewall"
  inbound_policy  = "DROP"
  outbound_policy = "ACCEPT"

  inbound {
    label    = "allow-app-range"
    action   = "ACCEPT"
    protocol = "TCP"
    ports    = "8000-9000"  # Port range
    ipv4     = ["10.0.0.0/8"]
  }
}
```

## Database Firewall

```hcl
resource "linode_firewall" "database" {
  label           = "database-firewall"
  inbound_policy  = "DROP"
  outbound_policy = "ACCEPT"

  inbound {
    label    = "allow-postgres"
    action   = "ACCEPT"
    protocol = "TCP"
    ports    = "5432"
    ipv4     = ["10.0.0.0/16"]  # Private network only
  }

  inbound {
    label    = "allow-redis"
    action   = "ACCEPT"
    protocol = "TCP"
    ports    = "6379"
    ipv4     = ["10.0.0.0/16"]
  }

  linodes = [linode_instance.database.id]
}
```

## Attaching a Firewall to Multiple Instances

```hcl
resource "linode_instance" "web" {
  count = 3
  # ...
}

resource "linode_firewall" "web" {
  label           = "web-firewall"
  inbound_policy  = "DROP"
  outbound_policy = "ACCEPT"

  inbound {
    label    = "allow-https"
    action   = "ACCEPT"
    protocol = "TCP"
    ports    = "443"
    ipv4     = ["0.0.0.0/0"]
    ipv6     = ["::/0"]
  }

  # Attach to all web instances
  linodes = linode_instance.web[*].id
}
```

## Conclusion

Linode Cloud Firewalls use a default-deny-inbound approach with explicit allow rules. Define rules with protocol, port, and IP range filters. Assign firewalls directly to Linode instances by ID in the `linodes` argument. Use separate firewall resources for different tiers (web, app, database) with different rule sets.
