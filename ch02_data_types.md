# Chapter 2 — Python Data Types: The Building Blocks of Network Automation

> *"Data is a precious thing and will last longer than the systems themselves."*  
> — Tim Berners-Lee

---

## Chapter Goal

Master Python's core data types through a networking lens. Every data type introduced here maps directly to something you already work with every day — hostnames are strings, VLAN IDs are integers, interface states are booleans, device inventories are lists, and connection parameters are dictionaries.

**Key Points:**
- Strings and their methods — the language your devices speak
- Numbers, booleans, and the arithmetic you already know
- Lists — your device inventory in code
- Dictionaries — the most important data structure in network automation
- Nested structures — lists of dictionaries, the real-world inventory pattern

---

## The Python Interactive Interpreter: Your Lab Bench

Every example in this chapter should be run in the Python interactive interpreter. Think of it as your lab environment — the same way you would verify a routing protocol before assuming it works, verify Python behavior before trusting it in a script.

```bash
$ python3
Python 3.8.12 ...
>>>
```

The `>>>` prompt means Python is waiting. The convention throughout this book:
- Lines starting with `>>>` are Python interpreter input
- Lines without `>>>` are the output Python returns
- Lines starting with `$` are terminal (bash) commands

---

## Strings: The Language of Network Devices

Every piece of text your devices produce — show command output, log messages, configuration text, hostname, IP addresses — is a string. A string is any sequence of characters enclosed in quotes (single or double, both work):

```python
>>> hostname = 'ROUTER_1'
>>> ipaddr = "192.168.1.1"
>>> banner = '\n  WELCOME TO ROUTER_1  \n'
```

### Verifying Types

Python's `type()` function tells you what you are working with — invaluable when debugging:

```python
>>> type(hostname)
<class 'str'>
>>> type(42)
<class 'int'>
>>> type(3.14)
<class 'float'>
```

> 💡 **The `dir()` and `help()` trio:** Three built-in functions accelerate your learning:
> - `type(obj)` — what type is this object?
> - `dir(obj)` — what methods does it have?
> - `help(obj.method)` — how do I use this method?
>
> ```python
> >>> dir(str)  # Shows all string methods
> >>> help(str.strip)  # Explains strip() in detail
> ```

### String Concatenation and F-Strings

The old way to combine strings was concatenation with `+`. The modern way is f-strings — use them:

```python
>>> hostname = 'sw-core-01'
>>> ip = '10.10.1.1'
>>> version = '17.9.4a'

# Old (avoid)
>>> 'Device ' + hostname + ' at ' + ip
'Device sw-core-01 at 10.10.1.1'

# Modern: f-strings (preferred)
>>> f'Device {hostname} at {ip} running IOS-XE {version}'
'Device sw-core-01 at 10.10.1.1 running IOS-XE 17.9.4a'

# F-strings can do math and formatting inline
>>> ports_used = 31
>>> ports_total = 48
>>> f'Port utilization: {ports_used/ports_total*100:.1f}%'
'Port utilization: 64.6%'
```

The `:.1f` inside the braces is a format specifier — it says "format this float to 1 decimal place." You will use this constantly for percentages and measurements.

### Essential String Methods

String methods are functions attached to string objects. Call them with dot notation: `string.method()`.

**Case conversion — essential for comparisons:**

```python
>>> interface = 'GigabitEthernet0/1'
>>> interface.lower()
'gigabitethernet0/1'
>>> interface.upper()
'GIGABITETHERNET0/1'

# Safe case-insensitive comparison
>>> user_input = 'GigabitEthernet0/1'
>>> user_input.lower() == 'gigabitethernet0/1'
True
```

**Strip — remove whitespace:**

```python
# Show command output often has leading/trailing whitespace
>>> raw_line = '  10.1.1.1      YES manual up      up  '
>>> raw_line.strip()
'10.1.1.1      YES manual up      up'
>>> raw_line.lstrip()   # Left side only
'10.1.1.1      YES manual up      up  '
>>> raw_line.rstrip()   # Right side only
'  10.1.1.1      YES manual up      up'
```

**Split — turn a string into a list:**

```python
>>> raw_line = '10.1.1.1      YES manual up      up'
>>> raw_line.split()  # Split on any whitespace
['10.1.1.1', 'YES', 'manual', 'up', 'up']

# Split on a specific character
>>> ipaddr = '10.1.20.30'
>>> ipaddr.split('.')
['10', '1', '20', '30']

# Split a show output line and extract fields
>>> parts = raw_line.split()
>>> interface_ip = parts[0]
>>> status = parts[3]
>>> protocol = parts[4]
```

**Join — combine a list into a string:**

```python
>>> commands = ['config t', 'interface Ethernet1/1', 'shutdown']
>>> '\n'.join(commands)      # Newline between commands
'config t\ninterface Ethernet1/1\nshutdown'
>>> ' ; '.join(commands)     # Semicolon (for NX-API)
'config t ; interface Ethernet1/1 ; shutdown'
```

**Membership testing:**

```python
>>> line = 'Interface GigabitEthernet0/1, changed state to down'
>>> 'changed state to down' in line
True
>>> line.startswith('Interface')
True
>>> line.endswith('down')
True
```

> 📝 **Note:** String methods return **new strings** — they do not modify the original. If you want to keep the result, assign it: `lower_intf = interface.lower()`

### Real-World: Parsing show ip interface brief

This combination of strip + split + in is the foundation of every show-command parser you will ever write:

```python
# Simulated line from 'show ip interface brief'
>>> raw = '  GigabitEthernet0/0          10.10.1.1   YES manual up       up  '
>>> parts = raw.strip().split()
>>> parts
['GigabitEthernet0/0', '10.10.1.1', 'YES', 'manual', 'up', 'up']

>>> interface = parts[0]
>>> ip        = parts[1]
>>> status    = parts[4]
>>> protocol  = parts[5]

>>> if status == 'up' and protocol == 'up':
...     print(f'✅ {interface}: {ip} — HEALTHY')
... else:
...     print(f'❌ {interface}: {ip} — PROBLEM')
✅ GigabitEthernet0/0: 10.10.1.1 — HEALTHY
```

---

## Numbers: Counting What Matters

Network automation uses numbers for port counts, VLAN IDs, prefix lengths, utilization percentages, and timing. Python handles two main number types:

- **int** — whole numbers: `48`, `100`, `9300`
- **float** — decimals: `52.3`, `99.9`

```python
>>> port_count = 48
>>> cpu_util = 52.3
>>> type(port_count)
<class 'int'>
>>> type(cpu_util)
<class 'float'>
```

### Arithmetic Operators

```python
>>> 5 + 3      # Addition
8
>>> 10 - 4     # Subtraction
6
>>> 3 * 4      # Multiplication
12
>>> 10 / 3     # Division (always returns float)
3.3333333333333335
>>> 10 // 3    # Integer (floor) division
3
>>> 10 % 3     # Modulo (remainder)
1
>>> 2 ** 8     # Exponentiation
256
```

### Networking Applications

```python
# Subnet host calculation
>>> prefix = 24
>>> 2 ** (32 - prefix) - 2
254

# Usable hosts for common prefixes
>>> for prefix in [24, 25, 26, 27, 28, 29, 30]:
...     hosts = 2 ** (32 - prefix) - 2
...     print(f'/{prefix}: {hosts} usable hosts')
/24: 254 usable hosts
/25: 126 usable hosts
/26: 62 usable hosts
/27: 30 usable hosts
/28: 14 usable hosts
/29: 6 usable hosts
/30: 2 usable hosts

# String repetition (useful for formatting)
>>> '=' * 50
'=================================================='
>>> print('=' * 50)
==================================================

# Increment pattern
>>> counter = 0
>>> counter += 1    # Same as counter = counter + 1
>>> counter
1
```

### Type Conversion

```python
>>> str(10)        # int → string
'10'
>>> int('10')      # string → int
10
>>> float('52.3')  # string → float
52.3

# Critical: input() always returns a string!
>>> age = input('How old are you? ')  # User types 25
>>> type(age)
<class 'str'>
>>> age = int(age)  # Must convert before arithmetic
>>> type(age)
<class 'int'>
```

---

## Booleans: The Yes/No of Your Network

Boolean values are `True` or `False`. They are the engine of every decision in your scripts:

```python
>>> is_layer3 = True
>>> has_ospf = False
>>> in_maintenance = False

# Boolean operations
>>> is_layer3 and has_ospf
False
>>> is_layer3 or has_ospf
True
>>> not is_layer3
False

# Comparison operators return booleans
>>> status = 'down'
>>> status == 'up'
False
>>> status != 'up'
True
>>> cpu = 87.4
>>> cpu > 80
True
```

### Empty Object Evaluation

One of Python's most elegant features: empty objects evaluate to `False`:

```python
>>> down_interfaces = []
>>> if not down_interfaces:
...     print('All interfaces are up!')
All interfaces are up!

>>> error_devices = ['10.1.1.5']
>>> if error_devices:
...     print(f'Warning: {len(error_devices)} devices had errors')
Warning: 1 devices had errors
```

This lets you write conditions that read like English: "if there are no down interfaces..." rather than "if the length of down_interfaces equals zero..."

---

## Lists: Your Device Inventory in Code

A list is an ordered collection of items enclosed in square brackets. If you have a mental model for a spreadsheet column, you understand lists:

```python
>>> hostnames = ['r1', 'r2', 'r3', 'r4', 'r5']
>>> interfaces = ['Eth1/1', 'Eth1/2', 'Eth1/3', 'Eth1/4']
>>> vlans = [10, 20, 30, 100, 200]
```

### Accessing Items

```python
# Index starts at 0
>>> hostnames[0]
'r1'
>>> hostnames[1]
'r2'
>>> hostnames[-1]    # Last item
'r5'
>>> hostnames[-2]    # Second to last
'r4'

# Slicing: list[start:end] (end is NOT included)
>>> hostnames[1:4]   # Index 1, 2, 3
['r2', 'r3', 'r4']
>>> hostnames[:3]    # First three
['r1', 'r2', 'r3']
>>> hostnames[-3:]   # Last three
['r3', 'r4', 'r5']
```

> ⚠️ **Warning:** Python indexing starts at 0. The first item is `[0]`, not `[1]`. Print the index and value side by side when debugging: `for i, item in enumerate(mylist): print(i, item)`

### Modifying Lists

```python
>>> switches = ['sw-core-01', 'sw-dist-01']

# Add to end
>>> switches.append('sw-access-01')
>>> switches
['sw-core-01', 'sw-dist-01', 'sw-access-01']

# Insert at position
>>> switches.insert(0, 'sw-access-00')
>>> switches
['sw-access-00', 'sw-core-01', 'sw-dist-01', 'sw-access-01']

# Remove by value
>>> switches.remove('sw-access-00')

# Sort
>>> switches.sort()

# Count items
>>> len(switches)
3

# Check membership
>>> 'sw-core-01' in switches
True
>>> 'sw-access-99' in switches
False
```

### Looping Through a List

The `for` loop processes every item in sequence:

```python
>>> devices = ['10.1.1.1', '10.1.1.2', '10.1.1.3']
>>> for ip in devices:
...     print(f'Connecting to {ip}...')
Connecting to 10.1.1.1...
Connecting to 10.1.1.2...
Connecting to 10.1.1.3...
```

This single pattern — a list of devices and a for loop — is the foundation of every multi-device automation script you will write.

### Copying a List

```python
# ❌ WRONG — both variables point to the SAME list
>>> original = ['sw-01', 'sw-02']
>>> copy = original
>>> copy.append('sw-03')
>>> original    # Original was also modified!
['sw-01', 'sw-02', 'sw-03']

# ✅ CORRECT — independent copy using [:]
>>> original = ['sw-01', 'sw-02']
>>> copy = original[:]
>>> copy.append('sw-03')
>>> original    # Unchanged
['sw-01', 'sw-02']
```

---

## Dictionaries: The Heart of Network Automation

If lists are spreadsheet columns, dictionaries are spreadsheet rows — they store related attributes together, accessible by name:

```python
>>> device = {
...     'hostname': 'router1',
...     'vendor':   'cisco',
...     'os':       'ios-xe',
...     'version':  '17.9.4a',
...     'ip':       '10.10.1.1',
... }
```

### Accessing Values

```python
>>> device['hostname']
'router1'
>>> device['version']
'17.9.4a'

# CRITICAL: Use get() for safe access
>>> device['model']       # KeyError if missing — crashes!
KeyError: 'model'

>>> device.get('model')           # Returns None if missing — safe
>>> device.get('model', 'UNKNOWN') # Returns default if missing
'UNKNOWN'
```

> 💡 **Always use `.get()` for dictionary access when the key might not exist.** Networks are inconsistent — parsed device data often has missing fields. Crashing because one device in fifty is missing a `model` key is unacceptable in production automation.

### Modifying Dictionaries

```python
>>> device = {'hostname': 'router1', 'vendor': 'cisco'}

# Add a new key
>>> device['version'] = '17.9.4a'

# Update existing key
>>> device['vendor'] = 'cisco-ios-xe'

# Remove a key
>>> del device['vendor']

# Merge two dictionaries
>>> oper_data = {'cpu': '5%', 'memory': '10%'}
>>> device.update(oper_data)
>>> device
{'hostname': 'router1', 'version': '17.9.4a', 'cpu': '5%', 'memory': '10%'}
```

### Iterating Over Dictionaries

Three methods, three use cases:

```python
>>> facts = {'hostname': 'r1', 'vendor': 'cisco', 'os': 'ios-xe'}

# Keys only
>>> for key in facts.keys():
...     print(key)
hostname
vendor
os

# Values only
>>> for val in facts.values():
...     print(val)
r1
cisco
ios-xe

# Key-value pairs simultaneously — most common!
>>> for key, val in facts.items():
...     print(f'{key}: {val}')
hostname: r1
vendor: cisco
os: ios-xe
```

The `.items()` pattern appears in nearly every automation script — when building config commands from a template dictionary, when printing device summaries, when comparing current state to desired state.

---

## The Most Important Data Structure: List of Dictionaries

Real-world device inventories are not flat lists of strings. They are **lists of dictionaries** — one dictionary per device, each containing that device's attributes:

```python
>>> inventory = [
...     {'hostname': 'sw-core-01', 'ip': '10.10.1.1', 'role': 'core'},
...     {'hostname': 'sw-dist-01', 'ip': '10.10.1.2', 'role': 'distribution'},
...     {'hostname': 'sw-access-01', 'ip': '10.10.1.3', 'role': 'access'},
... ]

# Access nested data
>>> inventory[0]['hostname']
'sw-core-01'
>>> inventory[0]['ip']
'10.10.1.1'

# Loop and print summary
>>> for device in inventory:
...     print(f"{device['hostname']:15} {device['ip']:15} {device['role']}")
sw-core-01      10.10.1.1       core
sw-dist-01      10.10.1.2       distribution
sw-access-01    10.10.1.3       access

# Filter: get only core devices
>>> core_devices = [d for d in inventory if d['role'] == 'core']
>>> len(core_devices)
1
```

> 🔑 **Key Insight:** The list-of-dictionaries pattern is the single most important data structure in network automation. Your device inventory, your parsed Genie output, your configuration records — all use this exact pattern. Master it here, recognize it everywhere.

---

## Data Types Reference

| Type | Notation | Example | Network Use |
|------|----------|---------|------------|
| `str` | `'text'` | `'sw-core-01'` | Hostnames, IPs, commands, show output |
| `int` | `42` | `48` | Port counts, VLANs, prefix lengths |
| `float` | `3.14` | `52.3` | CPU %, bandwidth utilization |
| `bool` | `True/False` | `True` | Interface state, reachability |
| `list` | `[a, b, c]` | `['sw-01', 'sw-02']` | Device inventories, command lists |
| `dict` | `{k: v}` | `{'ip': '10.0.0.1'}` | Device facts, connection params |
| `tuple` | `(a, b)` | `(1, 1002, 1003)` | Immutable: reserved VLANs, fixed values |
| `set` | `set([...])` | `set(vendors)` | Unique values, deduplication |

---

## Practice Exercises

1. Create a dictionary for one network device with at least 5 keys. Access each value using `.get()` with a default of `'UNKNOWN'`.

2. Build a list of 5 switch hostnames. Print the first two, the last two, and the length. Sort them alphabetically.

3. Take this raw show output line and extract the interface name, IP address, status, and protocol using `.strip().split()`:
   ```
   '  GigabitEthernet0/2          192.168.1.1     YES manual up       up  '
   ```

4. Create a list of 3 device dictionaries (different IPs and roles). Loop through and print only the devices with role `'access'`.

5. Open the Python interpreter and run: `2 ** (32 - n) - 2` for n = 24, 26, 28, 30. What are the usable host counts?

---

*Previous: [Chapter 1](ch01_why_python.md) | Next: [Chapter 3 — Control Flow, Loops, and Functions](ch03_control_flow.md)*
