# Chapter 10 — Nornir: Parallel Automation at Scale

## Chapter Goal

Run automation against your entire fleet simultaneously instead of sequentially. Nornir uses YAML-based inventory and Python's threading to connect to hundreds of devices at once. Covers lab scripts 11 and 12.

---

## Serial vs Parallel: The Performance Problem

| Fleet Size | Serial (30s/device) | Nornir Parallel |
|-----------|---------------------|-----------------|
| 10 devices | 5 minutes | ~30 seconds |
| 50 devices | 25 minutes | ~30 seconds |
| 100 devices | 50 minutes | ~30 seconds |
| 500 devices | 4+ hours | ~1 minute |

Netmiko processes devices one by one. Nornir runs them all simultaneously, bounded only by the slowest device. For large fleet automation, this is the difference between fitting in a maintenance window and not.

---

## Nornir Inventory Files

Nornir uses YAML files to define inventory — easy to read, version-control friendly.

### `inventory/hosts.yml`

```yaml
---
csr1:
  hostname: 198.18.1.11
  groups:
    - cisco

csr2:
  hostname: 198.18.1.12
  groups:
    - cisco
```

### `inventory/groups.yml`

```yaml
---
cisco:
  platform: ios
  port: 22
```

### `inventory/defaults.yml`

```yaml
---
username: "cisco"
password: "cisco"
```

### `config.yml`

```yaml
---
runners:
  plugin: threaded
  options:
    num_workers: 100    # Run up to 100 devices simultaneously

inventory:
  plugin: "SimpleInventory"
  options:
    host_file: "./inventory/hosts.yml"
    group_file: "./inventory/groups.yml"
    defaults_file: "./inventory/defaults.yml"
```

---

## Lab Script 11 — `11-nornir-basics.py`

### The Full Script

```python
#!/usr/bin/env python
from nornir import InitNornir
from nornir_netmiko.tasks import netmiko_send_command, netmiko_send_config
from nornir_utils.plugins.functions import print_result

# Initialize Nornir with YAML inventory
nr = InitNornir(
    config_file="config.yml", dry_run=True
)

# Run command on ALL devices simultaneously
result = nr.run(netmiko_send_command, command_string="show ip int brief")

# Print structured results
print_result(result)
```

### Walkthrough

**`nr = InitNornir(config_file="config.yml", dry_run=True)`**  
Loads the Nornir configuration and inventory. `dry_run=True` means Nornir reads the inventory but does not make any changes. Remove for production use.

**`result = nr.run(netmiko_send_command, command_string="show ip int brief")`**  
Dispatches `show ip int brief` to ALL devices in the inventory concurrently. The `num_workers: 100` setting in `config.yml` controls max parallelism.

**`print_result(result)`**  
Formats and prints all results in a readable way, organized by hostname.

---

## Lab Script 12 — `12-nornir-backup.py`

### The Full Script

```python
#!/usr/bin/env python
import os
from nornir import InitNornir
from nornir_netmiko.tasks import netmiko_send_command
from nornir_utils.plugins.functions import print_result

BACKUP_DIR = "backup/"

nr = InitNornir(
    config_file="config.yml", dry_run=True
)

def create_backups_dir():
    if not os.path.exists(BACKUP_DIR):
        os.mkdir(BACKUP_DIR)

def save_config_to_file(hostname, config):
    filename = f"{hostname}_nornir.txt"
    with open(os.path.join(BACKUP_DIR, filename), "w") as f:
        f.write(config)

def get_netmiko_backups():
    backup_results = nr.run(
        task=netmiko_send_command,
        command_string="show run"
    )

    for hostname in backup_results:
        save_config_to_file(
            hostname=hostname,
            config=backup_results[hostname][0].result,
        )

def main():
    create_backups_dir()
    get_netmiko_backups()

if __name__ == "__main__":
    main()
```

### Professional Code Structure

This script demonstrates the cleanest Python project structure you have seen so far:

- `BACKUP_DIR = "backup/"` — a **constant** (UPPERCASE by convention)
- `create_backups_dir()` — one job: make the directory
- `save_config_to_file()` — one job: save one config
- `get_netmiko_backups()` — one job: collect configs from all devices
- `main()` — orchestration: calls the others in order
- `if __name__ == '__main__': main()` — safely runnable and importable

**`backup_results[hostname][0].result`:**  
Nornir results are indexed by hostname (`backup_results[hostname]`), then by task index (`[0]` for the first task). `.result` contains the actual output string.

### Netmiko vs Nornir

| Feature | Netmiko (Scripts 1–10) | Nornir (Scripts 11–12) |
|---------|----------------------|----------------------|
| Execution | Serial (one by one) | Parallel (simultaneous) |
| Inventory | File/hardcoded | YAML inventory system |
| Speed (100 devices) | ~50 minutes | ~1 minute |
| Learning curve | Lower | Higher |
| Best for | Learning, small labs | Production, large fleets |

---

*Previous: [Chapter 9](ch09_backup_reports.md) | Next: [Chapter 11 — Git and CI/CD](ch11_git_cicd.md)*
