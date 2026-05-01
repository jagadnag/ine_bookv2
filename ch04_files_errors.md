# Chapter 4 — Files, Error Handling, and Writing Your First Automation Scripts

> *"It's not enough to be good. You have to be good for something."*  
> — Henry David Thoreau

---

## Chapter Goal

Wire Chapters 2 and 3 together into real automation scripts — scripts that read device inventories from files, collect credentials securely at runtime, handle failures gracefully, and write structured output. By the end of this chapter, you will have working patterns used in production network automation every day.

**Key Points:**
- Reading device inventories from text files and CSV files
- Writing results, reports, and configs to files
- `try/except/else/finally` — production exception handling
- `getpass` and environment variables — keeping secrets out of code
- JSON for storing structured automation data

---

## Working with Files

Production automation does not hardcode device IPs or credentials. It reads them from files — CSV inventories, YAML configs, plain text lists. Separating your data from your logic is the first step toward professional automation.

### The with Statement

The `with` statement manages file opening and closing automatically — even if an error occurs. Always use it:

```python
# ✅ Best practice: with statement
with open('device_list.txt') as f:
    contents = f.read()
# File automatically closed here, even if an exception occurred

# ❌ Old way: manual close required
f = open('device_list.txt')
contents = f.read()
f.close()   # If an exception occurs before here, file is never closed
```

### Reading a Plain Text Device List

The simplest inventory format — one IP per line:

```
# device_list.txt
198.18.1.11
198.18.1.12
198.18.1.13
```

```python
# Read all IPs into a list
with open('device_list.txt') as f:
    device_ips = f.read().splitlines()

# device_ips = ['198.18.1.11', '198.18.1.12', '198.18.1.13']

for ip in device_ips:
    print(f'Will process: {ip}')
```

The `.splitlines()` method splits on newline characters and strips them — you get clean IP strings, no trailing `\n`.

### Reading a CSV Inventory

For richer inventories with multiple fields:

```
# inventory.csv
hostname,ip,device_type,username,password
sw-core-01,10.10.1.1,cisco_ios,admin,cisco123
sw-dist-01,10.10.1.2,cisco_ios,admin,cisco123
sw-access-01,10.10.1.3,cisco_ios,admin,cisco123
```

```python
import csv

with open('inventory.csv') as f:
    reader = csv.DictReader(f)
    devices = [dict(row) for row in reader]

# devices[0] = {
#     'hostname': 'sw-core-01',
#     'ip': '10.10.1.1',
#     'device_type': 'cisco_ios',
#     'username': 'admin',
#     'password': 'cisco123'
# }

print(f'Loaded {len(devices)} devices from inventory')
for device in devices:
    print(f"  {device['hostname']:15} {device['ip']}")
```

`csv.DictReader` automatically uses the first row as column headers and returns each subsequent row as a dictionary — exactly the list-of-dictionaries pattern from Chapter 2.

### Checking if a File Exists

```python
from pathlib import Path

path = Path('inventory.csv')

if path.exists():
    with open(path) as f:
        devices = [dict(row) for row in csv.DictReader(f)]
else:
    print(f'ERROR: {path} not found. Create your inventory file first.')
    devices = []
```

---

## Writing Output to Files

Automation scripts that only print to screen are development tools. Production scripts save their output — reports to CSV, configurations to text files, results to JSON.

### Writing Text Files

```python
from pathlib import Path
from datetime import datetime

# Simple write (overwrites existing file)
path = Path('report.txt')
path.write_text('Automation Report\n')

# Write with a timestamp
timestamp = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
report = f'Automation Report — {timestamp}\n'
report += f'Devices processed: 12\n'
report += f'Errors: 0\n'
path.write_text(report)

# Append to existing file
existing = path.read_text()
existing += '\nDevice sw-core-01: SUCCESS'
path.write_text(existing)
```

### Writing CSV Reports

```python
import csv
from datetime import datetime

results = [
    {'device': 'sw-core-01',   'status': 'success', 'version': '17.9.4a'},
    {'device': 'sw-dist-01',   'status': 'timeout',  'version': None},
    {'device': 'sw-access-01', 'status': 'success', 'version': '17.9.4a'},
]

timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
filename = f'results_{timestamp}.csv'

with open(filename, 'w', newline='') as f:
    writer = csv.DictWriter(f, fieldnames=['device', 'status', 'version'])
    writer.writeheader()
    writer.writerows(results)

print(f'Results saved to {filename}')
```

### Writing JSON

JSON is the best format for structured automation data — human-readable, machine-parseable, and compatible with every other tool:

```python
import json
from pathlib import Path

# Save structured data as JSON
inventory_data = {
    'timestamp': datetime.now().isoformat(),
    'devices': [
        {'hostname': 'sw-core-01', 'ip': '10.10.1.1', 'version': '17.9.4a'},
        {'hostname': 'sw-dist-01', 'ip': '10.10.1.2', 'version': '17.9.4a'},
    ]
}

path = Path('inventory.json')
path.write_text(json.dumps(inventory_data, indent=2))

# Load it back
loaded = json.loads(path.read_text())
print(f"Loaded {len(loaded['devices'])} devices")
print(f"Timestamp: {loaded['timestamp']}")
```

### Creating Directories

```python
import os
from pathlib import Path

# Create backup directory if it doesn't exist
backup_dir = Path('backup/')
backup_dir.mkdir(exist_ok=True)   # exist_ok=True means no error if it exists

# Save a config file into it
hostname = 'sw-core-01'
config_text = '! Running config from sw-core-01\nhostname sw-core-01\n'
config_path = backup_dir / f'{hostname}.cfg'
config_path.write_text(config_text)

print(f'Saved: {config_path}')
```

---

## Exception Handling: Scripts That Survive Reality

The real world is messy. Devices are unreachable. Passwords are wrong. SSH sessions drop mid-transfer. A script without exception handling crashes at the first problem — losing all results from all devices that would have succeeded.

### The try/except Pattern

```python
try:
    # Code that might fail
    result = risky_operation()
except SpecificError:
    # Handle this specific type of failure
    print('Specific thing went wrong')
except AnotherError:
    # Handle another type of failure
    print('Another thing went wrong')
except Exception as e:
    # Catch-all for anything unexpected
    print(f'Unexpected error: {e}')
else:
    # Runs ONLY if no exception occurred
    print('Success!')
finally:
    # Always runs — for cleanup
    print('Cleanup code here')
```

### The Production Netmiko Exception Pattern

This is the complete exception handling structure from `5-netmiko-final.py` — memorize this pattern:

```python
from netmiko import ConnectHandler
from netmiko.exceptions import (
    NetmikoAuthenticationException,
    NetmikoTimeoutException,
)
from paramiko.ssh_exception import SSHException

results  = []
failures = []

for device_ip in device_list:
    print(f'Connecting to {device_ip}...')
    ios_device = {
        'device_type': 'cisco_ios',
        'ip':          device_ip,
        'username':    username,
        'password':    password,
    }

    try:
        net_connect = ConnectHandler(**ios_device)

    except NetmikoAuthenticationException:
        msg = f'Authentication failure: {device_ip}'
        print(f'  ❌ {msg}')
        failures.append({'ip': device_ip, 'error': 'auth_failed'})
        continue   # Skip to next device

    except NetmikoTimeoutException:
        msg = f'Timeout (unreachable): {device_ip}'
        print(f'  ❌ {msg}')
        failures.append({'ip': device_ip, 'error': 'timeout'})
        continue

    except EOFError:
        msg = f'EOF error (connection dropped): {device_ip}'
        print(f'  ❌ {msg}')
        failures.append({'ip': device_ip, 'error': 'eof'})
        continue

    except SSHException:
        msg = f'SSH error (is SSH enabled?): {device_ip}'
        print(f'  ❌ {msg}')
        failures.append({'ip': device_ip, 'error': 'ssh_error'})
        continue

    except Exception as e:
        msg = f'Unexpected error: {str(e)}'
        print(f'  ❌ {msg}')
        failures.append({'ip': device_ip, 'error': str(e)})
        continue

    # Only reaches here if connection succeeded
    output = net_connect.send_config_from_file('config_commands')
    net_connect.save_config()
    net_connect.disconnect()
    print(f'  ✅ Done: {device_ip}')
    results.append({'ip': device_ip, 'status': 'success'})

# Summary
print(f'\n{"="*40}')
print(f'Completed: {len(results)} success, {len(failures)} failed')
if failures:
    print('Failed devices:')
    for f in failures:
        print(f"  {f['ip']}: {f['error']}")
```

> 🔑 **Key Insight:** The four specific exception types (Auth, Timeout, EOF, SSH) each represent a different failure mode with different causes and different remediation steps. Catching them separately lets you give meaningful error messages instead of generic failures. The catch-all `Exception` at the end catches anything unexpected without crashing.

### Common Exception Types

| Exception | Meaning | Common Cause |
|-----------|---------|-------------|
| `NetmikoTimeoutException` | Connection timed out | Device unreachable, firewall, wrong IP |
| `NetmikoAuthenticationException` | Login failed | Wrong credentials, AAA issue |
| `SSHException` | SSH protocol error | SSH not enabled, version mismatch |
| `EOFError` | Connection dropped | VTY limit reached, session timeout |
| `FileNotFoundError` | File not found | Wrong path, file not created |
| `KeyError` | Dict key missing | Parsing error, unexpected output |
| `ValueError` | Wrong value type | `int('not a number')` |

---

## Secure Credentials: Secrets Out of Code

Never hardcode credentials in scripts. Ever. Even if the script lives only on your laptop.

### getpass — Interactive Password Input

```python
from getpass import getpass

# input() shows what the user types — fine for username
username = input('Enter SSH username: ')

# getpass() hides what the user types — required for passwords
password = getpass('Enter SSH password: ')
enable_secret = getpass('Enter enable secret (Enter to skip): ')

print(f'Will connect as: {username}')
# Password never printed
```

### os.getenv — Environment Variables

For automated runs (cron jobs, CI/CD pipelines) where there is no interactive terminal:

```python
import os
from getpass import getpass

# Use environment variable if set; otherwise prompt
username = os.getenv('NETMIKO_USERNAME') or input('Enter username: ')
password = os.getenv('NETMIKO_PASSWORD') or getpass('Enter password: ')

# Set in terminal before running:
# export NETMIKO_USERNAME="admin"
# export NETMIKO_PASSWORD="cisco123"

# Check if variable is set
if not os.getenv('ANTHROPIC_API_KEY'):
    print('WARNING: ANTHROPIC_API_KEY not set')
```

### .env Files with python-dotenv

For development, a `.env` file is the cleanest approach:

```bash
# .env file in project root (add to .gitignore!)
NETMIKO_USERNAME=cisco
NETMIKO_PASSWORD=cisco
ANTHROPIC_API_KEY=sk-ant-...
```

```python
from dotenv import load_dotenv
import os

load_dotenv()   # Reads .env file and sets environment variables

username = os.getenv('NETMIKO_USERNAME')
password = os.getenv('NETMIKO_PASSWORD')
```

> ⚠️ **Critical:** Add `.env` to your `.gitignore` file immediately. Credentials committed to Git — even a private repository — are a security incident waiting to happen. The `.env` file is for your local machine only.

---

## Your First Complete Automation Script

Combining everything from this chapter into a realistic, production-ready script:

```python
#!/usr/bin/env python3
"""
show_command_collector.py

Connects to all devices in device_list.txt and runs all commands
in show_commands.txt. Saves results to JSON.
"""

import os
import json
from datetime import datetime
from pathlib import Path
from getpass import getpass
from netmiko import ConnectHandler
from netmiko.exceptions import NetmikoAuthenticationException, NetmikoTimeoutException
from paramiko.ssh_exception import SSHException

# ─── Configuration ────────────────────────────────────────────
DEVICE_LIST_FILE  = 'device_list.txt'
COMMANDS_FILE     = 'show_commands.txt'
OUTPUT_DIR        = Path('output')

# ─── Load Files ───────────────────────────────────────────────
def load_device_list(filepath):
    """Load device IPs from a text file (one IP per line)."""
    path = Path(filepath)
    if not path.exists():
        print(f'ERROR: {filepath} not found')
        return []
    with open(path) as f:
        return [line.strip() for line in f if line.strip()]

def load_commands(filepath):
    """Load show commands from a text file (one command per line)."""
    path = Path(filepath)
    if not path.exists():
        print(f'ERROR: {filepath} not found')
        return []
    with open(path) as f:
        return [line.strip() for line in f if line.strip()]

# ─── Run Commands on One Device ───────────────────────────────
def run_commands_on_device(device_ip, username, password, commands):
    """Connect to one device and run all commands. Returns result dict."""
    ios_device = {
        'device_type': 'cisco_ios',
        'ip':          device_ip,
        'username':    username,
        'password':    password,
    }

    try:
        with ConnectHandler(**ios_device) as conn:
            hostname = conn.find_prompt().rstrip('#>')
            command_results = {}
            for cmd in commands:
                command_results[cmd] = conn.send_command(cmd)
            return {
                'ip':       device_ip,
                'hostname': hostname,
                'status':   'success',
                'results':  command_results,
            }

    except NetmikoAuthenticationException:
        return {'ip': device_ip, 'status': 'auth_failed', 'results': {}}
    except NetmikoTimeoutException:
        return {'ip': device_ip, 'status': 'timeout', 'results': {}}
    except SSHException:
        return {'ip': device_ip, 'status': 'ssh_error', 'results': {}}
    except Exception as e:
        return {'ip': device_ip, 'status': f'error: {e}', 'results': {}}

# ─── Main ──────────────────────────────────────────────────────
def main():
    print('=' * 55)
    print('  Network Show Command Collector')
    print('=' * 55)

    # Credentials
    username = os.getenv('NETMIKO_USERNAME') or input('Username: ')
    password = os.getenv('NETMIKO_PASSWORD') or getpass('Password: ')

    # Load data
    device_ips = load_device_list(DEVICE_LIST_FILE)
    commands   = load_commands(COMMANDS_FILE)

    if not device_ips or not commands:
        print('Nothing to do. Check your input files.')
        return

    print(f'Devices: {len(device_ips)} | Commands: {len(commands)}')
    print()

    # Run automation
    all_results = []
    for ip in device_ips:
        print(f'Processing {ip}...', end=' ')
        result = run_commands_on_device(ip, username, password, commands)
        all_results.append(result)
        status = '✅' if result['status'] == 'success' else '❌'
        hostname = result.get('hostname', ip)
        print(f'{status} {hostname} ({result["status"]})')

    # Save results
    OUTPUT_DIR.mkdir(exist_ok=True)
    ts = datetime.now().strftime('%Y%m%d_%H%M%S')
    output_file = OUTPUT_DIR / f'results_{ts}.json'
    output_file.write_text(json.dumps(all_results, indent=2))

    # Summary
    success = sum(1 for r in all_results if r['status'] == 'success')
    print(f'\nDone: {success}/{len(device_ips)} succeeded')
    print(f'Results saved: {output_file}')

if __name__ == '__main__':
    main()
```

---

## Practice Exercises

1. Create a `device_list.txt` file with 3 IP addresses (real or fake). Write a script that reads it and prints `"Processing: {ip}"` for each.

2. Write a script that collects a username via `input()` and a password via `getpass()`, then prints `"Would connect as: {username}"` without ever printing the password.

3. Add a `try/except` block to the script from Exercise 1 that catches `FileNotFoundError` and prints a helpful error message instead of crashing.

4. Extend the script from Exercise 1 to save results to a JSON file with this structure:  
   ```json
   [{"ip": "10.1.1.1", "processed": true}, ...]
   ```

---

*Previous: [Chapter 3](ch03_control_flow.md) | Next: [Chapter 5 — Classes, Modules, and Reusable Code](ch05_classes_modules.md)*
