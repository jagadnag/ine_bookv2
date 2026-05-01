# Chapter 8 — Production Scripts: Exceptions, Env Vars, and Show Commands

## Chapter Goal

Add the three elements that separate development scripts from production scripts: comprehensive exception handling, secure credential management with environment variables, and multi-command show collection from files. Covers lab scripts 5, 6, and 7.

---

## Lab Script 5 — `5-netmiko-final.py`: Full Exception Handling

### The Full Script

```python
#!/usr/bin/env python
from getpass import getpass
from netmiko import ConnectHandler
from netmiko import NetmikoAuthenticationException
from netmiko import NetmikoTimeoutException
from paramiko.ssh_exception import SSHException

# Collect login credentials
username = input('Enter your SSH username: ')
password = getpass('Enter your password: ')

# Read device list from file
with open('device_list') as f:
    device_list = f.read().splitlines()

# Iterate through device list and configure
for devices in device_list:
    print('Connecting to device ' + devices)
    ip_address_of_device = devices
    ios_device = {
        'device_type': 'cisco_ios',
        'ip': ip_address_of_device,
        'username': username,
        'password': password
    }

    try:
        net_connect = ConnectHandler(**ios_device)
    except (NetmikoAuthenticationException):
        print('Authentication failure: ' + ip_address_of_device)
        continue
    except (NetmikoTimeoutException):
        print('Timeout to device: ' + ip_address_of_device)
        continue
    except (EOFError):
        print('End of file while attempting device ' + ip_address_of_device)
        continue
    except (SSHException):
        print('SSH Issue. Are you sure SSH is enabled? ' + ip_address_of_device)
        continue
    except Exception as unknown_error:
        print('Some other error: ' + str(unknown_error))
        continue

    # Only runs if connection succeeded
    output  = net_connect.send_config_from_file('config_commands')
    output += net_connect.save_config()
    net_connect.disconnect()
    print(output)
```

### The Exception Architecture

Four specific exception types, each with a targeted message, followed by a catch-all:

| Exception | Meaning | Remediation |
|-----------|---------|-------------|
| `NetmikoAuthenticationException` | Wrong credentials | Check username/password, AAA config |
| `NetmikoTimeoutException` | Can't reach device | Check IP, firewall, SSH enabled |
| `EOFError` | Connection dropped mid-session | VTY limit, ACL blocking |
| `SSHException` | SSH protocol issue | Enable SSH v2, check key |
| `Exception` | Anything else | Log and skip |

**Why `continue`?**  
Without `continue`, a connection failure causes Python to try `send_config_from_file()` — which fails because there is no connection — and the script crashes. With `continue`, the failed device is skipped and the loop moves to the next IP.

**`output += net_connect.save_config()`**  
`+=` appends the save output to the config output string. The combined output gives you a complete record of what happened on the device.

> 🔑 **Script 5 is the template for every production Netmiko script you will write.** Every element — file inventory, runtime credentials, specific exception handling, continue-on-failure, save config — is a production requirement.

---

## Lab Script 6 — `6-netmiko-final.py`: Environment Variables

### The Full Script

```python
#!/usr/bin/env python
import os
from getpass import getpass
from netmiko import ConnectHandler
from netmiko import NetmikoAuthenticationException
from netmiko import NetmikoTimeoutException
from paramiko.ssh_exception import SSHException

# Check environment variable for credentials; fall back to getpass
username = os.getenv("NETMIKO_USERNAME") if os.getenv("NETMIKO_USERNAME") else input('Enter username: ')
password = os.getenv("NETMIKO_PASSWORD") if os.getenv("NETMIKO_PASSWORD") else getpass()

# (rest of script identical to script 5)
with open('device_list') as f:
    device_list = f.read().splitlines()

for devices in device_list:
    print('Connecting to device ' + devices)
    ios_device = {
        'device_type': 'cisco_ios',
        'ip': devices,
        'username': username,
        'password': password
    }
    try:
        net_connect = ConnectHandler(**ios_device)
    except NetmikoAuthenticationException:
        print('Authentication failure: ' + devices)
        continue
    except NetmikoTimeoutException:
        print('Timeout to device: ' + devices)
        continue
    except EOFError:
        print('End of file: ' + devices)
        continue
    except SSHException:
        print('SSH Issue: ' + devices)
        continue
    except Exception as e:
        print('Unknown error: ' + str(e))
        continue

    output  = net_connect.send_config_from_file('config_commands')
    output += net_connect.save_config()
    net_connect.disconnect()
    print(output)
```

### The Credential Pattern

```python
username = os.getenv("NETMIKO_USERNAME") if os.getenv("NETMIKO_USERNAME") else input('Enter username: ')
```

This reads as: "If the environment variable NETMIKO_USERNAME is set, use it. Otherwise, prompt the user."

**Setting variables before running:**

```bash
# Linux / Mac
export NETMIKO_USERNAME="cisco"
export NETMIKO_PASSWORD="cisco"
python 6-netmiko-final.py

# Windows PowerShell
$env:NETMIKO_USERNAME = "cisco"
$env:NETMIKO_PASSWORD = "cisco"
python 6-netmiko-final.py
```

**For automated runs (cron, CI/CD):**  
The platform injects environment variables. GitLab CI sets them in Settings → CI/CD → Variables. Jenkins sets them in credentials. Ansible Tower uses the Vault. Your script works the same way — it just reads the environment.

---

## Lab Script 7 — `7-netmiko-final.py`: Multiple Show Commands from File

### The Full Script

```python
#!/usr/bin/env python
import os
from getpass import getpass
from netmiko import ConnectHandler
from netmiko import NetmikoAuthenticationException
from netmiko import NetmikoTimeoutException
from paramiko.ssh_exception import SSHException

username = os.getenv("NETMIKO_USERNAME") if os.getenv("NETMIKO_USERNAME") else input('Enter username: ')
password = os.getenv("NETMIKO_PASSWORD") if os.getenv("NETMIKO_PASSWORD") else getpass()

# Read device IPs
with open('device_list') as f:
    device_list = f.read().splitlines()

# Read show commands from file
with open('show_command') as f:
    show_commands = f.readlines()

for devices in device_list:
    print('Connecting to device ' + devices)
    ios_device = {
        'device_type': 'cisco_ios',
        'ip': devices,
        'username': username,
        'password': password
    }
    try:
        net_connect = ConnectHandler(**ios_device)
    except NetmikoAuthenticationException:
        print('Authentication failure: ' + devices)
        continue
    except NetmikoTimeoutException:
        print('Timeout to device: ' + devices)
        continue
    except EOFError:
        print('End of file: ' + devices)
        continue
    except SSHException:
        print('SSH Issue: ' + devices)
        continue
    except Exception as e:
        print('Unknown error: ' + str(e))
        continue

    # Inner loop: run every show command on this device
    net_connect = ConnectHandler(**ios_device)
    for command in show_commands:
        output = net_connect.send_command(command)
        print(command + output + '\n')
    net_connect.disconnect()
```

### The `show_command` File

```
show runn | in logging
show runn | in ntp
show runn | in snmp
show ip access-lists TEST_ACL
show ip ospf neighbor
show ip ospf interface brief
show ip interface brief
```

### The Nested Loop

```python
for devices in device_list:       # Outer: each device
    ...
    for command in show_commands:  # Inner: each command
        output = net_connect.send_command(command)
        print(command + output + '\n')
```

For 2 devices and 7 commands: 14 `send_command()` calls total. Every command runs on every device. This is the **pre/post change validation pattern** — run it before a change window, save the output, run it again after, compare.

**`f.readlines()` vs `f.read().splitlines()`:**  
`readlines()` keeps the `\n` at the end of each line. `splitlines()` strips them. For `send_command()`, both work — Netmiko strips leading/trailing whitespace from commands automatically.

---

*Previous: [Chapter 7](ch07_multi_device.md) | Next: [Chapter 9 — Backup and Reports](ch09_backup_reports.md)*

---
