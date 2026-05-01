# Chapter 7 — Multi-Device Automation and File-Driven Inventory

> *"The goal of automation is not to reduce jobs, but to reduce the time spent on work that machines do better than humans."*

---

## Chapter Goal

Scale from one device to your entire fleet. This chapter covers the transition from hardcoded single-device scripts to data-driven, file-based multi-device automation — the exact progression of lab scripts 3 and 4.

**Key Points:**
- Lists of dictionaries — your fleet as a Python data structure (lab script 3)
- Reading device inventories from files — the `device_list` and `config_commands` files (lab script 4)
- `send_config_from_file()` — pushing configuration from a template file
- Collecting credentials at runtime with `getpass`

---

## Lab Script 3 — `3-netmiko-config.py`: Multiple Devices

### The Full Script

```python
#!/usr/bin/env python
from netmiko import ConnectHandler

# SSH Connection Details
ios1 = {
    'device_type': 'cisco_ios',
    'ip': '198.18.1.11',
    'username': 'cisco',
    'password': 'cisco',
}

ios2 = {
    'device_type': 'cisco_ios',
    'ip': '198.18.1.12',
    'username': 'cisco',
    'password': 'cisco',
}

devices = [ios1, ios2]

for device in devices:
    print('Connecting to device ' + device['ip'])
    net_connect = ConnectHandler(**device)
    output = net_connect.send_config_set('logging host 10.1.1.2')
    net_connect.disconnect()
    print(output)
```

### Walkthrough: The List of Dictionaries Pattern

**`devices = [ios1, ios2]`**  
This creates a list of dictionaries — each element is one complete connection dictionary. This is the same pattern from Chapter 2: a list where each item is a dictionary. The list holds your fleet.

**`for device in devices:`**  
On the first iteration, `device` holds `ios1`. On the second, it holds `ios2`. The code inside the loop runs identically for each — the same six config commands go to every switch.

**`ConnectHandler(**device)`**  
Each iteration creates a fresh connection. The `**device` unpacking works the same whether the variable is named `ios1` or `device` — it passes all keys as keyword arguments.

**Python Connection to Chapter 2:**  
`device['ip']` accesses the `'ip'` key from the current device dictionary. This is direct dictionary access from Chapter 2.

### Expected Output

```
Connecting to device 198.18.1.11
configure terminal
Enter configuration commands, one per line.  End with CNTL/Z.
CSR1(config)#logging host 10.1.1.2
CSR1(config)#end

Connecting to device 198.18.1.12
configure terminal
...
CSR2(config)#logging host 10.1.1.2
CSR2(config)#end
```

---

## Lab Script 4 — `4-netmiko-config.py`: File-Driven Inventory

### The Full Script

```python
#!/usr/bin/env python
from netmiko import ConnectHandler
from getpass import getpass

# SSH username and password provided by user
username = input('Enter your SSH username: ')
password = getpass('Enter your password: ')

# Sending device ip's stored in a file
with open('device_list') as f:
    device_list = f.read().splitlines()

# Iterate through device list and configure the devices
for device in device_list:
    print('Connecting to device ' + device)
    ip_address_of_device = device

    # SSH Connection details
    ios_device = {
        'device_type': 'cisco_ios',
        'ip': ip_address_of_device,
        'username': username,
        'password': password
    }

    net_connect = ConnectHandler(**ios_device)
    output = net_connect.send_config_from_file('config_commands')
    net_connect.disconnect()
    print(output)
```

### Walkthrough: All New Concepts

**`from getpass import getpass`**  
Import the `getpass` function for secure password input (no echo).

**`username = input('Enter your SSH username: ')`**  
`input()` prompts the user and returns what they type. Credentials collected at runtime — not hardcoded.

**`password = getpass('Enter your password: ')`**  
Same as `input()` but characters are not echoed. The user sees only the prompt, not what they type.

**`with open('device_list') as f: device_list = f.read().splitlines()`**  
Opens `device_list` file, reads it, splits on newlines. Result: `['198.18.1.11', '198.18.1.12']`

**`for device in device_list:`**  
Loop through each IP string. On first iteration, `device = '198.18.1.11'`.

**`ios_device = {'device_type': 'cisco_ios', 'ip': ip_address_of_device, ...}`**  
Build the connection dictionary inside the loop using the current IP. This is the key pattern: one dictionary built per device per iteration, using runtime data.

**`net_connect.send_config_from_file('config_commands')`**  
Reads commands from the `config_commands` file and sends them all. This separates configuration data from automation logic.

### The `device_list` File

```
198.18.1.11
198.18.1.12
```

One IP per line. No headers, no formatting. Clean and simple.

### The `config_commands` File

```
logging console
logging host 10.1.1.3
ntp server 10.1.1.4
ip name-server 10.1.1.5
no ip http server
no ip http secure-server
snmp-server community cisco_public RO
snmp-server community cisco_private RW
snmp-server location dCloud
ip access-list extended TEST_ACL
 permit ip 1.1.1.0 0.0.0.255 any
 permit ip 2.2.2.0 0.0.0.255 any
 permit ip 3.3.3.0 0.0.0.255 any
interface loopback 10
 description Created by Python
router ospf 10
 network 0.0.0.0 0.0.0.0 area 0
```

This file is pushed verbatim to every device. Change the file content, re-run the script, and all devices get the updated configuration.

> 🔑 **Key Insight:** Script 4 establishes the architecture of professional network automation: **data files** (device_list, config_commands) contain the specifics; **the script** contains the logic. To target different devices, change the data file. To push different config, change the config file. The script never changes.

### Python Concepts Used

| Concept | Where | Chapter |
|---------|-------|---------|
| `input()` | Get username | Ch 1 |
| `getpass()` | Get password securely | Ch 4 |
| `with open() as f:` | File reading | Ch 4 |
| `.splitlines()` | Parse file lines | Ch 2 |
| `for device in device_list:` | Loop through IPs | Ch 3 |
| Dict inside loop | Build per-device params | Ch 2 |
| `send_config_from_file()` | Push from file | Netmiko |

---

## Building a Scalable Multi-Device Runner

Here is the pattern from script 4 refactored into a reusable function structure:

```python
#!/usr/bin/env python3
"""
multi_device_config.py
Refactored version of 4-netmiko-config.py with function organization.
"""

import os
from getpass import getpass
from netmiko import ConnectHandler


def load_device_list(filepath='device_list'):
    """Load device IPs from a file."""
    with open(filepath) as f:
        return f.read().splitlines()


def build_device_params(ip, username, password):
    """Build a Netmiko connection dictionary."""
    return {
        'device_type': 'cisco_ios',
        'ip': ip,
        'username': username,
        'password': password,
    }


def configure_device(device_params, config_file='config_commands'):
    """Connect to one device and push config from file."""
    net_connect = ConnectHandler(**device_params)
    output = net_connect.send_config_from_file(config_file)
    net_connect.save_config()
    net_connect.disconnect()
    return output


def main():
    username = os.getenv('NETMIKO_USERNAME') or input('Username: ')
    password = os.getenv('NETMIKO_PASSWORD') or getpass('Password: ')

    device_ips = load_device_list()
    print(f'Processing {len(device_ips)} devices...\n')

    for ip in device_ips:
        print(f'Connecting to {ip}...', end=' ')
        params = build_device_params(ip, username, password)
        output = configure_device(params)
        print('✅ Done')
        print(output)


if __name__ == '__main__':
    main()
```

---

## Practice Exercises

1. Create a `device_list` file with 2 or 3 IP addresses. Run script 4 against your lab devices (or simulate it by printing instead of connecting).

2. Create a `config_commands` file with 3 configuration commands. Verify they were applied with `show running-config`.

3. Refactor script 3 to use a list comprehension to build the device list — start from a list of IP strings and build the list of dicts inline.

---

*Previous: [Chapter 6](ch06_netmiko_intro.md) | Next: [Chapter 8 — Production Scripts](ch08_production_scripts.md)*

---
