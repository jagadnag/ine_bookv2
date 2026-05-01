# Chapter 5 — Classes, Modules, and Building Reusable Automation Code

> *"Good code is its own best documentation."*  
> — Steve McConnell

---

## Chapter Goal

Organize your Python knowledge into professional, reusable code using classes and modules. By the end of this chapter, you will understand how Netmiko's `ConnectHandler` works from the inside, how to build your own `NetworkDevice` class, and how to structure a growing automation platform into maintainable modules.

**Key Points:**
- What classes are and why they make automation code cleaner
- Building a `NetworkDevice` class with attributes and methods
- Inheritance — `CiscoSwitch` built on `NetworkDevice`
- Modules — splitting code across files for a professional project structure
- Understanding `ConnectHandler` from the inside out

---

## What is a Class?

A class is a blueprint for creating objects. An object created from a class is called an **instance**. Classes bundle together related data (**attributes**) and behavior (**methods**) for a concept you want to model.

For network automation, the most natural class is a `NetworkDevice` — an object that knows its own connection parameters and can perform its own operations.

You have already been using classes without realizing it. When you run:

```python
net_connect = ConnectHandler(**ios1)
```

You are creating an **instance** of Netmiko's `ConnectHandler` class. When you call:

```python
net_connect.send_command('show version')
```

You are calling a **method** on that instance. Understanding classes from the inside makes you far better at using libraries from the outside.

---

## Creating Your First Class

```python
class NetworkDevice:
    """Represents a network device with connection capabilities."""

    def __init__(self, hostname, ip, device_type='cisco_ios'):
        """Initialize device attributes.
        
        __init__ runs automatically when you create an instance.
        self refers to the specific instance being created.
        """
        self.hostname    = hostname     # Instance attribute
        self.ip          = ip
        self.device_type = device_type
        self.connected   = False        # Default: not connected
        self.facts       = {}           # Empty dict for collected data

    def get_connection_params(self, username, password):
        """Return a Netmiko-compatible connection dictionary."""
        return {
            'device_type': self.device_type,
            'ip':          self.ip,
            'username':    username,
            'password':    password,
        }

    def mark_connected(self):
        """Mark this device as connected."""
        self.connected = True
        print(f'✅ Connected to {self.hostname} ({self.ip})')

    def show_summary(self):
        """Print a one-line device summary."""
        status = 'ONLINE' if self.connected else 'OFFLINE'
        print(f'{self.hostname:15} | {self.ip:15} | {status}')

    def __repr__(self):
        """String representation for debugging."""
        return f'NetworkDevice({self.hostname}, {self.ip})'
```

### Creating Instances

```python
# Create instances from the class
core_sw = NetworkDevice('sw-core-01', '10.10.1.1')
dist_sw = NetworkDevice('sw-dist-01', '10.10.1.2')

# Access attributes
print(core_sw.hostname)    # sw-core-01
print(core_sw.ip)          # 10.10.1.1
print(core_sw.connected)   # False

# Call methods
core_sw.mark_connected()   # ✅ Connected to sw-core-01 (10.10.1.1)
core_sw.show_summary()     # sw-core-01      | 10.10.1.1       | ONLINE
dist_sw.show_summary()     # sw-dist-01      | 10.10.1.2       | OFFLINE

# Each instance is independent
print(core_sw.connected)   # True
print(dist_sw.connected)   # False

# Use in automation
params = core_sw.get_connection_params('admin', 'cisco123')
# {'device_type': 'cisco_ios', 'ip': '10.10.1.1', ...}
```

### Understanding `self`

`self` is the instance the method was called on. When you call `core_sw.show_summary()`, Python automatically passes `core_sw` as `self`. That is why you can access `self.hostname` inside the method — it is the same as `core_sw.hostname` from outside.

```python
# These are equivalent:
core_sw.show_summary()
NetworkDevice.show_summary(core_sw)
```

---

## Class Attributes vs Instance Attributes

```python
class NetworkDevice:
    # Class attribute — shared by ALL instances
    vendor_default = 'cisco'
    connection_count = 0

    def __init__(self, hostname, ip):
        # Instance attributes — unique to EACH instance
        self.hostname = hostname
        self.ip = ip

sw1 = NetworkDevice('sw-01', '10.0.0.1')
sw2 = NetworkDevice('sw-02', '10.0.0.2')

# Class attribute is the same for both
print(sw1.vendor_default)   # cisco
print(sw2.vendor_default)   # cisco

# Instance attributes differ
print(sw1.hostname)   # sw-01
print(sw2.hostname)   # sw-02
```

---

## Inheritance: Extending What Exists

Inheritance lets you create a specialized version of an existing class. A `CiscoSwitch` is a `NetworkDevice` with extra Layer-2 capabilities. You get everything the parent has, plus whatever you add:

```python
class CiscoSwitch(NetworkDevice):
    """A Cisco switch with Layer-2 specific capabilities."""

    def __init__(self, hostname, ip, num_ports=48):
        """Initialize CiscoSwitch — call parent __init__ first."""
        super().__init__(hostname, ip)   # Initialize NetworkDevice part
        self.num_ports = num_ports       # New attribute
        self.vlans = []                  # New attribute

    def add_vlan(self, vlan_id, vlan_name=''):
        """Add a VLAN to this switch's tracking list."""
        vlan = {
            'id':   vlan_id,
            'name': vlan_name or f'VLAN{vlan_id}',
        }
        self.vlans.append(vlan)
        print(f'Added VLAN {vlan_id} ({vlan["name"]}) to {self.hostname}')

    def get_vlan_commands(self):
        """Return IOS commands to provision all tracked VLANs."""
        commands = []
        for vlan in self.vlans:
            commands.append(f"vlan {vlan['id']}")
            commands.append(f"name {vlan['name']}")
        return commands

    def show_vlans(self):
        """Display all tracked VLANs."""
        print(f'\nVLANs on {self.hostname}:')
        for vlan in self.vlans:
            print(f"  VLAN {vlan['id']:4}: {vlan['name']}")


# Use the subclass
sw = CiscoSwitch('sw-core-01', '10.10.1.1', num_ports=48)

# Inherited methods work
sw.mark_connected()         # From NetworkDevice
sw.show_summary()           # From NetworkDevice

# New methods from CiscoSwitch
sw.add_vlan(10, 'MGMT')
sw.add_vlan(20, 'USERS')
sw.add_vlan(100, 'SERVERS')
sw.show_vlans()

cmds = sw.get_vlan_commands()
print('\nCommands to push:')
for cmd in cmds:
    print(f'  {cmd}')
```

Output:
```
✅ Connected to sw-core-01 (10.10.1.1)
sw-core-01      | 10.10.1.1       | ONLINE
Added VLAN 10 (MGMT) to sw-core-01
Added VLAN 20 (USERS) to sw-core-01
Added VLAN 100 (SERVERS) to sw-core-01

VLANs on sw-core-01:
  VLAN   10: MGMT
  VLAN   20: USERS
  VLAN  100: SERVERS

Commands to push:
  vlan 10
  name MGMT
  vlan 20
  ...
```

### `super().__init__()`

`super().__init__(hostname, ip)` calls the parent class's `__init__` method. This initializes all the parent's attributes (`self.hostname`, `self.ip`, `self.connected`, `self.facts`) before adding the subclass-specific ones. Always call `super().__init__()` at the top of a subclass `__init__`.

---

## Understanding ConnectHandler from the Inside

Now you understand classes, you can understand exactly what Netmiko does:

```python
# Simplified version of how ConnectHandler works internally
class ConnectHandler:
    """Manages SSH connections to network devices."""

    def __init__(self, device_type, ip, username, password, **kwargs):
        """Establish SSH connection on instantiation."""
        self.device_type = device_type
        self.ip = ip
        self.username = username
        # ... (SSH handshake happens here)
        self._connection = self._establish_ssh()

    def __enter__(self):
        """Support 'with' statement — return self."""
        return self

    def __exit__(self, *args):
        """Disconnect when 'with' block exits."""
        self.disconnect()

    def send_command(self, command, **kwargs):
        """Send a show command and return output."""
        # Send over SSH, wait for prompt, return output
        return self._connection.send(command)

    def send_config_set(self, commands):
        """Enter config mode, send commands, exit."""
        self._connection.send('configure terminal')
        for cmd in commands:
            self._connection.send(cmd)
        self._connection.send('end')

    def find_prompt(self):
        """Return the device's current prompt string."""
        return self._connection.get_prompt()

    def save_config(self):
        """Save running config to startup config."""
        return self.send_command('write memory')

    def disconnect(self):
        """Close the SSH connection."""
        self._connection.close()
```

When you write `with ConnectHandler(**ios1) as conn:`, Python:
1. Calls `ConnectHandler.__init__()` — establishes SSH
2. Calls `ConnectHandler.__enter__()` — returns the object as `conn`
3. Runs your code block
4. Calls `ConnectHandler.__exit__()` — disconnects automatically

This is why the `with` statement is so important — it guarantees cleanup even if an exception occurs.

---

## Modules: Organizing Your Automation Platform

As your automation code grows, a single file becomes unmanageable. Modules let you split code into logical, importable Python files.

### Recommended Project Structure

```
network_automation/
├── inventory.py        # Device loading and management
├── connection.py       # Connection helpers, exception handling  
├── commands.py         # Show command runners and parsers
├── config_gen.py       # Configuration template generation
├── reporting.py        # Output formatting and file writing
├── models.py           # NetworkDevice and subclasses
└── main.py             # Orchestration (imports everything else)
```

### Creating and Importing Modules

```python
# File: models.py
class NetworkDevice:
    """Base class for all network devices."""
    def __init__(self, hostname, ip):
        self.hostname = hostname
        self.ip = ip

class CiscoSwitch(NetworkDevice):
    """Cisco switch with L2 capabilities."""
    pass
```

```python
# File: inventory.py
import csv
from pathlib import Path

def load_devices(filepath='inventory.csv'):
    """Load device inventory from CSV file."""
    path = Path(filepath)
    if not path.exists():
        return []
    with open(path) as f:
        return [dict(row) for row in csv.DictReader(f)]

def load_device_ips(filepath='device_list.txt'):
    """Load plain list of device IPs."""
    path = Path(filepath)
    if not path.exists():
        return []
    with open(path) as f:
        return [line.strip() for line in f if line.strip()]
```

```python
# File: connection.py
from netmiko import ConnectHandler
from netmiko.exceptions import NetmikoAuthenticationException, NetmikoTimeoutException
from paramiko.ssh_exception import SSHException

def safe_connect(device_params):
    """Connect to a device with full exception handling. Returns conn or None."""
    try:
        conn = ConnectHandler(**device_params)
        return conn
    except NetmikoAuthenticationException:
        print(f"  Auth failed: {device_params['ip']}")
        return None
    except NetmikoTimeoutException:
        print(f"  Timeout: {device_params['ip']}")
        return None
    except Exception as e:
        print(f"  Error: {e}")
        return None
```

```python
# File: main.py
from inventory import load_devices
from connection import safe_connect

def main():
    devices = load_devices('inventory.csv')
    
    for device in devices:
        conn = safe_connect(device)
        if not conn:
            continue
        
        output = conn.send_command('show version')
        conn.disconnect()
        print(f"  ✅ {device['hostname']}: collected {len(output)} chars")

if __name__ == '__main__':
    main()
```

### The `if __name__ == '__main__'` Pattern

This is one of Python's most important idioms:

```python
# When you RUN the file directly:
#   __name__ == '__main__'  → main() is called
#
# When you IMPORT the file as a module:
#   __name__ == 'main'      → main() is NOT called automatically

if __name__ == '__main__':
    main()
```

Without this pattern, importing your file would immediately run the automation — undesirable when you just want to use its functions in another script.

---

## Building a NetworkDevice Inventory Class

A more complete, production-ready model:

```python
# models.py

import json
import csv
from pathlib import Path
from datetime import datetime


class NetworkDevice:
    """Represents a managed network device."""

    def __init__(self, hostname, ip, device_type='cisco_ios',
                 username='', password='', role='unknown'):
        self.hostname    = hostname
        self.ip          = ip
        self.device_type = device_type
        self.username    = username
        self.password    = password
        self.role        = role
        self.connected   = False
        self.last_seen   = None
        self.facts       = {}

    @classmethod
    def from_dict(cls, data):
        """Create instance from a dictionary (e.g., CSV row)."""
        return cls(
            hostname    = data.get('hostname', data.get('ip', 'unknown')),
            ip          = data['ip'],
            device_type = data.get('device_type', 'cisco_ios'),
            username    = data.get('username', ''),
            password    = data.get('password', ''),
            role        = data.get('role', 'unknown'),
        )

    def to_dict(self):
        """Convert to dictionary for serialization."""
        return {
            'hostname':    self.hostname,
            'ip':          self.ip,
            'device_type': self.device_type,
            'role':        self.role,
            'connected':   self.connected,
            'last_seen':   self.last_seen,
            'facts':       self.facts,
        }

    def get_netmiko_params(self):
        """Return Netmiko connection dictionary."""
        return {
            'device_type': self.device_type,
            'ip':          self.ip,
            'username':    self.username,
            'password':    self.password,
        }

    def __repr__(self):
        return f'NetworkDevice({self.hostname!r}, {self.ip!r})'

    def __str__(self):
        status = 'ONLINE' if self.connected else 'OFFLINE'
        return f'{self.hostname} ({self.ip}) [{status}]'


class DeviceInventory:
    """Manages a collection of NetworkDevice instances."""

    def __init__(self):
        self.devices = []

    def load_from_csv(self, filepath):
        """Load devices from a CSV inventory file."""
        with open(filepath) as f:
            for row in csv.DictReader(f):
                self.devices.append(NetworkDevice.from_dict(row))
        return self

    def get_by_role(self, role):
        """Return all devices with a specific role."""
        return [d for d in self.devices if d.role == role]

    def get_by_ip(self, ip):
        """Find a device by IP address."""
        return next((d for d in self.devices if d.ip == ip), None)

    def summary(self):
        """Print a formatted inventory summary."""
        print(f'{"Hostname":15} {"IP":15} {"Role":15} {"Status"}')
        print('-' * 60)
        for d in self.devices:
            status = '✅ ONLINE' if d.connected else '❌ OFFLINE'
            print(f'{d.hostname:15} {d.ip:15} {d.role:15} {status}')

    def __len__(self):
        return len(self.devices)

    def __iter__(self):
        return iter(self.devices)


# Usage
if __name__ == '__main__':
    inv = DeviceInventory()
    inv.load_from_csv('inventory.csv')

    print(f'Loaded {len(inv)} devices\n')
    inv.summary()

    core_devices = inv.get_by_role('core')
    print(f'\nCore devices: {len(core_devices)}')
```

---

## Section 1 Complete — What You Can Now Do

You have covered every Python concept needed for professional network automation:

| Concept | Chapter | Network Application |
|---------|---------|-------------------|
| Variables, strings, f-strings | Ch 2 | Device names, IPs, log messages |
| Lists and dictionaries | Ch 2 | Inventories, connection params |
| Control flow (if/elif/else) | Ch 3 | Interface checks, VLAN validation |
| Loops (for, while) | Ch 3 | Multi-device automation |
| break and continue | Ch 3 | Skip failed devices |
| Functions | Ch 3 | Reusable automation logic |
| List comprehensions | Ch 3 | Filter, transform device data |
| File I/O | Ch 4 | Read inventories, save reports |
| Exception handling | Ch 4 | Production-grade error recovery |
| Secure credentials | Ch 4 | getpass, environment variables |
| Classes and objects | Ch 5 | NetworkDevice model |
| Inheritance | Ch 5 | CiscoSwitch, AreoRouter |
| Modules | Ch 5 | Organized project structure |

In Section 2, we apply all of this to real Cisco devices with Netmiko, Genie, and Nornir.

---

## Practice Exercises

1. Build a `NetworkDevice` class with at least 5 attributes. Add a `get_netmiko_params()` method that returns a connection dictionary.

2. Create a `CiscoRouter` subclass that adds routing-specific attributes (`routing_protocols`, `bgp_asn`) and a `get_bgp_summary()` method.

3. Build a simple `DeviceInventory` class with `add_device()`, `get_by_role()`, and `summary()` methods. Test it with 3 devices of different roles.

4. Create a `connection.py` module with a `safe_connect(device_params)` function that includes full exception handling. Import and use it from `main.py`.

---

*Previous: [Chapter 4](ch04_files_errors.md) | Next: [Chapter 6 — Netmiko: SSHing to Your Network at Python Speed](../section2/ch06_netmiko_intro.md)*
