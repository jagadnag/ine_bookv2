# Chapter 6 — Netmiko: SSHing to Your Network at Python Speed

> *"Don't repeat yourself. And when you must, let the computer do the repeating."*  
> — Andy Hunt

---

## Chapter Goal

Apply every concept from Section 1 to real network devices. Netmiko bridges your Python knowledge to your network — this chapter walks through the first two lab scripts in full detail, explaining every line and how it connects to what you already know.

**Key Points:**
- What Netmiko is and how to install it
- The device dictionary — the most important Netmiko pattern
- `send_command()` vs `send_config_set()` — two different purposes
- The `with` statement for automatic disconnection
- Your first single-device show command and config script

---

## What is Netmiko?

Netmiko is an open-source Python library that makes SSH connections to network devices simple. Without it, raw SSH handling in Python requires hundreds of lines of Paramiko code — dealing with channel setup, prompt detection, banner handling, and timing. Netmiko wraps all of that complexity in a clean interface.

One line connects you. One line sends a command. One line gets the output.

```bash
pip install netmiko
```

Verify the installation:

```python
import netmiko
print(netmiko.__version__)   # e.g., 4.3.0
```

---

## Lab Environment

The examples in this chapter and the next four use the **jagadnag/labato_1010** lab environment (Cisco dCloud or compatible CML lab):

```
Device IPs:   198.18.1.11,  198.18.1.12
Username:     cisco
Password:     cisco
Device type:  Cisco IOS / IOS-XE (CSR1000V)
```

You can substitute any Cisco IOS or IOS-XE device. Update the IPs in `device_list` to match your environment.

---

## Supported Device Types Reference

| Platform | `device_type` | Notes |
|----------|--------------|-------|
| Cisco IOS / IOS-XE | `cisco_ios` | Catalyst 9K, CSR, ISR, ASR |
| Cisco NX-OS | `cisco_nxos` | Nexus 5K/7K/9K |
| Cisco IOS-XR | `cisco_iosxr` | ASR 9000, NCS |
| Cisco ASA | `cisco_asa` | Firewall |
| Arista EOS | `arista_eos` | All Arista |
| Juniper JunOS | `juniper_junos` | MX, QFX, SRX |
| Palo Alto | `paloalto_panos` | PA-Series |
| Fortinet | `fortinet` | FortiGate |

> 📝 **Note:** Cisco IOS-XE uses `'cisco_ios'` — not `'cisco_iosxe'`. The Catalyst 9000 series uses `'cisco_ios'`.

---

## Lab Script 1 — `1-netmiko-show.py`

The simplest possible Netmiko script. Connect to one device, run one show command, print the output.

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

# Establish SSH to device and run show command
net_connect = ConnectHandler(**ios1)
output = net_connect.send_command('show version')
net_connect.disconnect()
print(output)
```

### Line-by-Line Walkthrough

**Line 1: `#!/usr/bin/env python`**  
The shebang line. On Linux/Mac, it tells the OS which interpreter to use when you run the script directly (`./1-netmiko-show.py`). Harmless on Windows. Good practice to include.

**Line 2: `from netmiko import ConnectHandler`**  
Import the `ConnectHandler` class from the netmiko package. Remember Chapter 5 — `ConnectHandler` is a class, and we are about to create an **instance** of it.

**Lines 4–8: The device dictionary**
```python
ios1 = {
    'device_type': 'cisco_ios',
    'ip': '198.18.1.11',
    'username': 'cisco',
    'password': 'cisco',
}
```
This is a plain Python **dictionary** (Chapter 2). Five key-value pairs define everything Netmiko needs to make the SSH connection. The variable name `ios1` is arbitrary — you could name it `device` or `router1` or anything else.

**Line 10: `net_connect = ConnectHandler(**ios1)`**  
The `**ios1` syntax **unpacks** the dictionary, passing each key as a keyword argument. This is equivalent to:
```python
net_connect = ConnectHandler(
    device_type='cisco_ios',
    ip='198.18.1.11',
    username='cisco',
    password='cisco'
)
```
Under the hood, Netmiko:
1. Opens a TCP connection to port 22
2. Performs the SSH handshake
3. Logs in with username and password
4. Waits for the device prompt (`#` or `>`)
5. Returns the connection object as `net_connect`

**Line 11: `output = net_connect.send_command('show version')`**  
Sends `show version` to the device and waits for the output. Netmiko detects when the command output ends (prompt returns) and captures everything in between. Returns the output as a **string**.

**Line 12: `net_connect.disconnect()`**  
Closes the SSH connection cleanly. Releases the VTY line on the device.

**Line 13: `print(output)`**  
Print the captured output string to the terminal.

### Expected Output (abbreviated)

```
Cisco IOS XE Software, Version 16.11.01
Cisco IOS Software [Gibraltar], Virtual XE Software (X86_64_LINUX_IOSD-UNIVERSALK9-M),
Version 16.11.01, RELEASE SOFTWARE (fc5)
...
cisco CSR1000V (VXE) processor (revision VXE) with 2392579K/3075K bytes of memory.
...
1 Gigabit Ethernet interface
32768K bytes of non-volatile configuration memory.
...
```

### Python Concepts Used

| Concept | Where Used | Chapter |
|---------|-----------|---------|
| Dictionary | `ios1 = {...}` | Ch 2 |
| `**` unpacking | `ConnectHandler(**ios1)` | Ch 3/5 |
| Variable assignment | `output = ...` | Ch 2 |
| Method call | `.send_command(...)` | Ch 5 |
| `print()` | `print(output)` | Ch 1 |

---

## Best Practice: The `with` Statement

Script 1 works, but it has a problem: if `send_command()` raises an exception, `disconnect()` is never called — the SSH session leaks. The `with` statement (Chapter 4) solves this:

```python
#!/usr/bin/env python
# 1-netmiko-show-improved.py

from netmiko import ConnectHandler

ios1 = {
    'device_type': 'cisco_ios',
    'ip': '198.18.1.11',
    'username': 'cisco',
    'password': 'cisco',
}

# with statement: disconnect() called automatically when block exits
# Even if an exception occurs
with ConnectHandler(**ios1) as net_connect:
    output = net_connect.send_command('show version')
    print(output)

# net_connect.disconnect() called here automatically
print('Script complete.')
```

> 💡 **Best practice:** Always use `with ConnectHandler(...) as conn:` instead of manually calling `.disconnect()`. It is cleaner, safer, and consistent with professional Python style.

---

## Lab Script 2 — `2-netmiko-config.py`

Sending configuration commands instead of show commands. One key difference: use `send_config_set()` instead of `send_command()`.

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

# Establish SSH to device and run config command
net_connect = ConnectHandler(**ios1)
output = net_connect.send_config_set('logging host 10.1.1.1')
net_connect.disconnect()
print(output)
```

### What `send_config_set()` Does

Unlike `send_command()`, `send_config_set()`:
1. Enters `configure terminal` automatically
2. Sends your command(s)
3. Exits config mode automatically (`end`)

It accepts either a **string** (single command) or a **list** (multiple commands):

```python
# Single command — string
output = net_connect.send_config_set('logging host 10.1.1.1')

# Multiple commands — list (preferred)
config_commands = [
    'logging console',
    'logging host 10.1.1.3',
    'ntp server 10.1.1.4',
    'ip name-server 10.1.1.5',
    'no ip http server',
]
output = net_connect.send_config_set(config_commands)
```

### Expected Output

```
configure terminal
Enter configuration commands, one per line.  End with CNTL/Z.
CSR1(config)#logging host 10.1.1.1
CSR1(config)#end
CSR1#
```

### Saving the Configuration

Always save after config changes:

```python
with ConnectHandler(**ios1) as conn:
    output = conn.send_config_set('logging host 10.1.1.1')
    save_output = conn.save_config()   # Equivalent to 'write memory'
    print(output)
    print(save_output)
```

---

## Key Methods Summary

| Method | Purpose | Returns |
|--------|---------|---------|
| `send_command(cmd)` | Run a show command | String |
| `send_command(cmd, use_textfsm=True)` | Run + parse with TextFSM | List of dicts |
| `send_command(cmd, use_genie=True)` | Run + parse with Genie | Nested dict |
| `send_config_set(cmds)` | Push config commands | String |
| `send_config_from_file(path)` | Push commands from file | String |
| `save_config()` | `write memory` | String |
| `find_prompt()` | Get device prompt (e.g., `CSR1#`) | String |
| `disconnect()` | Close SSH session | None |
| `enable()` | Enter enable mode | None |

---

## Running Multiple Show Commands

```python
from netmiko import ConnectHandler

ios1 = {
    'device_type': 'cisco_ios',
    'ip': '198.18.1.11',
    'username': 'cisco',
    'password': 'cisco',
}

show_commands = [
    'show version',
    'show ip interface brief',
    'show ip route summary',
    'show processes cpu sorted | head 10',
]

with ConnectHandler(**ios1) as conn:
    for command in show_commands:
        print(f'\n{"="*55}')
        print(f'Command: {command}')
        print('='*55)
        output = conn.send_command(command)
        print(output)
```

---

## Troubleshooting Common Issues

| Error | Cause | Fix |
|-------|-------|-----|
| `NetmikoTimeoutException` | Device unreachable | Check IP, firewall, SSH enabled |
| `NetmikoAuthenticationException` | Wrong credentials | Verify username/password |
| `SSHException` | SSH not enabled | `ip ssh version 2` on device |
| `ValueError: device_type not found` | Typo in device_type | Use exact strings from the table above |
| Hanging indefinitely | Slow device / wrong prompt | Increase `read_timeout` parameter |

```python
# Increase timeout for slow devices
with ConnectHandler(**ios1, read_timeout=60, conn_timeout=30) as conn:
    output = conn.send_command('show running-config', read_timeout=120)
```

---

## Practice Exercises

1. Run script `1-netmiko-show.py` against your lab device. What IOS version is it running?

2. Modify the script to run `show ip interface brief` instead of `show version`. What interfaces are shown?

3. Rewrite script 1 using the `with` statement.

4. Run script `2-netmiko-config.py`. Verify the logging config was applied with `show running-config | include logging`.

5. Modify script 2 to push 4 configuration commands (your choice) as a list.

---

*Previous: [Chapter 5](../section1/ch05_classes_modules.md) | Next: [Chapter 7 — Multi-Device Automation and File-Driven Inventory](ch07_multi_device.md)*
