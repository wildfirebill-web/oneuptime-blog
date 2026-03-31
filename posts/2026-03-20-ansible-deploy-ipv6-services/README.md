# How to Deploy IPv6 Services with Ansible

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ansible, IPv6, Service, Nginx, Apache, Deployment, Automation

Description: A guide to deploying IPv6-enabled services (nginx, Apache, SSH) on Linux servers using Ansible, ensuring services bind to IPv6 addresses.

Deploying an IPv6 service requires more than just enabling IPv6 on the network - the application must also be configured to listen on IPv6 addresses. This guide covers deploying and verifying nginx, Apache, and SSH for IPv6 with Ansible.

## Deploy nginx with IPv6 Support

```yaml
# deploy-nginx-ipv6.yml - Install and configure nginx to listen on IPv6

---
- name: Deploy nginx with IPv6 support
  hosts: web_servers
  become: true

  vars:
    server_name: "example.com"
    ipv6_address: "::"  # Listen on all IPv6 addresses
    port: 80

  tasks:
    - name: Install nginx
      ansible.builtin.package:
        name: nginx
        state: present

    - name: Write nginx virtual host configuration
      ansible.builtin.template:
        src: nginx-ipv6.conf.j2
        dest: "/etc/nginx/sites-available/{{ server_name }}"
        mode: "0644"
      notify: Reload nginx

    - name: Enable the virtual host
      ansible.builtin.file:
        src: "/etc/nginx/sites-available/{{ server_name }}"
        dest: "/etc/nginx/sites-enabled/{{ server_name }}"
        state: link
      notify: Reload nginx

    - name: Ensure nginx is running and enabled
      ansible.builtin.systemd:
        name: nginx
        state: started
        enabled: true

  handlers:
    - name: Reload nginx
      ansible.builtin.systemd:
        name: nginx
        state: reloaded
```

nginx template with IPv6:

```nginx
# templates/nginx-ipv6.conf.j2 - nginx config with dual-stack listeners
server {
    # Listen on all IPv4 addresses
    listen 0.0.0.0:{{ port }};

    # Listen on all IPv6 addresses (brackets required in nginx)
    listen [{{ ipv6_address }}]:{{ port }};

    server_name {{ server_name }};

    location / {
        root /var/www/{{ server_name }};
        index index.html;
    }
}
```

## Deploy Apache with IPv6 Support

```yaml
# deploy-apache-ipv6.yml
---
- name: Configure Apache to listen on IPv6
  hosts: web_servers
  become: true

  tasks:
    - name: Ensure Apache listens on IPv6 in ports.conf
      ansible.builtin.lineinfile:
        path: /etc/apache2/ports.conf
        line: "Listen [::]:80"
        insertafter: "^Listen 80"
      notify: Restart Apache

    - name: Write Apache virtual host for IPv6
      ansible.builtin.copy:
        dest: /etc/apache2/sites-available/000-default.conf
        content: |
          # Listen on all IPv4 and IPv6 addresses
          <VirtualHost *:80>
              ServerName example.com
              DocumentRoot /var/www/html
              ErrorLog ${APACHE_LOG_DIR}/error.log
              CustomLog ${APACHE_LOG_DIR}/access.log combined
          </VirtualHost>
        mode: "0644"
      notify: Restart Apache

  handlers:
    - name: Restart Apache
      ansible.builtin.systemd:
        name: apache2
        state: restarted
```

## Configure SSH for IPv6 Access

```yaml
# configure-ssh-ipv6.yml - Configure SSH daemon to listen on IPv6
---
- name: Configure SSH to listen on IPv6
  hosts: all
  become: true

  tasks:
    - name: Configure SSH to listen on all addresses (IPv4 and IPv6)
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "^#?ListenAddress"
        line: "ListenAddress ::"
        backrefs: false
      notify: Restart SSH

    - name: Ensure AddressFamily is any (allow both IPv4 and IPv6)
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "^#?AddressFamily"
        line: "AddressFamily any"
      notify: Restart SSH

  handlers:
    - name: Restart SSH
      ansible.builtin.systemd:
        name: sshd
        state: restarted
```

## Verify Services Are Listening on IPv6

```yaml
# verify-ipv6-services.yml
---
- name: Verify services listen on IPv6
  hosts: web_servers
  become: true

  tasks:
    - name: Check which addresses nginx is listening on
      ansible.builtin.command:
        cmd: ss -6 -tlnp | grep nginx
      register: nginx_listen
      changed_when: false

    - name: Assert nginx is listening on IPv6
      ansible.builtin.assert:
        that:
          - "':::80' in nginx_listen.stdout or '[::]:80' in nginx_listen.stdout"
        fail_msg: "nginx is not listening on IPv6 port 80"

    - name: Test HTTP via IPv6
      ansible.builtin.uri:
        url: "http://[::1]/"
        status_code: 200
```

## Run

```bash
ansible-playbook deploy-nginx-ipv6.yml -i inventory.ini
ansible-playbook verify-ipv6-services.yml -i inventory.ini
```

Configuring services to listen on IPv6 via Ansible ensures consistent dual-stack deployments across your entire server fleet, with automated verification that each service is correctly bound to both address families.
