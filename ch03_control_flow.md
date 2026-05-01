# Chapter 3 — Control Flow, Loops, and Functions: Making Python Do the Work

> *"Code is like humor. When you have to explain it, it's bad."*  
> — Cory House

---

## Chapter Goal

Transform your Python vocabulary into automation logic. Data types hold information — control flow decides what to do with it. By the end of this chapter, you will write scripts that make decisions, process every device in a list, and reuse logic through functions. Every concept maps directly to a real network automation scenario.

**Key Points:**
- `if/elif/else` — your script makes decisions about interface states, VLAN validity, and device health
- `for` and `while` loops — the engine that processes every device in your fleet
- `break` and `continue` — controlling loop flow for skip-and-continue error handling
- Functions — write once, call hundreds of times
- List comprehensions — transform data in a single readable line

---

## Conditionals: Teaching Your Script to Think

An `if` statement evaluates an expression and runs code only when it is `True`. This is how your script decides whether an interface is down, whether a VLAN is in the approved list, or whether CPU utilization has crossed a threshold.

### Basic Structure

```python
cpu_util = 87.4

if cpu_util > 90:
    print('CRITICAL: CPU above 90%')
elif cpu_util > 75:
    print('WARNING: High CPU utilization')
elif cpu_util > 50:
    print('NOTICE: CPU is elevated')
else:
    print('OK: CPU within normal range')

# Output: WARNING: High CPU utilization
```

Python uses **indentation** (4 spaces) to delimit code blocks — no curly braces. Every line at the same indent level belongs to the same block. This forces readable structure.

### Comparison Operators

```python
# These all return True or False
status = 'down'

status == 'up'        # False — equal to
status != 'up'        # True  — not equal to
cpu = 87.4
cpu > 80              # True  — greater than
cpu >= 90             # False — greater than or equal
cpu < 90              # True  — less than
cpu <= 87.4           # True  — less than or equal
```

### Logical Operators: and, or, not

```python
interface_up   = True
has_ip         = True
in_maintenance = False

# AND — both conditions must be True
if interface_up and has_ip:
    print('Interface is reachable')

# OR — at least one must be True
if interface_up or has_ip:
    print('Some connectivity exists')

# NOT — reverses True/False
if not in_maintenance:
    print('Device is in production')

# Combined — real-world config push check
if interface_up and has_ip and not in_maintenance:
    print('Safe to push configuration')
```

### Membership Testing: in and not in

One of Python's most network-friendly operators — reads exactly like English:

```python
approved_vlans = [10, 20, 30, 100, 200]
requested_vlan = 50

if requested_vlan in approved_vlans:
    print('VLAN approved — configuring')
else:
    print('VLAN not in approved list — blocked')

# Works on strings too (substring search)
syslog = 'Interface GigabitEthernet0/1, changed state to down'
if 'changed state to down' in syslog:
    print('ALERT: Interface down event detected!')

# not in
banned_devices = ['legacy-sw-01', 'eol-router-01']
device = 'sw-core-01'
if device not in banned_devices:
    print(f'Processing {device}...')
```

### Checking Empty Collections

```python
# Empty lists, dicts, and strings evaluate to False
down_interfaces = []
error_devices   = ['10.1.1.5']

if not down_interfaces:
    print('All interfaces are up!')

if error_devices:
    print(f'WARNING: {len(error_devices)} devices had errors')
```

---

## For Loops: Processing Every Device Without Writing It Twice

A `for` loop executes the same code block for every item in a collection. This is the most important control structure in network automation — write the logic once, Python applies it to every device:

```python
devices = [
    {'hostname': 'sw-core-01',   'ip': '10.10.1.1', 'status': 'up'},
    {'hostname': 'sw-dist-01',   'ip': '10.10.1.2', 'status': 'down'},
    {'hostname': 'sw-access-01', 'ip': '10.10.1.3', 'status': 'up'},
]

for device in devices:
    icon = '✅' if device['status'] == 'up' else '❌'
    print(f"{icon} {device['hostname']:15} {device['ip']:15} {device['status']}")
```

Output:
```
✅ sw-core-01      10.10.1.1       up
❌ sw-dist-01      10.10.1.2       down
✅ sw-access-01    10.10.1.3       up
```

The loop variable (`device`) holds one item from the list on each iteration. Code indented under `for` runs once per item. Code at the same level as `for` runs after all iterations.

### Nested Loops: Every Command on Every Device

```python
devices  = ['198.18.1.11', '198.18.1.12']
commands = ['show version', 'show ip int brief', 'show logging']

for ip in devices:              # outer loop: each device
    print(f'\n=== {ip} ===')
    for cmd in commands:        # inner loop: each command
        print(f'  Running: {cmd}')
        # In real code: output = conn.send_command(cmd)
```

### The range() Function

`range()` generates sequences of numbers without storing them in memory:

```python
# range(stop)          → 0 to stop-1
for i in range(5):
    print(i)            # 0 1 2 3 4

# range(start, stop)   → start to stop-1
for i in range(1, 6):
    print(i)            # 1 2 3 4 5

# range(start, stop, step)
for vlan in range(10, 51, 10):
    print(f'Creating VLAN {vlan}')   # 10 20 30 40 50

# Create a list from range
all_vlans = list(range(1, 4095))
print(len(all_vlans))   # 4094
```

### enumerate(): Loop with Index

```python
switches = ['sw-core-01', 'sw-dist-01', 'sw-access-01']

for i, switch in enumerate(switches, start=1):
    print(f'{i}. {switch}')

# Output:
# 1. sw-core-01
# 2. sw-dist-01
# 3. sw-access-01
```

### zip(): Loop Over Two Lists Together

```python
hostnames = ['sw-core-01', 'sw-dist-01', 'sw-access-01']
ip_addrs  = ['10.10.1.1',  '10.10.1.2',  '10.10.1.3']

for hostname, ip in zip(hostnames, ip_addrs):
    print(f'{hostname}: {ip}')
```

---

## break and continue: Controlling Loop Behavior

Two keywords let you control what happens inside loops.

### break — Exit the Loop Entirely

```python
# Find the first device with a problem and stop
for device in devices:
    if device['status'] == 'critical':
        print(f'CRITICAL device found: {device["hostname"]}')
        break       # Stop checking — escalate immediately
```

### continue — Skip This Item, Keep Going

`continue` is the most important loop control keyword in production network automation. When a device fails to connect, skip it and keep processing the rest:

```python
# Pattern from 5-netmiko-final.py (production lab script)
results  = []
failures = []

for device_ip in device_list:
    print(f'Connecting to {device_ip}...')

    try:
        connection = connect_to_device(device_ip)
    except TimeoutError:
        print(f'  Timeout: {device_ip} — skipping')
        failures.append({'ip': device_ip, 'error': 'timeout'})
        continue    # ← Skip to next device. Do NOT crash.
    except AuthError:
        print(f'  Auth failed: {device_ip} — skipping')
        failures.append({'ip': device_ip, 'error': 'auth_failed'})
        continue    # ← Skip to next device.

    # Only reaches here if connection succeeded
    output = connection.send_command('show version')
    results.append({'ip': device_ip, 'output': output})
    print(f'  ✅ Done')

print(f'\nSuccess: {len(results)}  Failed: {len(failures)}')
```

> ⚠️ **Critical pattern:** Without `continue`, the first device timeout crashes the entire script — you lose all results from all the devices that would have succeeded. With `continue`, failures are logged and the script processes every remaining device.

---

## While Loops: Running Until a Condition Changes

A `while` loop runs as long as its condition is `True`. Use it when you do not know in advance how many iterations you need:

```python
# Retry connection up to 3 times
max_attempts = 3
attempt = 0

while attempt < max_attempts:
    attempt += 1
    print(f'Attempt {attempt}/{max_attempts}...')
    
    if attempt_connection():        # Simulated
        print('Connected!')
        break
    
    print(f'Failed. Waiting...')

else:   # runs only if loop completed without break
    print('Max attempts reached — giving up')
```

### while True with break

```python
# Interactive command loop — keep running until user says 'quit'
while True:
    command = input('Enter command (or quit): ').strip()
    
    if not command:
        continue        # Empty input — ask again
    
    if command.lower() == 'quit':
        print('Exiting.')
        break
    
    print(f'Running: {command}')
    # output = connection.send_command(command)
```

---

## Functions: Write Once, Use Everywhere

A function is a named block of code that you can call as many times as you need, from anywhere in your script. Functions eliminate duplicated logic, make code readable, and enable testing.

### Defining and Calling

```python
def greet_device(hostname):
    """Print a connection banner for a device."""
    print('=' * 50)
    print(f'  Connecting to: {hostname}')
    print('=' * 50)

# Call the function
greet_device('sw-core-01')
greet_device('sw-dist-01')
```

The `def` keyword declares a function. The docstring (triple-quoted string on the first line) describes what it does — always write one. The parameter `hostname` accepts input.

### Return Values

```python
def get_vlan_commands(vlan_id, vlan_name):
    """Return IOS commands to create a VLAN."""
    return [
        f'vlan {vlan_id}',
        f'name {vlan_name}',
    ]

# Use the returned value
cmds = get_vlan_commands(100, 'PRODUCTION')
print(cmds)
# ['vlan 100', 'name PRODUCTION']

# Use in automation
vlans = [
    {'id': 10,  'name': 'MGMT'},
    {'id': 20,  'name': 'USERS'},
    {'id': 100, 'name': 'SERVERS'},
]

for vlan in vlans:
    commands = get_vlan_commands(vlan['id'], vlan['name'])
    print(f"Commands for VLAN {vlan['id']}: {commands}")
```

### Default Parameters

Parameters can have default values, making them optional:

```python
def build_connection_dict(ip, username='cisco', password='cisco',
                          device_type='cisco_ios', port=22):
    """Build a Netmiko connection dictionary."""
    return {
        'device_type': device_type,
        'ip':          ip,
        'username':    username,
        'password':    password,
        'port':        port,
    }

# Use defaults
params = build_connection_dict('10.10.1.1')

# Override specific parameters
params = build_connection_dict('10.10.1.1', port=8022)
params = build_connection_dict('10.10.1.1', username='admin', password='secret')
```

### Keyword Arguments

```python
# Call with keyword arguments — order doesn't matter
params = build_connection_dict(
    password='secret99',
    ip='10.10.1.2',
    username='netadmin'
)
```

### Functions That Return Multiple Values

```python
def analyze_interface(status, protocol):
    """Analyze interface state and return health info."""
    if status == 'up' and protocol == 'up':
        return 'HEALTHY', 'green', True
    elif status == 'up' and protocol == 'down':
        return 'DEGRADED', 'yellow', False
    else:
        return 'DOWN', 'red', False

health, color, is_up = analyze_interface('up', 'down')
print(f'State: {health}, Color: {color}, Reachable: {is_up}')
# State: DEGRADED, Color: yellow, Reachable: False
```

### *args and **kwargs

For functions that accept a variable number of arguments:

```python
def push_commands(hostname, *commands):
    """Push any number of commands to a device."""
    print(f'Pushing {len(commands)} commands to {hostname}')
    for cmd in commands:
        print(f'  → {cmd}')

push_commands('sw-core-01', 'vlan 10', 'name MGMT')
push_commands('sw-dist-01', 'vlan 10', 'name MGMT', 'vlan 20', 'name USERS')
```

```python
def build_device(**kwargs):
    """Build device dict from any keyword arguments."""
    return kwargs

device = build_device(hostname='sw-01', ip='10.0.0.1', role='core')
# {'hostname': 'sw-01', 'ip': '10.0.0.1', 'role': 'core'}
```

---

## List Comprehensions: Elegant Data Transformation

A list comprehension builds a new list from an existing one in a single readable line. The pattern: `[expression for item in collection if condition]`

```python
# Old way — loop
up_interfaces = []
for intf in all_interfaces:
    if intf['status'] == 'up':
        up_interfaces.append(intf)

# New way — comprehension
up_interfaces = [i for i in all_interfaces if i['status'] == 'up']

# Build IP list for a /24 subnet
host_ips = [f'10.10.1.{n}' for n in range(1, 255)]
print(host_ips[:3])   # ['10.10.1.1', '10.10.1.2', '10.10.1.3']

# Uppercase all hostnames
hostnames = ['sw-core-01', 'sw-dist-01', 'sw-access-01']
upper = [h.upper() for h in hostnames]
# ['SW-CORE-01', 'SW-DIST-01', 'SW-ACCESS-01']

# Filter: only Gigabit interfaces
interfaces = ['Gi0/0', 'Fa0/1', 'Gi0/1', 'Lo0']
gig_only = [i for i in interfaces if i.startswith('Gi')]
# ['Gi0/0', 'Gi0/1']
```

---

## Dictionary Comprehensions

Same idea, for dictionaries:

```python
# Map hostnames to IPs
hostnames = ['sw-core-01', 'sw-dist-01', 'sw-access-01']
ips       = ['10.10.1.1',  '10.10.1.2',  '10.10.1.3']

device_map = {h: ip for h, ip in zip(hostnames, ips)}
# {'sw-core-01': '10.10.1.1', ...}

# Squares
squares = {x: x**2 for x in range(5)}
# {0: 0, 1: 1, 2: 4, 3: 9, 4: 16}
```

---

## Putting It Together: A Complete Show Command Parser

Here is everything from this chapter combined into a realistic function:

```python
def parse_interface_brief(show_output):
    """
    Parse 'show ip interface brief' output.
    Returns: (up_interfaces, down_interfaces)
    """
    up   = []
    down = []

    for line in show_output.splitlines():
        # Skip header lines and empty lines
        if not line.strip() or 'Interface' in line:
            continue

        parts = line.split()
        if len(parts) < 6:
            continue

        interface = parts[0]
        ip_addr   = parts[1]
        status    = parts[4]
        protocol  = parts[5]

        record = {
            'interface': interface,
            'ip':        ip_addr,
            'status':    status,
            'protocol':  protocol,
        }

        if status == 'up' and protocol == 'up':
            up.append(record)
        else:
            down.append(record)

    return up, down


# Example usage
sample_output = """
Interface              IP-Address      OK? Method Status    Protocol
GigabitEthernet0/0     10.10.1.1       YES manual up        up
GigabitEthernet0/1     unassigned      YES unset  down      down
Loopback0              10.255.255.1    YES manual up        up
Vlan10                 10.10.10.1      YES manual up        up
Vlan20                 unassigned      YES unset  down      down
"""

up_ints, down_ints = parse_interface_brief(sample_output)

print(f'UP interfaces ({len(up_ints)}):')
for intf in up_ints:
    print(f"  ✅ {intf['interface']:25} {intf['ip']}")

print(f'\nDOWN interfaces ({len(down_ints)}):')
for intf in down_ints:
    print(f"  ❌ {intf['interface']}")
```

Output:
```
UP interfaces (3):
  ✅ GigabitEthernet0/0       10.10.1.1
  ✅ Loopback0                 10.255.255.1
  ✅ Vlan10                    10.10.10.1

DOWN interfaces (2):
  ❌ GigabitEthernet0/1
  ❌ Vlan20
```

---

## Practice Exercises

1. Write a function `check_vlan(vlan_id)` that returns `'reserved'` if the VLAN is in `(1, 1002, 1003, 1004, 1005)`, `'invalid'` if outside 1–4094, and `'valid'` otherwise. Test it with VLANs 0, 1, 10, 1002, 4094, 4095.

2. Write a loop that iterates through a list of CPU percentages `[23, 55, 72, 88, 95, 12]` and prints a severity label (`NORMAL`, `WARNING`, `HIGH`, `CRITICAL`) for each. Use a function to determine the severity.

3. Use a list comprehension to build a list of all VLAN IDs that are multiples of 10 between 10 and 100.

4. Write a `while` loop that simulates retrying a device connection up to 5 times, with a different error message for each failure. Use `break` when simulated success occurs.

---

*Previous: [Chapter 2](ch02_data_types.md) | Next: [Chapter 4 — Files, Error Handling, and First Scripts](ch04_files_errors.md)*
