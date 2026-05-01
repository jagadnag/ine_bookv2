# Chapter 9 — Automated Backup and Inventory Reports

## Chapter Goal

Build two of the highest-value automation tools in network operations: automated configuration backup and structured device inventory reporting. Covers lab scripts 8, 9, and 10.

---

## Lab Script 8 — `8-netmiko_backup.py`: Configuration Backup

### The Full Script

```python
#!/usr/bin/env python
import os, time
from datetime import datetime
from getpass import getpass
from netmiko import ConnectHandler

print(datetime.now())

# Credentials
password = os.getenv("NETMIKO_PASSWORD") if os.getenv("NETMIKO_PASSWORD") else getpass()
username = os.getenv("NETMIKO_USERNAME") if os.getenv("NETMIKO_USERNAME") else input('Enter username: ')

# Create backup folder
if not os.path.exists('backup/'):
    os.mkdir('backup')

# Read device list
with open('device_list') as f:
    device_list = f.read().splitlines()

# Iterate and backup each device
for devices in device_list:
    print('Connecting to device ' + devices)
    ios_device = {
        'device_type': 'cisco_ios',
        'ip': devices,
        'username': username,
        'password': password
    }

    net_connect = ConnectHandler(**ios_device)

    # Get hostname from device prompt
    hostname = net_connect.find_prompt()[:-1]

    # Collect running config
    print(f'Initiating config backup on {hostname}..')
    output = net_connect.send_command("show run")

    # Save to file named after hostname
    with open(f'backup/{hostname}.cfg', 'w') as file:
        file.write(output)

    print('Finished config backup \n')
    net_connect.disconnect()
    time.sleep(3)

print(datetime.now())
```

### Key Patterns

**`print(datetime.now())`** (first and last lines)  
Timestamps the run. When you look at the output later, you know exactly when it ran and how long it took.

**`if not os.path.exists('backup/'): os.mkdir('backup')`**  
Create the backup directory only if it does not exist. Without the `if`, `os.mkdir()` raises `FileExistsError` on the second run.

**`hostname = net_connect.find_prompt()[:-1]`**  
`find_prompt()` returns the current prompt, e.g., `CSR1#`. The `[:-1]` slice removes the last character (the `#`), leaving `CSR1`. This is how the script names each backup file after the device — no manual hostname mapping needed.

**`with open(f'backup/{hostname}.cfg', 'w') as file:`**  
Creates `backup/CSR1.cfg` for device 1, `backup/CSR2.cfg` for device 2, etc.

**`time.sleep(3)`**  
A 3-second pause between devices. Prevents overwhelming devices with rapid back-to-back connections and gives SSH sessions time to fully close.

### Output Files

```
backup/
  CSR1.cfg     ← full running config of device 1
  CSR2.cfg     ← full running config of device 2
```

---

## Lab Script 9 — `9-netmiko-report.py`: Genie Parsing Introduction

### The Full Script

```python
#!/usr/bin/env python
from netmiko import ConnectHandler
from pprint import pprint

ios1 = {
    'device_type': 'cisco_ios',
    'ip': '198.18.1.11',
    'username': 'cisco',
    'password': 'cisco',
}

# Connect and use Genie to parse 'show version'
net_connect = ConnectHandler(**ios1)
output = net_connect.send_command('show version', use_genie=True)
net_connect.disconnect()

print(output)
print()
pprint(output)
print()
```

### What `use_genie=True` Does

Without Genie:
```python
output = net_connect.send_command('show version')
type(output)   # <class 'str'>
# "Cisco IOS XE Software, Version 16.11.01\n..."
```

With Genie:
```python
output = net_connect.send_command('show version', use_genie=True)
type(output)   # <class 'dict'>
# {
#   'version': {
#     'hostname': 'CSR1',
#     'chassis': 'CSR1000V',
#     'chassis_sn': '9KIBQAQ3OPE',
#     'os': 'IOS-XE',
#     'version': '16.11.01',
#     ...
#   }
# }
```

**`pprint`** ("pretty print") formats nested dictionaries readably. Regular `print()` on a large dict produces one unreadable line.

---

## Lab Script 10 — `10-netmiko-report.py`: Multi-Device Inventory Report

### The Full Script

```python
#!/usr/bin/env python
import os
import csv
from getpass import getpass
from netmiko import ConnectHandler
from pprint import pprint
from tabulate import tabulate

password = os.getenv("NETMIKO_PASSWORD") if os.getenv("NETMIKO_PASSWORD") else getpass()
username = os.getenv("NETMIKO_USERNAME") if os.getenv("NETMIKO_USERNAME") else input('Enter username: ')

# Initialize inventory with header row
inventory = []
title = ["Hostname", "Chassis", "Serial No", "os", "version"]
inventory.append(title)

# Read device list
with open('device_list') as f:
    device_list = f.read().splitlines()

# Collect data from each device
for devices in device_list:
    print('Connecting to device ' + devices)
    ios_device = {
        'device_type': 'cisco_ios',
        'ip': devices,
        'username': username,
        'password': password
    }

    net_connect = ConnectHandler(**ios_device)
    output = net_connect.send_command('show version', use_genie=True)
    net_connect.disconnect()

    # Extract structured fields from Genie output
    hostname = output["version"]["hostname"]
    chassis  = output["version"]["chassis"]
    serial   = output["version"]["chassis_sn"]
    os_type  = output["version"]["os"]
    version  = output["version"]["version"]

    device_details = [hostname, chassis, serial, os_type, version]
    inventory.append(device_details)

# Print formatted table
print(tabulate(inventory, headers="firstrow", tablefmt="grid"))

# Save to CSV
with open("inventory.csv", "w", newline="") as f:
    writer = csv.writer(f)
    writer.writerows(inventory)
```

### The Inventory Build Pattern

```python
# Start with a list containing the header row
inventory = [["Hostname", "Chassis", "Serial No", "os", "version"]]

# For each device, extract fields and append a row
device_details = [hostname, chassis, serial, os_type, version]
inventory.append(device_details)

# Result after 2 devices:
# [
#   ["Hostname", "Chassis", ...],     <- header
#   ["CSR1", "CSR1000V", ...],        <- device 1
#   ["CSR2", "CSR1000V", ...],        <- device 2
# ]
```

### Expected Table Output

```
+----------+----------+--------------+--------+---------+
| Hostname | Chassis  | Serial No    | os     | version |
+==========+==========+==============+========+=========+
| CSR1     | CSR1000V | 9KIBQAQ3OPE  | IOS-XE | 16.11.01|
+----------+----------+--------------+--------+---------+
| CSR2     | CSR1000V | 9KIBQAQ3OPF  | IOS-XE | 16.11.01|
+----------+----------+--------------+--------+---------+
```

**`tabulate(inventory, headers="firstrow", tablefmt="grid")`:**
- `headers="firstrow"` — use the first list as column headers
- `tablefmt="grid"` — draw the border grid

> 🔑 **Business value:** This script is immediately demonstrable. "Press Enter, it SSHes to every device and gives you a spreadsheet of what is running where." From zero to actionable inventory in under 30 lines.

---

*Previous: [Chapter 8](ch08_production_scripts.md) | Next: [Chapter 10 — Nornir: Parallel Automation at Scale](ch10_nornir_parallel.md)*

---
