# How to Retrieve and Parse IPv4 Routing Tables with Netmiko and TextFSM

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Netmiko, TextFSM, Python, Routing Table, Network Automation, Cisco

Description: Learn how to retrieve and parse IPv4 routing tables from Cisco devices using Netmiko with TextFSM templates, converting raw CLI output into structured Python data.

## Why Parse Routing Tables with TextFSM?

Raw `show ip route` output is human-readable but difficult to process programmatically. TextFSM parses structured text using templates, converting CLI output into Python dictionaries that you can filter, compare, and analyze.

## Step 1: Install Dependencies

```bash
pip install netmiko ntc-templates

# ntc-templates provides pre-built TextFSM templates for many Cisco commands
python3 -c "from netmiko import ConnectHandler; print('OK')"
```

## Step 2: Get Routing Table as Structured Data

Netmiko with TextFSM (`use_textfsm=True`) automatically parses output:

```python
from netmiko import ConnectHandler

device = {
    'device_type': 'cisco_ios',
    'host': '192.168.1.1',
    'username': 'admin',
    'password': 'password',
    'secret': 'enablepass',
}

with ConnectHandler(**device) as conn:
    conn.enable()

    # use_textfsm=True parses output into list of dicts
    routes = conn.send_command('show ip route', use_textfsm=True)

    if isinstance(routes, list):
        print(f"Found {len(routes)} routes")
        for route in routes[:5]:    # Show first 5
            print(route)
    else:
        print("TextFSM parsing failed, raw output:")
        print(routes)
```

## Step 3: Filter Routes by Type or Network

```python
from netmiko import ConnectHandler
from ipaddress import ip_network, ip_address

device = {
    'device_type': 'cisco_ios',
    'host': '192.168.1.1',
    'username': 'admin',
    'password': 'password',
    'secret': 'enablepass',
}

with ConnectHandler(**device) as conn:
    conn.enable()
    routes = conn.send_command('show ip route', use_textfsm=True)

if not isinstance(routes, list):
    print("Parse failed")
    exit(1)

# Filter by protocol type
bgp_routes = [r for r in routes if r.get('protocol') == 'B']
ospf_routes = [r for r in routes if r.get('protocol') == 'O']
static_routes = [r for r in routes if r.get('protocol') == 'S']
connected_routes = [r for r in routes if r.get('protocol') == 'C']

print(f"BGP routes: {len(bgp_routes)}")
print(f"OSPF routes: {len(ospf_routes)}")
print(f"Static routes: {len(static_routes)}")
print(f"Connected routes: {len(connected_routes)}")

# Find routes to a specific supernet
target_network = ip_network('10.0.0.0/8')
matching = [
    r for r in routes
    if r.get('network') and ip_network(
        f"{r['network']}/{r.get('mask', '32')}", strict=False
    ).subnet_of(target_network)
]
print(f"\nRoutes within 10.0.0.0/8: {len(matching)}")
for route in matching:
    print(f"  {route['network']}/{route.get('mask', '')} via {route.get('nexthop_ip', 'directly connected')}")
```

## Step 4: Compare Routing Tables Between Devices

```python
from netmiko import ConnectHandler
from concurrent.futures import ThreadPoolExecutor

devices = [
    {'device_type': 'cisco_ios', 'host': '192.168.1.1', 'username': 'admin', 'password': 'pass', 'secret': 'ep'},
    {'device_type': 'cisco_ios', 'host': '192.168.1.2', 'username': 'admin', 'password': 'pass', 'secret': 'ep'},
]

def get_routes(device):
    with ConnectHandler(**device) as conn:
        conn.enable()
        routes = conn.send_command('show ip route', use_textfsm=True)
    return device['host'], {r['network'] for r in routes if isinstance(routes, list) and r.get('network')}

# Get routes from all devices in parallel
with ThreadPoolExecutor(max_workers=len(devices)) as executor:
    results = dict(executor.map(get_routes, devices))

# Compare
hosts = list(results.keys())
if len(hosts) == 2:
    only_in_first = results[hosts[0]] - results[hosts[1]]
    only_in_second = results[hosts[1]] - results[hosts[0]]

    print(f"Routes only in {hosts[0]}: {only_in_first}")
    print(f"Routes only in {hosts[1]}: {only_in_second}")
    print(f"Routes in both: {len(results[hosts[0]] & results[hosts[1]])}")
```

## Step 5: Generate a Route Report

```python
from netmiko import ConnectHandler
import json
from datetime import datetime

def generate_route_report(devices, output_file):
    report = {
        'timestamp': datetime.now().isoformat(),
        'devices': {}
    }

    for device in devices:
        host = device['host']
        try:
            with ConnectHandler(**device) as conn:
                conn.enable()
                routes = conn.send_command('show ip route', use_textfsm=True)
                hostname = conn.send_command('show hostname')

            report['devices'][host] = {
                'hostname': hostname.strip(),
                'route_count': len(routes) if isinstance(routes, list) else 0,
                'routes': routes if isinstance(routes, list) else [],
            }
            print(f"✓ {host}: {report['devices'][host]['route_count']} routes")

        except Exception as e:
            report['devices'][host] = {'error': str(e)}
            print(f"✗ {host}: {e}")

    with open(output_file, 'w') as f:
        json.dump(report, f, indent=2)

    print(f"\nReport saved to {output_file}")

generate_route_report(devices, '/tmp/route-report.json')
```

## Conclusion

Netmiko with `use_textfsm=True` converts raw Cisco CLI output into Python lists of dictionaries, making routing table analysis simple. Use `send_command('show ip route', use_textfsm=True)` to get structured route data, then filter by `protocol`, `network`, or `nexthop_ip` fields. Compare routing tables between devices to detect inconsistencies, and generate JSON reports for documentation. The ntc-templates library provides pre-built TextFSM templates for hundreds of Cisco commands.
