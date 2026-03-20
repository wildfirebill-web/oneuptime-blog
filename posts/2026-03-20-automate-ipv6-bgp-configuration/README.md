# How to Automate IPv6 BGP Configuration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, BGP, Automation, Python, Ansible, Routing, Jinja2

Description: Automate IPv6 BGP neighbor configuration across multiple routers using Ansible playbooks and Jinja2 templates, enabling consistent MP-BGP IPv6 deployments at scale.

## Introduction

Deploying IPv6 BGP at scale requires consistent configuration of MP-BGP (Multiprotocol BGP) sessions, address families, and route policies. Ansible with Jinja2 templates enables repeatable, auditable BGP configuration that can be deployed across hundreds of routers.

## Step 1: BGP Neighbor Inventory

```yaml
# inventory/bgp_peers.yml
all:
  hosts:
    router1:
      ansible_host: 2001:db8::r1
      bgp_local_as: 65001
      bgp_peers:
        - peer_ip: 2001:db8::peer1
          remote_as: 65002
          description: "Upstream Provider 1"
          ipv6_unicast: true
          soft_reconfiguration: true
        - peer_ip: 2001:db8::peer2
          remote_as: 65003
          description: "Peering Partner"
          ipv6_unicast: true
          route_policy_in: "IPV6_FROM_PARTNER"
          route_policy_out: "IPV6_TO_PARTNER"

    router2:
      ansible_host: 2001:db8::r2
      bgp_local_as: 65001
      bgp_peers:
        - peer_ip: 2001:db8::peer3
          remote_as: 65004
          description: "Data Center Fabric"
          ipv6_unicast: true
```

## Step 2: Ansible Playbook

```yaml
# playbooks/configure_ipv6_bgp.yml
---
- name: Configure IPv6 BGP Neighbors
  hosts: all
  gather_facts: false

  tasks:
    - name: Generate BGP configuration
      template:
        src: "templates/{{ ansible_network_os }}_ipv6_bgp.j2"
        dest: "/tmp/{{ inventory_hostname }}_bgp.txt"
      delegate_to: localhost

    - name: Apply BGP config on IOS-XR
      cisco.iosxr.iosxr_config:
        src: "/tmp/{{ inventory_hostname }}_bgp.txt"
        replace: block
      when: ansible_network_os == "cisco.iosxr.iosxr"

    - name: Wait for BGP sessions to establish
      pause:
        seconds: 60
      when: ansible_network_os == "cisco.iosxr.iosxr"

    - name: Verify BGP sessions
      cisco.iosxr.iosxr_command:
        commands:
          - "show bgp ipv6 unicast summary"
      register: bgp_summary

    - name: Check all configured peers are established
      assert:
        that:
          - item.peer_ip in bgp_summary.stdout[0]
        fail_msg: "BGP peer {{ item.peer_ip }} not found in summary"
      loop: "{{ bgp_peers }}"
```

## Step 3: IOS-XR BGP Jinja2 Template

```jinja2
{# templates/cisco.iosxr.iosxr_ipv6_bgp.j2 #}
router bgp {{ bgp_local_as }}
 address-family ipv6 unicast
  network 2001:db8:{{ inventory_hostname[-1] }}::/48
 !
{% for peer in bgp_peers %}
 neighbor {{ peer.peer_ip }}
  remote-as {{ peer.remote_as }}
  description {{ peer.description }}
  address-family ipv6 unicast
{% if peer.soft_reconfiguration is defined and peer.soft_reconfiguration %}
   soft-reconfiguration inbound always
{% endif %}
{% if peer.route_policy_in is defined %}
   route-policy {{ peer.route_policy_in }} in
{% endif %}
{% if peer.route_policy_out is defined %}
   route-policy {{ peer.route_policy_out }} out
{% endif %}
  !
 !
{% endfor %}
!
```

## Step 4: Python BGP State Checker

```python
# scripts/check_bgp_ipv6.py
from netmiko import ConnectHandler
import yaml

def check_bgp_sessions(host: str, platform: str, expected_peers: list) -> dict:
    """Verify that all expected IPv6 BGP peers are established."""
    device = {
        "device_type": platform,
        "host": host,
        "username": "admin",
        "password": "secret",
    }

    commands = {
        "cisco_iosxr": "show bgp ipv6 unicast summary",
        "juniper_junos": "show bgp summary | match inet6",
    }

    with ConnectHandler(**device) as conn:
        output = conn.send_command(commands.get(platform, "show bgp ipv6 unicast summary"))

    results = {}
    for peer in expected_peers:
        peer_ip = peer["peer_ip"]
        results[peer_ip] = {
            "found": peer_ip in output,
            "established": "Established" in output and peer_ip in output,
        }

    return results

# Run compliance check
with open("inventory/bgp_peers.yml") as f:
    inventory = yaml.safe_load(f)

for hostname, host_data in inventory["all"]["hosts"].items():
    results = check_bgp_sessions(
        host_data["ansible_host"],
        "cisco_iosxr",
        host_data.get("bgp_peers", [])
    )
    for peer_ip, status in results.items():
        status_str = "UP" if status["established"] else "DOWN"
        print(f"{hostname} → {peer_ip}: {status_str}")
```

## Step 5: Deploy and Validate

```bash
# Syntax check
ansible-playbook playbooks/configure_ipv6_bgp.yml --syntax-check

# Dry run
ansible-playbook playbooks/configure_ipv6_bgp.yml --check --diff

# Deploy
ansible-playbook playbooks/configure_ipv6_bgp.yml

# Verify
python scripts/check_bgp_ipv6.py
```

## Conclusion

IPv6 BGP automation with Ansible reduces configuration drift and enables consistent MP-BGP deployments. Store BGP peer definitions in YAML inventory, generate device-specific configurations with Jinja2, and verify session state post-deployment. Monitor BGP peer state changes with OneUptime alerts to detect flapping sessions before they impact traffic.
