# Chapter 12 — LLM APIs and the Art of Prompt Engineering

> *"AI is neither magic nor threat. It is a tool — the most powerful tool we have ever built for working with language and knowledge."*

---

## Chapter Goal

Introduce Large Language Model APIs and establish the three-level prompt engineering framework used throughout Section 3. By the end, you will make your first API call, understand how to structure prompts for consistent results, get structured JSON back from an AI, and maintain multi-turn conversations with memory.

**Key Points:**
- What LLMs are and why they matter for network automation
- Setting up the Anthropic API (Claude)
- Three levels of prompt engineering: R-T-F → Structured → RAG + CoT
- Getting structured JSON output from AI (programmatic integration)
- Multi-turn conversation: how AI memory works in code

---

## What LLMs Change for Network Automation

Everything in Sections 1 and 2 was **deterministic** — the same input always produces the same output, and you had to write a rule for every possible scenario.

LLMs add a **reasoning layer**. Instead of writing regex to parse syslog messages — spending weeks building rules for every possible format — you describe the problem in English and the AI figures it out.

| Traditional Automation | AI-Augmented Automation |
|-----------------------|------------------------|
| You write rules for every case | AI reasons about cases you did not anticipate |
| Fails on unexpected input | Handles novel situations with context |
| Requires structured data | Works with raw text, logs, configs |
| You define the logic | You describe the intent |

This is not replacing your judgment — it is amplifying it. The AI applies your expertise (embedded in the prompt) at machine speed to every device in your fleet.

---

## Setup: Anthropic API

```bash
pip install anthropic
```

Set your API key as an environment variable (never in code):

```bash
# Linux / Mac
export ANTHROPIC_API_KEY="sk-ant-..."

# Windows PowerShell
$env:ANTHROPIC_API_KEY = "sk-ant-..."
```

Get your API key from [console.anthropic.com](https://console.anthropic.com).

---

## Your First API Call

```python
# 01_hello_ai.py
import anthropic

# Client automatically reads ANTHROPIC_API_KEY from environment
client = anthropic.Anthropic()

message = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "What is OSPF and why do network engineers care about it?"}
    ]
)

print(message.content[0].text)
```

### The API Response Object

```python
# message is an anthropic.types.Message object
print(type(message))
# <class 'anthropic.types.message.Message'>

# Extract the text
print(message.content[0].text)

# Other useful fields
print(message.model)               # claude-opus-4-6
print(message.usage.input_tokens)  # tokens in your prompt
print(message.usage.output_tokens) # tokens in the response
```

### API Parameters

| Parameter | Description | Typical Values |
|-----------|-------------|---------------|
| `model` | Which AI model to use | `"claude-opus-4-6"` |
| `max_tokens` | Max response length (1 token ≈ 0.75 words) | 1024, 2048, 4096 |
| `messages` | Conversation history | List of `{"role", "content"}` dicts |
| `system` | Stable persona/instructions | String |
| `temperature` | Randomness (0=deterministic, 1=creative) | 0.0–1.0 |

---

## Three Levels of Prompt Engineering

Prompts are the instructions you give the AI. Better prompts produce more consistent, accurate, and useful responses. These three levels build progressively.

---

### Level 1 — Role-Task-Format (R-T-F)

The simplest effective structure. Tell the AI who it is, what to do, and how to format the answer.

```python
# 02_level1_rtf.py
import anthropic

client = anthropic.Anthropic()

syslog = "%LINK-3-UPDOWN: Interface GigabitEthernet0/1, changed state to down"

prompt = f"""
Role: You are a Cisco CCNP-certified network engineer with 15 years 
      of enterprise troubleshooting experience.

Task: Analyze this Cisco syslog message and identify the problem 
      and likely causes.

Format: Respond using EXACTLY this structure:
  SEVERITY: [P1/P2/P3/P4]
  SUMMARY: [one sentence describing what happened]
  LIKELY CAUSES:
    - [cause 1]
    - [cause 2]
    - [cause 3]
  IMMEDIATE ACTIONS:
    1. [first action]
    2. [second action]
  VERIFY WITH:
    - [show command 1]
    - [show command 2]

Syslog message to analyze:
{syslog}
"""

response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": prompt}]
)

print(response.content[0].text)
```

**What R-T-F gives you:**
- Consistent structure — every analysis looks the same
- Appropriate expertise level — "CCNP-certified" grounds the response
- Specific format — you can parse the output reliably

---

### Level 2 — Structured Prompts with Context Injection

Level 2 separates the AI's stable **persona** (system prompt) from the **variable context** (device data) using the `system` parameter. Live device data collected by your Netmiko scripts is injected into the user message.

```python
# 03_level2_structured.py
import anthropic

client = anthropic.Anthropic()

# This data comes from your Netmiko automation
device_context = {
    "hostname":        "sw-core-01",
    "model":           "C9300-48P",
    "ios_version":     "17.9.4a",
    "uptime":          "3 days, 4 hours",
    "cpu_5min":        "78%",
    "free_memory_mb":  "512",
    "down_interfaces": ["GigabitEthernet0/1", "GigabitEthernet0/7"],
    "recent_syslogs":  [
        "%LINK-3-UPDOWN: Interface GigabitEthernet0/1, changed state to down",
        "%BGP-5-ADJCHANGE: neighbor 10.0.0.2 Down Hold Timer Expired",
        "%SYS-2-MALLOCFAIL: Memory allocation of 65536 bytes failed",
    ],
}

# System prompt: stable across all calls — defines the AI's identity
system_prompt = """You are a Cisco CX Senior Network Engineer performing 
device health assessments. You have expert-level knowledge of IOS-XE, 
Catalyst 9000, BGP, OSPF, and enterprise network troubleshooting.

Always structure your response with these exact sections:
## Health Score: X/10
[one line justification]

## Critical Issues (P1/P2)
[list only — skip this section if none]

## Warnings (P3)
[list only — skip this section if none]

## Root Cause Analysis
[the most likely underlying problem]

## Recommended Actions
1. [highest priority first]
2. [next action]

## Show Commands to Run
- `command` — what it reveals"""

# User message: device-specific, changes per device
user_message = f"""
Perform a complete health assessment for this device.

**Device:** {device_context['hostname']} | {device_context['model']}
**IOS-XE:** {device_context['ios_version']}
**Uptime:** {device_context['uptime']}
**CPU (5min):** {device_context['cpu_5min']}
**Free Memory:** {device_context['free_memory_mb']} MB

**Down Interfaces:** {', '.join(device_context['down_interfaces'])}

**Recent Syslogs:**
{chr(10).join('  ' + log for log in device_context['recent_syslogs'])}
"""

response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=2048,
    system=system_prompt,
    messages=[{"role": "user", "content": user_message}]
)

print(response.content[0].text)
```

**What Level 2 adds:**
- The `system` parameter maintains the persona across the entire session
- Real device data makes the analysis specific, not generic
- The AI now has CPU, memory, interfaces, AND syslogs — it can connect them

---

### Level 3 — RAG + Chain-of-Thought Reasoning

**RAG** (Retrieval Augmented Generation): inject relevant documents — Cisco EoL tables, PSIRT advisories, security hardening guides — directly into the prompt. The AI analyzes against *your knowledge base*, not a generic one.

**CoT** (Chain of Thought): instruct the AI to reason step-by-step before answering. This produces dramatically more accurate analysis for complex, multi-factor assessments.

```python
# 04_level3_rag_cot.py
import anthropic
from pathlib import Path

client = anthropic.Anthropic()

# RAG: load relevant knowledge from local files
def load_knowledge(topic):
    path = Path(f"knowledge_base/{topic}.txt")
    return path.read_text() if path.exists() else ""

eol_knowledge      = load_knowledge("cisco_eol_platforms")
security_standards = load_knowledge("cisco_security_hardening")

# Device under assessment
device = {
    "hostname":   "sw-access-01",
    "platform":   "WS-C3650-24TS",
    "ios_version": "03.07.04E",
    "config_snippet": """
aaa new-model
username admin privilege 15 password 0 cisco123
snmp-server community public RO
no service password-encryption
line vty 0 4
 transport input all
"""
}

prompt = f"""
You are assessing a network device for lifecycle and security risk.

=== REFERENCE: CISCO EOL DATA ===
{eol_knowledge[:2000] if eol_knowledge else "Use your training knowledge for Cisco EoL timelines."}

=== REFERENCE: SECURITY STANDARDS ===
{security_standards[:1500] if security_standards else "Use your training knowledge for IOS-XE hardening."}

=== DEVICE UNDER ASSESSMENT ===
Hostname:    {device['hostname']}
Platform:    {device['platform']}
IOS Version: {device['ios_version']}

Config excerpt:
{device['config_snippet']}

=== CHAIN-OF-THOUGHT INSTRUCTIONS ===
Work through these steps BEFORE writing your final answer:

STEP 1 — LIFECYCLE: Is {device['platform']} currently EoL? 
         What specific dates apply (EoSWM, EoSS, LDoS)?

STEP 2 — IOS VERSION: Is {device['ios_version']} still receiving 
         security patches? Any known CVEs?

STEP 3 — SECURITY AUDIT: Review each config line.
         What is insecure? What is missing?

STEP 4 — RISK SCORE: Based on steps 1-3, assign a risk score 1-10.

STEP 5 — ACTION PLAN: List top 5 actions in priority order.

Format your response as:
## Lifecycle Assessment
[findings from step 1]

## IOS Version Assessment  
[findings from step 2]

## Security Findings
[table: Check | Status | Finding | Remediation Command]

## Overall Risk Score: X/10
[justification]

## Priority Action Plan
1. [P1 — Critical]
2. [P2 — High]
...
"""

response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=4096,
    messages=[{"role": "user", "content": prompt}]
)

print(response.content[0].text)
```

> 🔑 **RAG is what turns a general AI into a specialist in your environment.** When you inject your actual EoL tables, your security policies, and your change management procedures into the prompt, the AI analyzes against your reality — not a generic one.

---

## Getting Structured JSON Output

For programmatic processing — feeding AI output into dashboards, databases, or alerting systems — ask for JSON:

```python
# 05_json_output.py
import anthropic
import json
import re

client = anthropic.Anthropic()

syslog_messages = [
    "%LINK-3-UPDOWN: Interface GigabitEthernet0/1, changed state to down",
    "%BGP-5-ADJCHANGE: neighbor 10.0.0.2 Down Hold Timer Expired",
    "%SYS-5-CONFIG_I: Configured from console by admin on vty0 (10.10.1.100)",
    "%SEC-6-IPACCESSLOGP: list MGMT_ACL denied tcp 192.168.99.1(52341) -> 10.10.1.1(22)",
]

prompt = f"""Analyze these Cisco syslog messages.

SYSLOG MESSAGES:
{chr(10).join(f'{i+1}. {msg}' for i, msg in enumerate(syslog_messages))}

Return a JSON array — NOTHING ELSE. No markdown, no explanation:
[
  {{
    "syslog": "original message",
    "severity": "P1|P2|P3|P4",
    "category": "link_event|routing|security|config_change|other",
    "summary": "one-sentence plain English explanation",
    "action_required": true|false,
    "suggested_command": "most useful verification show command"
  }}
]
"""

response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": prompt}]
)

raw = response.content[0].text.strip()

# Clean any accidental markdown fences
raw = re.sub(r'```[a-z]*\n?', '', raw).strip()

try:
    events = json.loads(raw)
    print(f"Parsed {len(events)} events:\n")
    for event in events:
        action = "⚠️  ACTION NEEDED" if event['action_required'] else "ℹ️  informational"
        print(f"[{event['severity']}] {event['category'].upper()}")
        print(f"  Summary:  {event['summary']}")
        print(f"  Status:   {action}")
        print(f"  Verify:   {event['suggested_command']}")
        print()
except json.JSONDecodeError as e:
    print(f"JSON parse error: {e}")
    print(f"Raw response: {raw}")
```

> 💡 **Tip:** Tell the AI explicitly: "Return ONLY valid JSON. No markdown. No explanation outside the JSON." Add a JSON validation wrapper in your code — `try: json.loads(raw) except json.JSONDecodeError:` — as a fallback.

---

## Multi-Turn Conversation: How AI Memory Works

By default, each API call is stateless — the AI has no memory of previous calls. You create memory by passing the full conversation history with every request:

```python
# 06_multi_turn.py
import anthropic

client = anthropic.Anthropic()

system = """You are a Cisco network expert helping diagnose a network issue.
Be concise but thorough. Ask clarifying questions when needed."""

# Conversation history — grows with every exchange
history = []

def chat(user_message):
    """Send a message and maintain full conversation history."""
    history.append({"role": "user", "content": user_message})

    response = client.messages.create(
        model="claude-opus-4-6",
        max_tokens=1024,
        system=system,
        messages=history   # ← Full history every time
    )

    reply = response.content[0].text
    history.append({"role": "assistant", "content": reply})
    return reply


# A realistic troubleshooting conversation
print("AI:", chat("My BGP neighbor on sw-core-01 just dropped"))
print()
print("AI:", chat("The neighbor IP is 10.0.0.2, we're AS 65001, they're AS 65002"))
print()
print("AI:", chat("The syslog shows: %BGP-5-ADJCHANGE: neighbor 10.0.0.2 Down Hold Timer Expired"))
print()
print("AI:", chat("What show commands should I run to diagnose this?"))
```

The AI in the last call has context from all three previous messages — it knows the neighbor IP, the AS numbers, and the exact syslog message. This context-awareness is what separates a chatbot from a coworker.

---

## Prompt Engineering Reference

| Level | Structure | Best For | When to Use |
|-------|-----------|----------|-------------|
| 1 — R-T-F | Role + Task + Format | Single-question analysis | Quick syslog lookups, one-off queries |
| 2 — Structured | System + context injection | Device health reports | Regular assessments with device data |
| 3 — RAG + CoT | Knowledge + step-by-step | Complex multi-factor analysis | Lifecycle, security, compliance |

### Tips for Network AI Prompts

- **Be specific:** "Cisco IOS-XE 17.9.4 on Catalyst 9300" beats "Cisco switch"
- **Inject real data:** The more device context you provide, the more accurate the analysis
- **Specify the format:** Always describe how you want output structured
- **Use CoT for complex decisions:** "Think step by step before answering"
- **Ask for JSON when processing programmatically:** Cleaner than parsing free text
- **Keep system prompts stable:** Define the persona once, vary the user message

---

## Practice Exercises

1. Write a Level 1 R-T-F prompt to analyze this BGP syslog:  
   `%BGP-5-ADJCHANGE: neighbor 192.168.1.1 Down Notification sent`

2. Modify the Level 2 script to add `show ip ospf neighbor` output to the device context. Test it.

3. Build a 3-turn conversation where you: (a) describe a network problem, (b) provide show command output, (c) ask for a specific fix.

4. Write a prompt that returns JSON with a list of security issues from the config snippet in the Level 3 example.

---

*Previous: [Chapter 11](../section2/ch11_git_cicd.md) | Next: [Chapter 13 — The AI Network Troubleshooter](ch13_ai_troubleshooter.md)*
