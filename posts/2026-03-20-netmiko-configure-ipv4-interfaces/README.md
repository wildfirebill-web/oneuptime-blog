# How to Configure IPv4 Interfaces on Cisco Devices with Netmiko

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Netmiko, Python, Cisco, IPv4, Network Automation, Interface Configuration

Description: Learn how to use Netmiko to programmatically configure IPv4 addresses on Cisco router and switch interfaces using Python configuration mode commands.

## Configuring Interfaces with Netmiko

Netmiko's `send_config_set()` method sends a list of commands in configuration mode, handling the `configure terminal` and `end` commands automatically.

## Step 1: Basic Interface Configuration

```python
from netmiko import ConnectHandler

device = {
    'device_type': 'cisco_ios',
    'host': '192.168.1.1',
    'username': 'admin',
    'password': 'password',
    'secret': 'enablepass',
}

# Configuration commands as a list

config_commands = [
    'interface GigabitEthernet0/1',
    'ip address 10.0.1.1 255.255.255.0',
    'no shutdown',
    'description WAN_Link',
]

with ConnectHandler(**device) as conn:
    conn.enable()
    output = conn.send_config_set(config_commands)
    print(output)

    # Verify the configuration
    verify = conn.send_command('show ip interface brief')
    print(verify)
```

## Step 2: Configure Multiple Interfaces from a Dictionary

```python
from netmiko import ConnectHandler

device = {
    'device_type': 'cisco_ios',
    'host': '192.168.1.1',
    'username': 'admin',
    'password': 'password',
    'secret': 'enablepass',
}

# Define interface configurations
interfaces = {
    'GigabitEthernet0/0': {
        'ip_address': '192.168.1.1',
        'subnet_mask': '255.255.255.0',
        'description': 'LAN_Interface',
    },
    'GigabitEthernet0/1': {
        'ip_address': '203.0.113.2',
        'subnet_mask': '255.255.255.252',
        'description': 'WAN_ISP1',
    },
    'GigabitEthernet0/2': {
        'ip_address': '198.51.100.2',
        'subnet_mask': '255.255.255.252',
        'description': 'WAN_ISP2',
    },
}

def configure_interfaces(conn, interfaces):
    """Configure multiple interfaces on a Cisco device."""
    all_commands = []
    for intf, config in interfaces.items():
        all_commands.extend([
            f'interface {intf}',
            f'description {config["description"]}',
            f'ip address {config["ip_address"]} {config["subnet_mask"]}',
            'no shutdown',
        ])

    output = conn.send_config_set(all_commands)
    return output

with ConnectHandler(**device) as conn:
    conn.enable()
    result = configure_interfaces(conn, interfaces)
    print(result)

    # Save configuration
    conn.send_command('write memory')
    print("Configuration saved")
```

## Step 3: Configure Interface with Verification

```python
from netmiko import ConnectHandler

def configure_and_verify_interface(host, username, password, enable_pass,
                                   interface, ip_address, prefix_length):
    """Configure an interface and verify the IP assignment."""
    device = {
        'device_type': 'cisco_ios',
        'host': host,
        'username': username,
        'password': password,
        'secret': enable_pass,
    }

    # Convert prefix to mask
    from ipaddress import IPv4Network
    network = IPv4Network(f'0.0.0.0/{prefix_length}', strict=False)
    subnet_mask = str(network.netmask)

    config_commands = [
        f'interface {interface}',
        f'ip address {ip_address} {subnet_mask}',
        'no shutdown',
    ]

    with ConnectHandler(**device) as conn:
        conn.enable()

        # Apply configuration
        output = conn.send_config_set(config_commands)
        print(f"Configuration applied:\n{output}")

        # Verify
        verify = conn.send_command(
            f'show interfaces {interface} | include Internet address'
        )
        print(f"Verification:\n{verify}")

        # Save
        conn.save_config()
        print("Configuration saved to NVRAM")

# Example usage
configure_and_verify_interface(
    host='192.168.1.1',
    username='admin',
    password='password',
    enable_pass='enablepass',
    interface='GigabitEthernet0/1',
    ip_address='10.0.1.1',
    prefix_length=24
)
```

## Step 4: Configure Loopback Interfaces

```python
from netmiko import ConnectHandler

device = {
    'device_type': 'cisco_ios',
    'host': '192.168.1.1',
    'username': 'admin',
    'password': 'password',
    'secret': 'enablepass',
}

# Configure loopback for router ID and management
loopback_commands = [
    'interface Loopback0',
    'description Router_Management',
    'ip address 1.1.1.1 255.255.255.255',    # /32 loopback
    'no shutdown',
]

with ConnectHandler(**device) as conn:
    conn.enable()
    conn.send_config_set(loopback_commands)

    # Verify loopback
    output = conn.send_command('show interfaces loopback 0')
    print(output)

    conn.save_config()
```

## Step 5: Push Configuration to Multiple Routers

```python
from netmiko import ConnectHandler
import csv

# routers.csv format: hostname,ip,username,password,enable,interface,ip_addr,mask
def configure_from_csv(csv_file):
    with open(csv_file) as f:
        reader = csv.DictReader(f)
        for row in reader:
            device = {
                'device_type': 'cisco_ios',
                'host': row['ip'],
                'username': row['username'],
                'password': row['password'],
                'secret': row['enable'],
            }
            commands = [
                f"interface {row['interface']}",
                f"ip address {row['ip_addr']} {row['mask']}",
                'no shutdown',
            ]
            try:
                with ConnectHandler(**device) as conn:
                    conn.enable()
                    conn.send_config_set(commands)
                    conn.save_config()
                    print(f"✓ {row['ip']}: {row['interface']} configured")
            except Exception as e:
                print(f"✗ {row['ip']}: {e}")

configure_from_csv('routers.csv')
```

## Conclusion

Netmiko's `send_config_set()` is the key method for interface configuration - it accepts a list of IOS commands, wraps them in `configure terminal`/`end`, and returns the output. Build your configuration as a list of strings, verify with `send_command('show ip interface brief')`, and persist with `conn.save_config()` or `send_command('write memory')`. This pattern scales to hundreds of devices when combined with a CSV or YAML inventory.
