# How to Use Python Paramiko for Basic SSH Network Automation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Paramiko, Python, SSH, Network Automation, Cisco

Description: Learn how to use Python Paramiko to connect to network devices via SSH and run commands, as a lower-level alternative to Netmiko for basic automation tasks.

## What Is Paramiko?

Paramiko is a Python library implementing the SSH protocol. Unlike Netmiko (which is built on Paramiko), Paramiko provides lower-level SSH access - useful when:
- Netmiko doesn't support your device type
- You need SSH tunneling or SFTP
- You want fine-grained control over the SSH session

## Step 1: Install Paramiko

```bash
pip install paramiko cryptography

python3 -c "import paramiko; print(paramiko.__version__)"
```

## Step 2: Basic SSH Connection and Command Execution

```python
import paramiko
import time

# Create SSH client

client = paramiko.SSHClient()

# Trust the host key automatically (not for production - verify in prod)
client.set_missing_host_key_policy(paramiko.AutoAddPolicy())

# Connect to device
client.connect(
    hostname='192.168.1.1',
    port=22,
    username='admin',
    password='password',
    timeout=10,
    look_for_keys=False,
    allow_agent=False
)

# Execute a command
stdin, stdout, stderr = client.exec_command('show version')

# Read output
output = stdout.read().decode('utf-8')
error = stderr.read().decode('utf-8')

print(output)
if error:
    print(f"Error: {error}")

client.close()
```

## Step 3: Interactive Shell (for Cisco IOS Enable Mode)

For Cisco IOS, commands run in interactive mode and require entering enable mode:

```python
import paramiko
import time

def cisco_ios_command(hostname, username, password, enable_password, commands):
    """Run commands on Cisco IOS using interactive shell."""
    client = paramiko.SSHClient()
    client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    client.connect(hostname, username=username, password=password,
                   look_for_keys=False, allow_agent=False)

    # Open interactive shell
    shell = client.invoke_shell()
    time.sleep(1)

    # Enter enable mode
    shell.send('enable\n')
    time.sleep(0.5)
    shell.send(enable_password + '\n')
    time.sleep(0.5)

    # Disable paging (prevent --More-- prompts)
    shell.send('terminal length 0\n')
    time.sleep(0.5)

    # Execute each command
    outputs = {}
    for cmd in commands:
        shell.send(cmd + '\n')
        time.sleep(1)    # Wait for output
        output = shell.recv(65535).decode('utf-8', errors='replace')
        outputs[cmd] = output

    client.close()
    return outputs

# Example usage
results = cisco_ios_command(
    hostname='192.168.1.1',
    username='admin',
    password='userpass',
    enable_password='enablepass',
    commands=['show ip interface brief', 'show ip route', 'show version']
)

for cmd, output in results.items():
    print(f"\n=== {cmd} ===\n{output}")
```

## Step 4: Read Files via SFTP

```python
import paramiko

def read_remote_file(hostname, username, password, remote_path):
    """Read a file from a remote Linux server via SFTP."""
    client = paramiko.SSHClient()
    client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    client.connect(hostname, username=username, password=password)

    # Open SFTP session
    sftp = client.open_sftp()

    with sftp.open(remote_path, 'r') as f:
        content = f.read().decode('utf-8')

    sftp.close()
    client.close()
    return content

# Read a configuration file from a Linux router
config = read_remote_file('192.168.1.50', 'admin', 'pass', '/etc/frr/frr.conf')
print(config)
```

## Step 5: SSH Key Authentication

```python
import paramiko

def connect_with_key(hostname, username, key_path):
    """Connect using SSH private key (more secure than password)."""
    client = paramiko.SSHClient()
    client.set_missing_host_key_policy(paramiko.AutoAddPolicy())

    # Load private key
    private_key = paramiko.RSAKey.from_private_key_file(key_path)

    client.connect(
        hostname=hostname,
        username=username,
        pkey=private_key,
        look_for_keys=False,
        allow_agent=False,
    )

    stdin, stdout, stderr = client.exec_command('show ip interface brief')
    output = stdout.read().decode('utf-8')
    client.close()
    return output

output = connect_with_key('192.168.1.1', 'admin', '/home/user/.ssh/network_key')
print(output)
```

## Step 6: Robust Multi-Device Automation

```python
import paramiko
import time
from concurrent.futures import ThreadPoolExecutor

def run_command_on_device(device):
    """Run commands on a device with proper error handling."""
    client = paramiko.SSHClient()
    client.set_missing_host_key_policy(paramiko.AutoAddPolicy())

    try:
        client.connect(
            hostname=device['host'],
            username=device['username'],
            password=device['password'],
            timeout=10,
            look_for_keys=False,
        )

        shell = client.invoke_shell()
        time.sleep(0.5)

        if device.get('enable_password'):
            shell.send(f"enable\n{device['enable_password']}\nterminal length 0\n")
            time.sleep(1)

        results = {}
        for cmd in device.get('commands', []):
            shell.send(cmd + '\n')
            time.sleep(0.5)
            output = shell.recv(32768).decode('utf-8', errors='replace')
            results[cmd] = output

        return {'host': device['host'], 'results': results}

    except paramiko.AuthenticationException:
        return {'host': device['host'], 'error': 'Authentication failed'}
    except Exception as e:
        return {'host': device['host'], 'error': str(e)}
    finally:
        client.close()

devices = [
    {'host': '192.168.1.1', 'username': 'admin', 'password': 'pass',
     'enable_password': 'ep', 'commands': ['show version', 'show ip route']},
    {'host': '192.168.1.2', 'username': 'admin', 'password': 'pass',
     'enable_password': 'ep', 'commands': ['show version', 'show ip route']},
]

with ThreadPoolExecutor(max_workers=5) as executor:
    results = list(executor.map(run_command_on_device, devices))

for result in results:
    if 'error' in result:
        print(f"{result['host']}: ERROR - {result['error']}")
    else:
        print(f"{result['host']}: {len(result['results'])} commands executed")
```

## Conclusion

Paramiko provides raw SSH access to network devices. For Cisco IOS, use `invoke_shell()` with `terminal length 0` to disable paging. Handle enable mode by sending the enable command followed by the enable password. While Paramiko works for any SSH device, consider Netmiko for Cisco-specific automation as it handles paging, enable mode, and configuration mode automatically. Use Paramiko directly when you need SFTP, SSH tunneling, or support for non-standard devices.
