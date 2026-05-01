# Chapter 11 — Git, CI/CD, and Treating Your Network as Code

> *"Infrastructure as Code is not about writing scripts. It is about managing infrastructure with the same rigor and discipline applied to software."*  
> — Kief Morris

---

## Chapter Goal

Apply software engineering discipline to network automation. Every script you write should be version-controlled, tested, and deployed through a repeatable process. This chapter covers Git fundamentals, branching strategies, and CI/CD pipelines for network automation — the foundation of a professional NetDevOps practice.

**Key Points:**
- Why version control is non-negotiable for automation code
- Git workflow: add, commit, push, branch, merge
- GitLab CI/CD pipeline for network automation validation
- The `.gitignore` patterns that protect your credentials
- Infrastructure as Code mindset: your network config as a Git repo

---

## Why Network Automation Code Needs Git

Before automation, network configuration changes were tracked loosely — change tickets, email threads, maybe a spreadsheet. The "authoritative version" of a configuration was whatever was running on the device.

With automation code, you need a better answer to: "What version of this script changed 200 devices last Tuesday, and who approved it?"

Git answers that question. Every commit is:
- A complete snapshot of your code at that moment
- Timestamped and attributed to a specific author
- Accompanied by a message explaining what changed and why
- Reversible — you can roll back to any previous commit

For a network team, this provides:

| Without Git | With Git |
|-------------|----------|
| "Who changed the script?" | `git log` shows every change |
| "What did it look like before?" | `git checkout <hash>` |
| "Two people edited it at once" | Merge conflicts are surfaced and resolved |
| "Is prod running the latest version?" | `git status` shows instantly |

---

## Git Fundamentals

### Initial Setup

```bash
# Set your identity (one-time)
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

# Check version
git --version
```

### Creating a Repository

```bash
# Initialize a new repo in your project directory
mkdir network-automation
cd network-automation
git init

# Or clone an existing repo
git clone https://github.com/your-org/network-automation.git
```

### The Basic Workflow

```bash
# 1. See what has changed
git status

# 2. Stage changes for commit
git add script.py            # One file
git add .                    # All changed files

# 3. Commit with a message
git commit -m "Add VLAN provisioning script for access switches"

# 4. Push to remote (GitLab/GitHub)
git push origin main
```

### Essential Git Commands

| Command | What It Does |
|---------|-------------|
| `git status` | Show current state — what's changed, staged, untracked |
| `git log --oneline` | Short commit history |
| `git diff` | Show uncommitted changes |
| `git add .` | Stage all changes |
| `git commit -m "msg"` | Commit staged changes |
| `git push origin main` | Push to remote |
| `git pull` | Get latest from remote |
| `git branch feature/x` | Create a new branch |
| `git checkout feature/x` | Switch to a branch |
| `git merge feature/x` | Merge branch into current |
| `git log --oneline -10` | Last 10 commits, one line each |

---

## The `.gitignore` File: Protecting Your Credentials

Before your first commit, create `.gitignore` to prevent credentials and temporary files from being committed:

```bash
# .gitignore for network automation projects

# Credentials — NEVER commit these
.env
secrets.yml
*password*
*credentials*

# Python artifacts
__pycache__/
*.pyc
*.pyo
.pytest_cache/
*.egg-info/

# Virtual environments
venv/
.venv/
env/

# Output files (may contain sensitive data)
backup/
output/
*.cfg
*.log

# OS files
.DS_Store
Thumbs.db
```

> ⚠️ **Critical:** Add `.gitignore` BEFORE your first commit. If you accidentally commit credentials, they persist in Git history even after deletion. Use `git filter-branch` or BFG Repo Cleaner to scrub them — a painful process. Prevention is far easier.

---

## Writing Good Commit Messages

Commit messages are the documentation of your automation history. Future-you reading git log six months from now will thank present-you for writing clear messages.

```bash
# ❌ Bad commit messages
git commit -m "fix"
git commit -m "stuff"
git commit -m "changes"

# ✅ Good commit messages
git commit -m "Add exception handling for authentication failures in backup script"
git commit -m "Fix VLAN command generation for NX-OS devices"
git commit -m "Refactor inventory loading to support CSV and YAML formats"
```

The convention: imperative mood, specific about what changed and why. "Add X" not "Added X". "Fix Y" not "Fixed Y".

---

## Branching Strategy for Network Automation

Use branches to isolate changes until they are validated:

```bash
# Create and switch to a feature branch
git checkout -b feature/add-nornir-backup

# Make changes, test them
# ...

# Commit on the branch
git add .
git commit -m "Add parallel backup using Nornir"

# Push the branch
git push origin feature/add-nornir-backup

# Create a merge request (GitLab) or pull request (GitHub)
# After review and approval, merge to main
```

A simple branching model for network automation teams:

```
main          ← always deployable, protected branch
  └── feature/add-nornir-backup   ← new feature development
  └── fix/timeout-exception       ← bug fixes
  └── release/2024-Q2             ← optional: release branches
```

---

## CI/CD Pipelines for Network Automation

A CI/CD pipeline automatically tests your scripts every time you push code. No more "it worked on my laptop" — the pipeline proves it works in a clean, repeatable environment.

### GitLab CI Example: `.gitlab-ci.yml`

```yaml
# .gitlab-ci.yml — pipeline for network automation repo
stages:
  - lint
  - test
  - validate
  - report

# Stage 1: Code quality
lint:
  stage: lint
  image: python:3.8
  script:
    - pip install flake8 pylint
    - flake8 scripts/ --max-line-length=100
  allow_failure: false

# Stage 2: Unit tests
test:
  stage: test
  image: python:3.8
  script:
    - pip install pytest netmiko
    - pytest tests/ -v --tb=short
  coverage: '/TOTAL.*\s+(\d+%)$/'

# Stage 3: Validate configuration templates
validate_configs:
  stage: validate
  image: python:3.8
  script:
    - pip install netmiko pyyaml
    - python scripts/validate_templates.py
  only:
    - main
    - merge_requests

# Stage 4: Generate inventory report (on schedule)
inventory_report:
  stage: report
  script:
    - pip install netmiko tabulate genie pyats
    - python scripts/10-netmiko-report.py
  only:
    - schedules    # Run on nightly schedule
  artifacts:
    paths:
      - inventory.csv
    expire_in: 30 days
```

### What Each Stage Does

**`lint`:** Runs `flake8` to check code style. Catches syntax errors, undefined variables, unused imports before they reach production.

**`test`:** Runs your pytest unit tests. These are fast tests against mock devices or test data — not against live production.

**`validate_configs`:** Runs a validation script that checks your config templates are syntactically valid and would not generate invalid IOS commands.

**`inventory_report`:** Runs on a nightly schedule against production devices and saves the CSV as a pipeline artifact (downloadable from GitLab UI).

---

## Project Structure for a Professional Automation Repo

```
network-automation/
├── .gitignore              ← Exclude credentials, backups, pyc
├── .gitlab-ci.yml          ← CI/CD pipeline definition
├── README.md               ← How to use this repo
├── requirements.txt        ← Python dependencies
│
├── scripts/                ← Runnable automation scripts
│   ├── 01_vlan_deploy.py
│   ├── 02_config_backup.py
│   └── 03_inventory_report.py
│
├── templates/              ← Configuration templates
│   ├── access_switch_baseline.cfg
│   └── core_switch_baseline.cfg
│
├── inventory/              ← Device inventory files
│   ├── production.csv
│   └── staging.csv
│
├── tests/                  ← Unit tests
│   ├── test_vlan_commands.py
│   └── test_inventory.py
│
└── output/                 ← Generated reports (in .gitignore)
    └── .gitkeep            ← Keeps empty dir in Git
```

---

## Infrastructure as Code: Your Network as a Git Repo

The ultimate goal is treating your **network configuration** as code — not just your automation scripts. This means:

1. **Desired state in YAML/JSON:** Define what every device should look like
2. **Automation enforces state:** Scripts compare current vs desired, fix drift
3. **All changes via Git:** No ad-hoc CLI changes that bypass version control
4. **CI/CD validates changes:** Every proposed change passes automated tests before reaching production

```yaml
# desired_state/sw-core-01.yml
hostname: sw-core-01
platform: catalyst9300
vlans:
  - id: 10
    name: MGMT
  - id: 20
    name: USERS
ntp_servers:
  - 10.1.1.4
  - 10.1.1.5
logging_servers:
  - 10.1.1.3
```

```python
# enforce_state.py — compare desired vs actual, fix drift
def enforce_device_state(device, desired_state_file):
    """Ensure device matches its desired state YAML."""
    with open(desired_state_file) as f:
        desired = yaml.safe_load(f)
    
    with ConnectHandler(**device) as conn:
        # Collect current state
        current_vlans = conn.send_command('show vlan brief', use_genie=True)
        
        # Compare and fix
        for vlan in desired['vlans']:
            if not vlan_exists(current_vlans, vlan['id']):
                print(f"Drift detected: VLAN {vlan['id']} missing — applying")
                conn.send_config_set([
                    f"vlan {vlan['id']}",
                    f"name {vlan['name']}",
                ])
```

This pattern — declare desired state, detect drift, correct it — is the foundation of Ansible, Terraform, and every mature IaC tool. Understanding it from first principles here makes those tools immediately comprehensible.

---

## Practice Exercises

1. Initialize a Git repository in your automation project folder. Create a `.gitignore` file and make your first commit.

2. Write a commit message for each of these changes (practice the imperative format):
   - You added exception handling for timeouts
   - You changed the output format from text to JSON
   - You fixed a bug where hostnames with special characters crashed the backup script

3. Create a `.gitlab-ci.yml` with a single `lint` stage that runs `python -m py_compile` on all `.py` files. This is the simplest possible pipeline.

---

*Previous: [Chapter 10](ch10_nornir_parallel.md) | Next: [Chapter 12 — LLM APIs and Prompt Engineering](../section3/ch12_llm_api_prompts.md)*
