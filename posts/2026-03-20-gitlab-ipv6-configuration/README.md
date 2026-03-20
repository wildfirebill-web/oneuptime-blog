# How to Configure GitLab for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GitLab, IPv6, DevOps, Git, Nginx, Linux, Self-Hosted

Description: Configure GitLab to serve web UI, Git operations, and APIs over IPv6 by updating the Nginx listener configuration and external URL settings.

---

GitLab (Omnibus package) uses a bundled Nginx. Enabling IPv6 requires updating the external URL setting and configuring the bundled Nginx to listen on IPv6 addresses alongside IPv4.

## Configuring GitLab for IPv6

```ruby
# /etc/gitlab/gitlab.rb

# Set external URL with FQDN (must have AAAA record for IPv6)

external_url 'https://gitlab.example.com'

# Enable Nginx to listen on IPv6
nginx['listen_addresses'] = ['0.0.0.0', '::']

# If using IPv6-only
# nginx['listen_addresses'] = ['::']

# For HTTP (non-SSL)
nginx['listen_port'] = 80

# HTTPS redirect
nginx['redirect_http_to_https'] = true

# GitLab Shell SSH over IPv6
gitlab_rails['gitlab_ssh_host'] = 'gitlab.example.com'

# SSH port
gitlab_shell['ssh_port'] = 22
```

```bash
# Apply configuration
sudo gitlab-ctl reconfigure

# Check Nginx is listening on IPv6
sudo ss -6 -tlnp | grep nginx

# Reload Nginx
sudo gitlab-ctl restart nginx
```

## SSH over IPv6 for Git Operations

```bash
# GitLab SSH daemon configuration
# /etc/gitlab/gitlab.rb
gitlab_rails['gitlab_shell_ssh_host'] = 'gitlab.example.com'

# Ensure sshd is listening on IPv6
# /etc/ssh/sshd_config
# AddressFamily any
# ListenAddress 0.0.0.0
# ListenAddress ::

# Restart SSH
sudo systemctl restart sshd

# Verify SSH on IPv6
ss -6 -tlnp | grep :22

# Clone repository over IPv6 SSH
git clone git@gitlab.example.com:user/repo.git
# (Requires AAAA record for gitlab.example.com)
```

## GitLab Registry over IPv6

```ruby
# /etc/gitlab/gitlab.rb - Container Registry IPv6

registry_external_url 'https://registry.example.com'
registry_nginx['listen_addresses'] = ['0.0.0.0', '::']

# After reconfigure:
# registry.example.com must have AAAA record
```

## GitLab Pages over IPv6

```ruby
# /etc/gitlab/gitlab.rb - Pages IPv6

pages_external_url 'https://pages.example.com'
gitlab_pages['listen_https'] = ['::', '0.0.0.0']
gitlab_pages['listen_http'] = ['::', '0.0.0.0']
```

## GitLab Runner over IPv6

```bash
# Register GitLab Runner to connect to GitLab over IPv6
sudo gitlab-runner register \
  --url "https://gitlab.example.com/" \
  --registration-token "TOKEN" \
  --executor "shell" \
  --description "IPv6 Runner"

# Verify runner can reach GitLab via IPv6
curl -6 https://gitlab.example.com/api/v4/version \
  -H "PRIVATE-TOKEN: your-token"
```

## Firewall Rules for GitLab IPv6

```bash
# HTTP/HTTPS for web interface
sudo ip6tables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo ip6tables -A INPUT -p tcp --dport 443 -j ACCEPT

# SSH for Git operations
sudo ip6tables -A INPUT -p tcp --dport 22 -j ACCEPT

# Container Registry
sudo ip6tables -A INPUT -p tcp --dport 5050 -j ACCEPT

sudo ip6tables-save > /etc/ip6tables/rules.v6
```

## Testing GitLab IPv6 Access

```bash
# Test GitLab web over IPv6
curl -6 -I https://gitlab.example.com/

# Test GitLab API over IPv6
curl -6 https://gitlab.example.com/api/v4/version

# Clone via HTTPS over IPv6
git clone https://gitlab.example.com/group/project.git

# Test SSH connectivity (for Git operations)
ssh -T git@gitlab.example.com -6
# Expected: Welcome to GitLab, @username!

# Check GitLab logs for IPv6 connections
sudo gitlab-ctl tail nginx | grep "2001:"
```

GitLab Omnibus IPv6 support is enabled via `nginx['listen_addresses'] = ['0.0.0.0', '::']` in `gitlab.rb`, providing dual-stack access to the web interface, API, Container Registry, and Pages with a single reconfigure operation.
