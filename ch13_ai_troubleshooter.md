# Chapter 13 — The AI Network Troubleshooter

> *"The expert in anything was once a beginner — but unlike a beginner, the expert knows what questions to ask."*

---

## Chapter Goal

Build a complete AI-powered troubleshooter that combines Netmiko's device connectivity with Claude's analytical capabilities. The tool connects to any device, runs a full diagnostic suite, and returns a structured health assessment — in the time it used to take to type `show version`.

**Key Points:**
- Architecture: collect → structure → analyze → display
- Collecting 11 diagnostic commands from live devices
- Formatting device data for AI consumption (token management)
- The full `NetworkAIAssessor` implementation
- Interactive follow-up questions with conversation memory

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│  1. User provides device IP + credentials                    │
└────────────────────────────┬────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────┐
│  2. Netmiko connects and runs 11 diagnostic commands         │
│     show version | show ip int brief | show processes cpu    │
│     show logging | show ip bgp summary | show ip ospf nbr    │
│     show platform resources | show cdp neighbors | ...       │
└────────────────────────────┬────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────┐
│  3. All outputs formatted into structured context string      │
│     Each command labeled, truncated to fit token limits       │
└────────────────────────────┬────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────┐
│  4. Claude API analyzes full context                         │
│     Returns: Health Score, Issues, Root Cause, Actions       │
└────────────────────────────┬────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────┐
│  5. Display assessment + optional follow-up Q&A              │
└─────────────────────────────────────────────────────────────┘
```

---

## The Full Script

```python
#!/usr/bin/env python3
"""
network_ai_troubleshooter.py

AI-powered network device health assessor.
Collects live device data via Netmiko, analyzes with Claude.

Usage:
    python network_ai_troubleshooter.py
    
Environment variables:
    NETMIKO_USERNAME, NETMIKO_PASSWORD, ANTHROPIC_API_KEY
"""

import anthropic
import os
from getpass import getpass
from netmiko import ConnectHandler
from netmiko.exceptions import NetmikoAuthenticationException, NetmikoTimeoutException
from paramiko.ssh_exception import SSHException


# ─────────────────────────────────────────────────────────────
# CONFIGURATION
# ─────────────────────────────────────────────────────────────

DIAGNOSTIC_COMMANDS = {
    "version":        "show version",
    "interfaces":     "show ip interface brief",
    "cpu":            "show processes cpu sorted | head 15",
    "memory":         "show platform resources",
    "environment":    "show environment all",
    "logs":           "show logging | tail 40",
    "routes":         "show ip route summary",
    "bgp_summary":    "show ip bgp summary",
    "ospf_neighbors": "show ip ospf neighbor",
    "cdp_neighbors":  "show cdp neighbors",
    "spanning_tree":  "show spanning-tree summary",
}

SYSTEM_PROMPT = """You are a senior Cisco CX Network Engineer performing 
a comprehensive device health assessment. You have expert-level knowledge of:
- Cisco IOS-XE, Catalyst 9000, ASR, ISR, CSR platforms
- BGP, OSPF, EIGRP, STP, and enterprise protocols
- Network performance analysis and security hardening
- Cisco lifecycle management and PSIRT advisories

When analyzing device data:
1. Identify all issues by severity (P1=Critical outage, P2=Degraded service, P3=Risk, P4=Advisory)
2. Explain root cause clearly in plain English
3. Provide specific, actionable remediation steps
4. Cite relevant show commands for verification
5. Note any lifecycle or security concerns

Structure EVERY response with these exact sections:

## Health Score: X/10
[one-line justification]

## Issues Found
[P1/P2/P3/P4] **Issue Title** — brief description and impact

## Root Cause Analysis
[the most likely underlying problem connecting all the symptoms]

## Recommended Actions
1. [action — highest priority first]
2. [action]
3. [action]

## Verification Commands
- `command` — what to look for"""


# ─────────────────────────────────────────────────────────────
# DEVICE DATA COLLECTION
# ─────────────────────────────────────────────────────────────

def collect_device_data(device_params):
    """
    SSH to device and run all diagnostic commands.
    
    Returns:
        (dict, None)  on success — dict keyed by command name
        (None, str)   on failure — str is the error message
    """
    ip = device_params.get('ip', 'unknown')
    collected = {}
    errors = []

    print(f"\n{'='*55}")
    print(f"  Connecting to {ip}...")
    print(f"{'='*55}")

    try:
        with ConnectHandler(**device_params) as conn:
            # Get actual hostname from device prompt
            hostname = conn.find_prompt().rstrip('#>')
            collected['_hostname'] = hostname
            print(f"  Connected: {hostname}")

            for key, command in DIAGNOSTIC_COMMANDS.items():
                try:
                    print(f"  ├── {command}")
                    output = conn.send_command(command, read_timeout=30)
                    collected[key] = output
                except Exception as cmd_err:
                    collected[key] = f"[ERROR running '{command}': {cmd_err}]"
                    errors.append(command)

    except NetmikoAuthenticationException:
        return None, "Authentication failure — check credentials"
    except NetmikoTimeoutException:
        return None, "Connection timeout — device unreachable"
    except SSHException as e:
        return None, f"SSH error: {e}"
    except Exception as e:
        return None, f"Unexpected error: {e}"

    if errors:
        print(f"\n  ⚠️  {len(errors)} commands failed")

    print(f"\n  ✅ Data collection complete ({len(collected)-1} outputs)")
    return collected, None


# ─────────────────────────────────────────────────────────────
# AI ANALYSIS
# ─────────────────────────────────────────────────────────────

def build_context_string(collected_data):
    """Format device data as a structured context string for the AI."""
    hostname = collected_data.get('_hostname', 'Unknown')
    sections = [f"# Device Health Data: {hostname}\n"]

    for key, output in collected_data.items():
        if key.startswith('_'):
            continue
        command = DIAGNOSTIC_COMMANDS.get(key, key)
        # Truncate long outputs — token budget management
        truncated = output[:1500] if len(output) > 1500 else output
        sections.append(f"\n## {command}\n```\n{truncated}\n```")

    return '\n'.join(sections)


def analyze_with_ai(collected_data, question=None):
    """
    Send collected device data to Claude for analysis.
    Returns the AI assessment as a string.
    """
    client = anthropic.Anthropic()
    context = build_context_string(collected_data)
    hostname = collected_data.get('_hostname', 'the device')

    default_q = (
        f"Perform a comprehensive health assessment of {hostname}. "
        f"Identify all issues, explain root causes, and provide a prioritized action plan."
    )

    user_message = f"""
{context}

---

**Assessment Request:** {question or default_q}
"""

    print(f"\n{'='*55}")
    print("  Sending data to Claude AI for analysis...")
    print(f"{'='*55}\n")

    response = client.messages.create(
        model="claude-opus-4-6",
        max_tokens=3000,
        system=SYSTEM_PROMPT,
        messages=[{"role": "user", "content": user_message}]
    )

    return response.content[0].text


# ─────────────────────────────────────────────────────────────
# INTERACTIVE FOLLOW-UP
# ─────────────────────────────────────────────────────────────

def interactive_followup(collected_data, initial_analysis):
    """Allow follow-up questions about the device using full context."""
    client = anthropic.Anthropic()
    context = build_context_string(collected_data)

    # Start history with initial exchange
    history = [
        {
            "role": "user",
            "content": f"Device data:\n\n{context}\n\nProvide initial health assessment."
        },
        {
            "role": "assistant",
            "content": initial_analysis
        }
    ]

    print("\n" + "="*55)
    print("  Interactive mode — type 'done' to exit")
    print("="*55)

    while True:
        try:
            question = input("\n❓ Your question: ").strip()
        except (KeyboardInterrupt, EOFError):
            break

        if question.lower() in ['done', 'exit', 'quit', 'q']:
            break
        if not question:
            continue

        history.append({"role": "user", "content": question})

        response = client.messages.create(
            model="claude-opus-4-6",
            max_tokens=1024,
            system=SYSTEM_PROMPT,
            messages=history
        )

        answer = response.content[0].text
        history.append({"role": "assistant", "content": answer})
        print(f"\n🤖 AI: {answer}")


# ─────────────────────────────────────────────────────────────
# MAIN
# ─────────────────────────────────────────────────────────────

def main():
    print("╔══════════════════════════════════════════════╗")
    print("║     AI-Powered Network Troubleshooter        ║")
    print("║     Netmiko + Claude                         ║")
    print("╚══════════════════════════════════════════════╝")

    # Credentials
    username   = os.getenv("NETMIKO_USERNAME") or input("\nSSH Username: ")
    password   = os.getenv("NETMIKO_PASSWORD") or getpass("SSH Password: ")
    device_ip  = input("Device IP address: ").strip()

    device_params = {
        "device_type": "cisco_ios",
        "ip":          device_ip,
        "username":    username,
        "password":    password,
    }

    # Step 1: Collect
    collected, error = collect_device_data(device_params)
    if not collected:
        print(f"\n❌ Collection failed: {error}")
        return

    # Step 2: Analyze
    analysis = analyze_with_ai(collected)

    print("\n" + "━"*55)
    print("  AI HEALTH ASSESSMENT")
    print("━"*55)
    print(analysis)

    # Step 3: Optional interactive follow-up
    answer = input("\n\nAsk follow-up questions? (y/n): ").strip().lower()
    if answer == 'y':
        interactive_followup(collected, analysis)

    print("\n✅ Assessment complete.")


if __name__ == "__main__":
    main()
```

---

## Quick Version: Minimal Code

For a fast one-off check without the full framework:

```python
# quick_ai_check.py
import anthropic
from netmiko import ConnectHandler

def quick_check(ip, username, password, question="Identify any issues."):
    commands = ["show version", "show ip interface brief", "show logging | tail 20"]

    with ConnectHandler(device_type="cisco_ios", ip=ip,
                        username=username, password=password) as conn:
        context = ""
        for cmd in commands:
            context += f"\n### {cmd}\n{conn.send_command(cmd)}\n"

    client = anthropic.Anthropic()
    response = client.messages.create(
        model="claude-opus-4-6",
        max_tokens=1500,
        system="You are a Cisco network expert. Be concise and specific.",
        messages=[{"role": "user", "content": f"{context}\n\n{question}"}]
    )
    return response.content[0].text

if __name__ == "__main__":
    import os
    from getpass import getpass
    ip  = input("Device IP: ")
    u   = os.getenv("NETMIKO_USERNAME") or input("Username: ")
    p   = os.getenv("NETMIKO_PASSWORD") or getpass("Password: ")
    print(quick_check(ip, u, p))
```

---

## Practice Exercises

1. Run the troubleshooter against your lab device. What health score does it assign?
2. Add `show interfaces` to `DIAGNOSTIC_COMMANDS` and test.
3. Modify the script to save the AI assessment to `{hostname}_{timestamp}_assessment.txt`.

---

*Previous: [Chapter 12](ch12_llm_api_prompts.md) | Next: [Chapter 14 — AI Syslog Analyzer and Config Auditor](ch14_ai_syslog_auditor.md)*

---
