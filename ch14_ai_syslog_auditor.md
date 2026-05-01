# Chapter 14 — AI Syslog Analyzer and Configuration Auditor

## Chapter Goal

Build two high-value AI tools: a syslog analyzer that triages events and groups them into incidents, and a config auditor that performs a 22-point security check against any running configuration.

---

## Part A: The AI Syslog Analyzer

### Why AI Syslog Analysis

Devices generate hundreds of syslog events per day. Most are informational noise. A few indicate real problems. Finding the signal in the noise manually is tedious and error-prone. AI does it in seconds.

### The Full Script

```python
#!/usr/bin/env python3
"""
ai_syslog_analyzer.py

Analyzes Cisco syslog messages using AI.
Groups related events into incidents and generates prioritized alerts.

Usage:
    python ai_syslog_analyzer.py               # analyze sample logs
    python ai_syslog_analyzer.py syslogs.txt   # analyze a log file
"""

import anthropic
import json
import re
import sys
from datetime import datetime
from pathlib import Path


# ─────────────────────────────────────────────────────────────
# PARSING
# ─────────────────────────────────────────────────────────────

SEVERITY_LEVELS = {
    "0": ("EMERGENCY", "🔴"), "1": ("ALERT",    "🔴"),
    "2": ("CRITICAL",  "🔴"), "3": ("ERROR",    "🟠"),
    "4": ("WARNING",   "🟡"), "5": ("NOTICE",   "🟢"),
    "6": ("INFO",      "🔵"), "7": ("DEBUG",    "⚪"),
}

# Cisco syslog format: %FACILITY-SEVERITY-MNEMONIC: description
SYSLOG_PATTERN = re.compile(
    r"%(?P<facility>[A-Z_0-9]+)-(?P<severity>[0-7])-"
    r"(?P<mnemonic>[A-Z_0-9]+):\s*(?P<description>.*)"
)

def parse_syslog_line(line):
    """Parse a raw Cisco syslog line into structured fields."""
    line = line.strip()
    if not line:
        return None
    match = SYSLOG_PATTERN.search(line)
    if match:
        sev = match.group("severity")
        sev_name, sev_icon = SEVERITY_LEVELS.get(sev, ("UNKNOWN", "❓"))
        return {
            "facility":      match.group("facility"),
            "severity":      sev,
            "severity_name": sev_name,
            "severity_icon": sev_icon,
            "mnemonic":      match.group("mnemonic"),
            "description":   match.group("description").strip(),
            "raw":           line,
            "parsed":        True,
        }
    return {"raw": line, "severity": "7", "parsed": False}


# ─────────────────────────────────────────────────────────────
# AI ANALYSIS
# ─────────────────────────────────────────────────────────────

SYSLOG_SYSTEM = """You are a Cisco network operations expert specializing 
in syslog analysis and incident detection. 

You analyze syslog messages and:
1. Group related events into logical incidents
2. Identify the most likely root cause for each incident
3. Assess business impact (P1=outage, P2=degraded, P3=risk, P4=advisory)
4. Provide specific remediation steps

You return ONLY valid JSON. No markdown. No text outside the JSON."""


def analyze_syslogs_with_ai(log_lines, device_context=""):
    """
    Use Claude to analyze a batch of syslog events.
    Returns a dict with incidents and summary.
    """
    client = anthropic.Anthropic()

    parsed = [parse_syslog_line(l) for l in log_lines if l.strip()]
    parsed = [p for p in parsed if p is not None]

    # Filter to severity 0-4 (Emergency through Warning)
    significant = [l for l in parsed if int(l.get("severity", "7")) <= 4]

    if not significant:
        return {"summary": "No significant events found.", "incidents": []}

    log_text = "\n".join(l["raw"] for l in significant[:60])

    prompt = f"""Analyze these Cisco syslog events{f' from {device_context}' if device_context else ''}.

SYSLOG EVENTS:
{log_text}

Return this exact JSON structure (no other text):
{{
  "overall_priority": "P1|P2|P3|P4",
  "total_events_analyzed": {len(significant)},
  "summary": "one sentence describing the overall situation",
  "incidents": [
    {{
      "id": 1,
      "title": "brief incident title",
      "priority": "P1|P2|P3|P4",
      "related_events": ["syslog line 1", "syslog line 2"],
      "root_cause": "most likely cause",
      "impact": "what is affected",
      "actions": ["specific action 1", "specific action 2"],
      "verify_commands": ["show command 1", "show command 2"]
    }}
  ]
}}"""

    response = client.messages.create(
        model="claude-opus-4-6",
        max_tokens=2048,
        system=SYSLOG_SYSTEM,
        messages=[{"role": "user", "content": prompt}]
    )

    raw = response.content[0].text.strip()
    raw = re.sub(r"```[a-z]*\n?", "", raw).strip()

    try:
        return json.loads(raw)
    except json.JSONDecodeError as e:
        return {"error": str(e), "raw": raw, "incidents": []}


# ─────────────────────────────────────────────────────────────
# DISPLAY
# ─────────────────────────────────────────────────────────────

PRIORITY_ICONS = {"P1": "🔴", "P2": "🟠", "P3": "🟡", "P4": "🟢"}

def display_analysis(result):
    """Pretty-print the AI syslog analysis."""
    if "error" in result:
        print(f"⚠️  Parse error: {result['error']}")
        return

    priority = result.get("overall_priority", "?")
    icon     = PRIORITY_ICONS.get(priority, "⚪")

    print(f"\n{'━'*60}")
    print(f"  SYSLOG ANALYSIS — {datetime.now().strftime('%Y-%m-%d %H:%M')}")
    print(f"{'━'*60}")
    print(f"  Overall Priority: {icon} {priority}")
    print(f"  Summary: {result.get('summary', '')}")
    print(f"  Incidents Found: {len(result.get('incidents', []))}")
    print(f"{'━'*60}")

    for incident in result.get("incidents", []):
        prio = incident.get("priority", "?")
        icon = PRIORITY_ICONS.get(prio, "⚪")
        print(f"\n{icon} [{prio}] {incident.get('title', 'Untitled')}")
        print(f"  Root Cause: {incident.get('root_cause', '?')}")
        print(f"  Impact:     {incident.get('impact', '?')}")
        for i, action in enumerate(incident.get("actions", []), 1):
            print(f"  Action {i}: {action}")
        for cmd in incident.get("verify_commands", []):
            print(f"  Verify:   > {cmd}")


# ─────────────────────────────────────────────────────────────
# SAMPLE DATA + MAIN
# ─────────────────────────────────────────────────────────────

SAMPLE_SYSLOGS = [
    "%LINK-3-UPDOWN: Interface GigabitEthernet0/1, changed state to down",
    "%LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet0/1, changed state to down",
    "%BGP-5-ADJCHANGE: neighbor 10.0.0.2 Down Hold Timer Expired",
    "%BGP-3-NOTIFICATION: sent to neighbor 10.0.0.2 4/0 (hold time expired) 0 bytes",
    "%OSPF-5-ADJCHG: Process 10, Nbr 10.0.0.2 on GigabitEthernet0/1 from FULL to DOWN",
    "%SYS-2-MALLOCFAIL: Memory allocation of 65536 bytes failed from 0x0",
    "%SYS-5-CONFIG_I: Configured from console by admin on vty0 (10.10.1.100)",
    "%SEC-6-IPACCESSLOGP: list MGMT_ACL denied tcp 192.168.99.5(52341) -> 10.10.1.1(22)",
]


def main():
    if len(sys.argv) > 1:
        filepath = sys.argv[1]
        lines = Path(filepath).read_text().splitlines()
        context = filepath
        print(f"Analyzing {len(lines)} lines from {filepath}...")
    else:
        lines   = SAMPLE_SYSLOGS
        context = "sw-core-01"
        print("Using sample syslog data...")

    result = analyze_syslogs_with_ai(lines, context)
    display_analysis(result)


if __name__ == "__main__":
    main()
```

---

## Part B: The AI Configuration Auditor

```python
#!/usr/bin/env python3
"""
ai_config_auditor.py

AI-powered Cisco IOS-XE security configuration auditor.
22-point security check with structured JSON output.

Usage:
    python ai_config_auditor.py --device 10.10.1.1
    python ai_config_auditor.py --config running_config.txt
"""

import anthropic, json, os, re, sys
from getpass import getpass
from datetime import datetime
from pathlib import Path

try:
    from netmiko import ConnectHandler
    NETMIKO_AVAILABLE = True
except ImportError:
    NETMIKO_AVAILABLE = False


SECURITY_CHECKS = """
Audit this IOS-XE configuration against these 22 security requirements:

AUTHENTICATION & ACCESS
1. Is 'aaa new-model' configured?
2. Is 'service password-encryption' enabled?
3. Is 'enable secret' used (not 'enable password')?
4. Do VTY lines have 'access-class' ACLs?
5. Is 'transport input ssh' enforced on VTY (no telnet)?
6. Is 'exec-timeout' set on VTY and console?
7. Is 'ip ssh version 2' configured?

MANAGEMENT PLANE
8. Is 'no ip http server' set?
9. Is NTP configured with authentication?
10. Is syslog configured to a remote server?
11. Is SNMPv3 used? (no v1/v2 community strings)
12. Is CDP disabled on external interfaces?
13. Are login and MOTD banners configured?

NETWORK SERVICES
14. Are 'tcp-small-servers' and 'udp-small-servers' disabled?
15. Is 'no ip proxy-arp' set on interfaces?
16. Is 'no ip directed-broadcast' configured?
17. Is 'no ip source-route' configured?
18. Is LLDP disabled if not needed?

ADDITIONAL SECURITY
19. Is gratuitous ARP protection configured?
20. Is DHCP snooping enabled on appropriate VLANs?
21. Is port security configured on access ports?
22. Are there any cleartext passwords in the config?
"""

AUDITOR_SYSTEM = """You are a Cisco security specialist performing configuration audits.
You return ONLY valid JSON. No markdown. No text outside the JSON structure."""


def collect_config_from_device(ip, username, password):
    """SSH to device and collect running config."""
    if not NETMIKO_AVAILABLE:
        print("ERROR: netmiko not installed. Use --config flag.")
        return None, None
    device = {"device_type": "cisco_ios", "ip": ip,
               "username": username, "password": password}
    try:
        with ConnectHandler(**device) as conn:
            hostname = conn.find_prompt().rstrip("#>")
            conn.enable()
            config  = conn.send_command("show running-config", read_timeout=60)
            version = conn.send_command("show version")
            return hostname, config + "\n\n--- SHOW VERSION ---\n" + version
    except Exception as e:
        print(f"Connection failed: {e}")
        return None, None


def audit_config_with_ai(hostname, config):
    """Run AI security audit. Returns structured JSON result."""
    client = anthropic.Anthropic()
    config_snippet = config[:6000] if len(config) > 6000 else config

    prompt = f"""Audit this Cisco IOS-XE running configuration for security issues.

Device: {hostname}
Audit Date: {datetime.now().strftime('%Y-%m-%d')}

Running Configuration:
```
{config_snippet}
```

{SECURITY_CHECKS}

Return this exact JSON (no other text):
{{
  "device": "{hostname}",
  "audit_date": "{datetime.now().strftime('%Y-%m-%d')}",
  "overall_risk_score": 0,
  "risk_level": "CRITICAL|HIGH|MEDIUM|LOW",
  "pass_count": 0,
  "fail_count": 0,
  "warning_count": 0,
  "checks": [
    {{
      "id": 1,
      "category": "AUTHENTICATION",
      "check": "check name",
      "result": "PASS|FAIL|WARNING|NOT_APPLICABLE",
      "finding": "what was found in the config",
      "recommendation": "specific IOS-XE command to fix this",
      "severity": "CRITICAL|HIGH|MEDIUM|LOW|INFO"
    }}
  ]
}}"""

    response = client.messages.create(
        model="claude-opus-4-6", max_tokens=4096,
        system=AUDITOR_SYSTEM,
        messages=[{"role": "user", "content": prompt}]
    )

    raw = response.content[0].text.strip()
    raw = re.sub(r"```[a-z]*\n?", "", raw).strip()
    try:
        return json.loads(raw)
    except json.JSONDecodeError as e:
        return {"error": str(e), "raw": raw, "checks": []}


def display_audit_report(result):
    """Print formatted audit report."""
    if "error" in result:
        print(f"Audit error: {result['error']}")
        return

    device   = result.get("device", "Unknown")
    score    = result.get("overall_risk_score", 0)
    risk     = result.get("risk_level", "?")
    checks   = result.get("checks", [])
    fails    = [c for c in checks if c.get("result") == "FAIL"]
    warnings = [c for c in checks if c.get("result") == "WARNING"]
    passes   = [c for c in checks if c.get("result") == "PASS"]

    print(f"\n{'━'*65}")
    print(f"  SECURITY AUDIT REPORT: {device}")
    print(f"  Date: {result.get('audit_date', 'Unknown')}")
    print(f"{'━'*65}")
    print(f"  Risk Score: {score}/100  |  Level: {risk}")
    print(f"  ✅ {len(passes)} Pass  |  ❌ {len(fails)} Fail  |  ⚠️  {len(warnings)} Warning")
    print(f"{'━'*65}")

    if fails:
        print(f"\n  ❌ FAILURES — Fix Immediately")
        print(f"  {'─'*62}")
        for f in fails:
            sev = f.get("severity", "?")
            print(f"\n  [{sev}] {f.get('check', '?')}")
            print(f"    Finding: {f.get('finding', '?')}")
            print(f"    Fix:     {f.get('recommendation', '?')}")

    if warnings:
        print(f"\n  ⚠️  WARNINGS — Review Soon")
        print(f"  {'─'*62}")
        for w in warnings:
            print(f"\n  {w.get('check', '?')}: {w.get('finding', '?')}")
            print(f"    Recommendation: {w.get('recommendation', '?')}")


def save_report(result):
    """Save audit result to JSON file."""
    Path("audit_reports").mkdir(exist_ok=True)
    ts = datetime.now().strftime("%Y%m%d_%H%M%S")
    device = result.get("device", "unknown")
    filename = f"audit_reports/{device}_{ts}_audit.json"
    Path(filename).write_text(json.dumps(result, indent=2))
    print(f"\nReport saved: {filename}")


def main():
    print("╔══════════════════════════════════════════╗")
    print("║   AI Security Configuration Auditor     ║")
    print("╚══════════════════════════════════════════╝")

    if "--config" in sys.argv:
        idx = sys.argv.index("--config")
        filepath = sys.argv[idx+1] if idx+1 < len(sys.argv) else input("Config file: ")
        path = Path(filepath)
        hostname, config = path.stem, path.read_text()
    elif "--device" in sys.argv:
        idx = sys.argv.index("--device")
        ip  = sys.argv[idx+1] if idx+1 < len(sys.argv) else input("Device IP: ")
        u   = os.getenv("NETMIKO_USERNAME") or input("Username: ")
        p   = os.getenv("NETMIKO_PASSWORD") or getpass("Password: ")
        hostname, config = collect_config_from_device(ip, u, p)
    else:
        print("Usage:\n  python ai_config_auditor.py --device <ip>")
        print("  python ai_config_auditor.py --config <file>")
        return

    if not config:
        return

    print(f"\nAuditing: {hostname}")
    result = audit_config_with_ai(hostname, config)
    display_audit_report(result)
    save_report(result)


if __name__ == "__main__":
    main()
```

---

*Previous: [Chapter 13](ch13_ai_troubleshooter.md) | Next: [Chapter 15 — The Digital CX Coworker](ch15_digital_coworker.md)*

---
