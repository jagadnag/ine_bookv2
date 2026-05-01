# Chapter 16 — LangChain and LangGraph: Building Multi-Agent Network Automation

> *"The goal is not to build AI that replaces engineers. The goal is to build AI systems sophisticated enough that engineers can stop doing things machines should be doing."*

---

## Chapter Goal

Take everything from Chapters 12–15 and wire it together with **LangChain** and **LangGraph** — the frameworks that turn individual AI interactions into structured, orchestrated, multi-agent workflows. You will use **OpenAI's GPT-4o** as the primary model, learn to swap to **local Ollama** (Llama 3.2, Gemma 4) for private or zero-cost operation, and understand exactly when each approach is right.

**Key Points:**
- What LangChain adds over raw API calls — and when you need it
- The ReAct agent pattern: tool-calling, memory, and the reasoning loop
- A weather agent that teaches the pattern before touching network devices
- A Netmiko agent where AI decides which show commands to run
- Local Ollama for private config analysis — free, no data leaves your network
- LangGraph: the state machine for multi-agent coordination
- A complete 3-agent system: Inventory → Backup → AI Health Report

---

## Why LangChain? The Gap Raw API Calls Leave

```
RAW API CALLS                    │  LANGCHAIN FRAMEWORK
(Chapters 12–15)                 │  (This chapter)
─────────────────────────────────┼───────────────────────────────────
You manage history[] manually    │  Memory handled automatically
You write tool-calling glue code │  @tool decorator + AgentExecutor
You write the agent loop         │  ReAct loop built in
No structured state              │  TypedDict flows through graph
Model swap = rewrite imports     │  One line: ChatOpenAI → ChatOllama
Hard to compose multiple agents  │  LangGraph orchestrates N agents
```

The raw API approach from Chapters 12–15 is still the right choice for single-purpose tools — the syslog analyzer, the config auditor. LangChain earns its place when:
- The AI needs to **decide** which tools to call (not just call one tool)
- You need **conversation memory** across multiple turns
- Multiple **specialized agents** need to coordinate

---

## Setup

```bash
pip install langchain langchain-openai langchain-anthropic langchain-community langgraph
pip install netmiko tabulate requests

# Cloud APIs
export OPENAI_API_KEY="sk-..."            # OpenAI — used in all main examples
export ANTHROPIC_API_KEY="sk-ant-..."    # Anthropic — swap in with one line

# Local Ollama (free, private, no API key)
# Linux/Mac install:
curl -fsSL https://ollama.ai/install.sh | sh

ollama pull llama3.2      # 2.0 GB — fast, good for structured tasks
ollama pull gemma3:4b     # 3.3 GB — strong reasoning, Google's model

# Verify
ollama run llama3.2 "What is OSPF?"
```

---

## The Model Swap Pattern

One of LangChain's most valuable features: **identical code, any backend**.

```python
from langchain_openai     import ChatOpenAI
from langchain_anthropic  import ChatAnthropic
from langchain_community.chat_models import ChatOllama

# Pick ONE — everything else in your code stays identical
llm = ChatOpenAI(model="gpt-4o",          temperature=0, max_tokens=3000)
# llm = ChatAnthropic(model="claude-opus-4-6",  max_tokens=3000)
# llm = ChatOllama(model="llama3.2",            temperature=0)
# llm = ChatOllama(model="gemma3:4b",           temperature=0)
```

Use this pattern throughout the chapter. Every example shows you where to swap.

---

## Part 1: The ReAct Agent Pattern — Weather First

Before connecting to network devices, learn the LangChain agent pattern with a simple, testable example: a weather agent. No VPN, no lab devices, no credentials needed.

### The ReAct Loop

ReAct stands for **Re**asoning + **Act**ing. The agent loops until it has enough information to answer:

```
User question arrives
       │
       ▼
   OBSERVE: Read question + conversation history + available tools
       │
       ▼
   THINK: Which tool do I need? What arguments?
       │
       ├── Need more data → ACT: Call tool → Get result → OBSERVE again
       │
       └── Have enough → FINAL ANSWER: Synthesize and respond
```

### Weather Agent (GPT-4o)

```python
#!/usr/bin/env python3
# 07_langchain_weather_agent.py
"""
LangChain fundamentals: tool-use pattern with GPT-4o.
Demonstrates @tool, AgentExecutor, memory, and the ReAct loop.
Change model to ChatOllama("llama3.2") to run locally — free.
"""
import os, requests
from langchain_openai import ChatOpenAI
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain.tools import tool
from langchain.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain.memory import ConversationBufferMemory


# ── Tools ─────────────────────────────────────────────────────
# The @tool decorator + docstring is what the AI reads to decide
# WHEN and HOW to call this function. Write docstrings carefully.

@tool
def get_weather(city: str) -> str:
    """
    Get current weather conditions for a city.
    Use when user asks about weather, temperature, or conditions.

    Args:
        city: City name (e.g. 'London', 'Tokyo', 'Sydney')
    Returns:
        Formatted current weather data string
    """
    try:
        resp = requests.get(f"https://wttr.in/{city}?format=j1", timeout=10)
        c = resp.json()["current_condition"][0]
        return (
            f"City: {city}\n"
            f"Temperature: {c['temp_C']}°C ({c['temp_F']}°F)\n"
            f"Condition: {c['weatherDesc'][0]['value']}\n"
            f"Humidity: {c['humidity']}%\n"
            f"Wind: {c['windspeedKmph']} km/h {c['winddir16Point']}"
        )
    except Exception as e:
        return f"Weather unavailable for {city}: {e}"


@tool
def convert_temperature(value: float, from_unit: str, to_unit: str) -> str:
    """
    Convert temperature between Celsius (C), Fahrenheit (F), Kelvin (K).

    Args:
        value: Numeric temperature to convert
        from_unit: Source unit — 'C', 'F', or 'K'
        to_unit: Target unit — 'C', 'F', or 'K'
    """
    fu, tu = from_unit.upper(), to_unit.upper()
    to_c   = {"C": lambda v: v, "F": lambda v: (v-32)*5/9, "K": lambda v: v-273.15}
    from_c = {"C": lambda v: v, "F": lambda v: v*9/5+32,   "K": lambda v: v+273.15}
    if fu not in to_c or tu not in from_c:
        return "Units must be C, F, or K"
    result = from_c[tu](to_c[fu](value))
    return f"{value}°{fu} = {result:.1f}°{tu}"


# ── Build Agent ───────────────────────────────────────────────
def build_weather_agent(provider="openai"):
    """Same agent, any model provider — swap with one argument."""
    if provider == "openai":
        llm = ChatOpenAI(model="gpt-4o", temperature=0, max_tokens=1024)
    elif provider == "anthropic":
        from langchain_anthropic import ChatAnthropic
        llm = ChatAnthropic(model="claude-opus-4-6", max_tokens=1024)
    elif provider == "ollama":
        from langchain_community.chat_models import ChatOllama
        llm = ChatOllama(model="llama3.2", temperature=0)

    tools  = [get_weather, convert_temperature]
    prompt = ChatPromptTemplate.from_messages([
        ("system", "You are a weather assistant. Always use get_weather for real data."),
        MessagesPlaceholder(variable_name="chat_history"),
        ("human", "{input}"),
        MessagesPlaceholder(variable_name="agent_scratchpad"),
    ])
    agent  = create_tool_calling_agent(llm, tools, prompt)
    memory = ConversationBufferMemory(memory_key="chat_history", return_messages=True)

    return AgentExecutor(
        agent=agent, tools=tools, memory=memory,
        verbose=True,     # Shows the ReAct reasoning loop
        max_iterations=5, handle_parsing_errors=True,
    )


if __name__ == "__main__":
    agent = build_weather_agent("openai")   # Change to "ollama" to run free + local

    demos = [
        "What is the weather in London right now?",
        "Compare Tokyo and Dubai — which is hotter?",
        "Convert London's temperature to Fahrenheit",   # Uses memory from query 1
    ]
    for q in demos:
        print(f"\nQ: {q}")
        result = agent.invoke({"input": q})
        print(f"A: {result['output']}")
```

**Run it and watch `verbose=True`** — you will see the agent's internal reasoning:
```
> Entering new AgentExecutor chain...
Invoking: `get_weather` with `{'city': 'London'}`
City: London, Temperature: 14°C, Condition: Overcast...
I now have the data I need to answer.
> Finished chain.
```

This is the ReAct loop made visible. Tool called, result observed, reasoning updated, answer generated.

---

## Part 2: The Netmiko LangChain Agent (GPT-4o)

Same architecture, real network tools. The agent decides which devices to query and which commands to run based on your question.

```python
#!/usr/bin/env python3
# 08_langchain_netmiko_agent.py
"""
LangChain agent with Netmiko tools — GPT-4o.
AI decides which show commands to run based on natural language questions.

Ask:
  "What devices do you have?"
  "Check interface status on sw-core-01"
  "Are there BGP issues on all devices?"
  "Get IOS version on sw-core-01 — is 17.9.4a still supported?"
"""
import os, json
from langchain_openai import ChatOpenAI
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain.tools import tool
from langchain.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain.memory import ConversationBufferMemory
from netmiko import ConnectHandler
from netmiko.exceptions import NetmikoAuthenticationException, NetmikoTimeoutException

USERNAME = os.getenv("NETMIKO_USERNAME", "cisco")
PASSWORD = os.getenv("NETMIKO_PASSWORD", "cisco")

INVENTORY = {
    "sw-core-01": {"device_type": "cisco_ios", "ip": "198.18.1.11",
                   "username": USERNAME, "password": PASSWORD},
    "sw-core-02": {"device_type": "cisco_ios", "ip": "198.18.1.12",
                   "username": USERNAME, "password": PASSWORD},
}


def _ssh_run(device_name: str, command: str) -> str:
    """Helper: SSH to device and run one show command."""
    if not command.strip().lower().startswith("show"):
        return f"[BLOCKED] Only show commands allowed. Got: '{command}'"
    device = INVENTORY.get(device_name)
    if not device:
        return f"[ERROR] '{device_name}' not found. Available: {list(INVENTORY.keys())}"
    try:
        with ConnectHandler(**device) as conn:
            return conn.send_command(command, read_timeout=30)
    except NetmikoAuthenticationException:
        return f"[ERROR] Auth failed: {device_name}"
    except NetmikoTimeoutException:
        return f"[ERROR] Timeout: {device_name}"
    except Exception as e:
        return f"[ERROR] {device_name}: {e}"


@tool
def list_devices() -> str:
    """List all network devices available in the inventory with their IPs."""
    return json.dumps([{"name": n, "ip": p["ip"]} for n, p in INVENTORY.items()])


@tool
def run_show_command(device_name: str, command: str) -> str:
    """
    Run a single show command on one specific network device.
    Only 'show' commands are permitted — never configuration commands.

    Args:
        device_name: Device name from inventory (e.g. 'sw-core-01')
        command: IOS show command (e.g. 'show ip interface brief')
    """
    return f"Device: {device_name}\nCommand: {command}\n\n{_ssh_run(device_name, command)}"


@tool
def run_on_all_devices(command: str) -> str:
    """
    Run a show command on ALL devices in the inventory simultaneously.
    Use when the user asks about 'all devices', 'every switch', or 'the whole network'.

    Args:
        command: Show command to run everywhere
    """
    if not command.strip().lower().startswith("show"):
        return f"[BLOCKED] Only show commands: '{command}'"
    sections = [f"{'='*40}\n{name}\n{'='*40}\n{_ssh_run(name, command)}"
                for name in INVENTORY]
    return "\n\n".join(sections)


@tool
def get_device_facts(device_name: str) -> str:
    """
    Collect comprehensive device facts in one SSH session.
    Use for device health assessment or version/platform questions.

    Args:
        device_name: Device name from inventory
    """
    device = INVENTORY.get(device_name)
    if not device:
        return f"Device '{device_name}' not found"
    CMDS = {"version": "show version", "interfaces": "show ip interface brief",
             "bgp": "show ip bgp summary", "ospf": "show ip ospf neighbor"}
    facts = {"device": device_name}
    try:
        with ConnectHandler(**device) as conn:
            facts["hostname"] = conn.find_prompt().rstrip("#>").strip()
            for k, cmd in CMDS.items():
                try:
                    facts[k] = conn.send_command(cmd, read_timeout=20)[:600]
                except Exception as e:
                    facts[k] = f"ERROR: {e}"
        facts["status"] = "success"
    except Exception as e:
        facts.update({"status": "failed", "error": str(e)})
    return json.dumps(facts, indent=2)


def build_network_agent(verbose=True):
    # Swap this one line to change model
    llm = ChatOpenAI(model="gpt-4o", temperature=0, max_tokens=3000)
    # llm = ChatAnthropic(model="claude-opus-4-6", max_tokens=3000)
    # llm = ChatOllama(model="llama3.2")

    tools  = [list_devices, run_show_command, run_on_all_devices, get_device_facts]
    prompt = ChatPromptTemplate.from_messages([
        ("system", """You are an expert Cisco network engineer with SSH access to devices.
- Call list_devices first if unsure which devices exist
- Use run_show_command for one device, run_on_all_devices for the fleet
- Use get_device_facts for health assessment questions
- Only show commands — configuration changes are not permitted
- Be specific: cite actual output lines in your analysis"""),
        MessagesPlaceholder(variable_name="chat_history"),
        ("human", "{input}"),
        MessagesPlaceholder(variable_name="agent_scratchpad"),
    ])
    agent  = create_tool_calling_agent(llm, tools, prompt)
    memory = ConversationBufferMemory(memory_key="chat_history", return_messages=True)
    return AgentExecutor(
        agent=agent, tools=tools, memory=memory,
        verbose=verbose, max_iterations=8, handle_parsing_errors=True,
    )


if __name__ == "__main__":
    print("Network Agent (GPT-4o + Netmiko) — type 'exit' to quit\n")
    agent = build_network_agent(verbose=True)
    while True:
        try:
            q = input("You: ").strip()
        except (KeyboardInterrupt, EOFError):
            break
        if not q or q.lower() == "exit":
            break
        result = agent.invoke({"input": q})
        print(f"\n🤖 {result['output']}\n")
```

---

## Part 3: Local Ollama — Free, Private, Offline

Running models locally means: **no API cost, no data leaving your network, no rate limits.** This is the right approach for:
- Processing sensitive device configurations
- Development and testing (no cost)
- Air-gapped environments
- High-volume batch report generation

### Generating Reports with Llama 3.2 and Gemma 4

```python
#!/usr/bin/env python3
# ollama_report_generator.py
"""
Generate network health reports using local Ollama models.
Zero cost. Data never leaves your machine.

Requirements:
  ollama pull llama3.2
  ollama pull gemma3:4b
  pip install langchain langchain-community ollama
"""
from langchain_community.chat_models import ChatOllama
from langchain_core.messages import HumanMessage, SystemMessage
from datetime import datetime
from pathlib import Path
import time


REPORT_SYSTEM = """You are a network operations analyst. Generate concise,
accurate Markdown health reports. Be specific about issues found."""


def generate_report(device_data: dict, model: str) -> tuple[str, float]:
    """Generate a health report with the specified Ollama model. Returns (report, seconds)."""
    llm = ChatOllama(model=model, temperature=0, num_predict=1500)

    facts_text = ""
    for name, facts in device_data.items():
        facts_text += f"\n### {name} ({facts.get('ip', '?')})\n"
        facts_text += f"Status: {facts.get('status', '?')}\n"
        if facts.get("interfaces"):
            facts_text += f"Interfaces:\n{facts['interfaces'][:400]}\n"
        if facts.get("bgp"):
            facts_text += f"BGP:\n{facts['bgp'][:200]}\n"

    prompt = f"""Generate a network health report for these devices:

{facts_text}

Format as:
# Network Health Report — {model}
**Date:** {datetime.now().strftime('%Y-%m-%d %H:%M')}

## Overall Status: [HEALTHY / DEGRADED / CRITICAL]

## Summary
[2 sentences on network state]

## Issues Found
[Specific issues or "None detected"]

## Actions Required
1. [Specific action or "None"]
"""
    start = time.time()
    messages = [SystemMessage(content=REPORT_SYSTEM), HumanMessage(content=prompt)]
    response = llm.invoke(messages)
    elapsed  = time.time() - start
    return response.content, elapsed


def compare_models(device_data: dict):
    """Run same data through Llama 3.2 and Gemma 4 — compare output and speed."""
    models = [
        ("llama3.2",  "Meta Llama 3.2  (2.0 GB — fast, general purpose)"),
        ("gemma3:4b", "Google Gemma 3 4B (3.3 GB — stronger reasoning)"),
    ]

    results = {}
    for model_name, description in models:
        print(f"\n{'─'*55}")
        print(f"Running: {description}")
        print("─" * 55)
        try:
            report, seconds = generate_report(device_data, model_name)
            results[model_name] = {"report": report, "seconds": seconds}
            print(report)
            print(f"\n⏱  Generation time: {seconds:.1f}s")

            fname = f"report_{model_name.replace(':', '_')}_{datetime.now().strftime('%Y%m%d_%H%M')}.md"
            Path(fname).write_text(report)
            print(f"💾  Saved: {fname}")
        except Exception as e:
            print(f"❌  Failed: {e}")
            print(f"   Make sure Ollama is running: ollama serve")
            print(f"   And model is pulled: ollama pull {model_name}")

    # Speed comparison
    if len(results) == 2:
        t1 = results.get("llama3.2",  {}).get("seconds", 0)
        t2 = results.get("gemma3:4b", {}).get("seconds", 0)
        print(f"\n{'─'*55}")
        print(f"Speed comparison:")
        print(f"  llama3.2:  {t1:.1f}s")
        print(f"  gemma3:4b: {t2:.1f}s")
        faster = "llama3.2" if t1 < t2 else "gemma3:4b"
        print(f"  {faster} was faster — evaluate quality to decide which suits your use case")


# Sample device data (replace with real Netmiko output)
SAMPLE_DATA = {
    "sw-core-01": {
        "ip": "198.18.1.11", "status": "collected",
        "interfaces": (
            "Interface              IP-Address      OK? Status   Protocol\n"
            "GigabitEthernet1       198.18.1.11     YES up       up\n"
            "GigabitEthernet2       unassigned      YES down     down\n"
            "Loopback0              10.255.255.1    YES up       up"
        ),
        "bgp": "Neighbor 10.0.0.2 AS 65002 — State: Active (NOT Established!)",
    },
    "sw-core-02": {
        "ip": "198.18.1.12", "status": "collected",
        "interfaces": (
            "Interface              IP-Address      OK? Status   Protocol\n"
            "GigabitEthernet1       198.18.1.12     YES up       up\n"
            "GigabitEthernet2       192.168.1.1     YES up       up\n"
            "Loopback0              10.255.255.2    YES up       up"
        ),
        "bgp": "BGP not configured on this device",
    },
}


if __name__ == "__main__":
    print("Local Ollama Report Generator")
    print("No API keys. No cost. Data stays on your machine.\n")
    compare_models(SAMPLE_DATA)
```

---

## Part 4: LangGraph — Coordinating Multiple Agents

Where LangChain gives you a single agent that loops, LangGraph gives you a **directed graph of agents** where each node is a specialist and edges define how state flows between them.

### The Three-Agent Architecture

```
User: "Run full network health check"
                    │
                    ▼
         ┌──────────────────────┐
         │  SHARED STATE        │
         │  (flows through all  │
         │  agent nodes)        │
         │                      │
         │  device_facts: {}    │
         │  backup_results: {}  │
         │  health_report: ""   │
         └──────────────────────┘
                    │
     ┌──────────────┼──────────────┐
     ▼              ▼              ▼
┌─────────┐   ┌─────────┐   ┌─────────┐
│INVENTORY│──▶│ BACKUP  │──▶│ REPORT  │──▶ END
│  AGENT  │   │  AGENT  │   │  AGENT  │
│         │   │         │   │         │
│show ver │   │show run │   │GPT-4o   │
│int brief│   │Save .cfg│   │analyzes │
│bgp sum  │   │files    │   │all data │
│ospf nbr │   │to disk  │   │→ .md    │
└─────────┘   └─────────┘   └─────────┘
     │              │              │
     └──────────────┴──────────────┘
              Write to shared state
```

### The Complete System (GPT-4o)

```python
#!/usr/bin/env python3
# 09_langgraph_network_ops.py
"""
LangGraph multi-agent network operations — GPT-4o orchestrated.
Three specialized agents share state through a directed graph.

To swap models: edit get_llm() — one line change.
"""
import os, json
from datetime import datetime
from pathlib import Path
from typing import TypedDict, List

from langchain_openai import ChatOpenAI
from langchain_core.messages import BaseMessage, HumanMessage
from langgraph.graph import StateGraph, END
from netmiko import ConnectHandler
from netmiko.exceptions import NetmikoAuthenticationException, NetmikoTimeoutException

USERNAME = os.getenv("NETMIKO_USERNAME", "cisco")
PASSWORD = os.getenv("NETMIKO_PASSWORD", "cisco")

INVENTORY = {
    "sw-core-01": {"device_type": "cisco_ios", "ip": "198.18.1.11",
                   "username": USERNAME, "password": PASSWORD},
    "sw-core-02": {"device_type": "cisco_ios", "ip": "198.18.1.12",
                   "username": USERNAME, "password": PASSWORD},
}


def get_llm():
    """Swap model here — one line change affects every agent."""
    return ChatOpenAI(model="gpt-4o", temperature=0, max_tokens=4096)
    # from langchain_anthropic import ChatAnthropic
    # return ChatAnthropic(model="claude-opus-4-6", max_tokens=4096)
    # from langchain_community.chat_models import ChatOllama
    # return ChatOllama(model="llama3.2", temperature=0)


# ── Shared State ─────────────────────────────────────────────
class NetworkOpsState(TypedDict):
    messages:       List[BaseMessage]
    device_facts:   dict     # Written by: Inventory Agent
    backup_results: dict     # Written by: Backup Agent
    health_report:  str      # Written by: Report Agent
    errors:         List[str]
    task_complete:  bool


# ── Agent 1: Inventory ────────────────────────────────────────
def inventory_agent(state: NetworkOpsState) -> NetworkOpsState:
    print("\n" + "═"*50)
    print("  [INVENTORY AGENT] Collecting device facts")
    print("═"*50)

    COMMANDS = {
        "version":    "show version",
        "interfaces": "show ip interface brief",
        "bgp":        "show ip bgp summary",
        "ospf":       "show ip ospf neighbor",
        "cpu":        "show processes cpu sorted | head 10",
    }
    device_facts = {}

    for name, params in INVENTORY.items():
        print(f"  {name} ({params['ip']})...", end=" ", flush=True)
        facts = {"device": name, "ip": params["ip"],
                 "timestamp": datetime.now().isoformat()}
        try:
            with ConnectHandler(**params) as conn:
                facts["hostname"] = conn.find_prompt().rstrip("#>").strip()
                for key, cmd in COMMANDS.items():
                    try:
                        facts[key] = conn.send_command(cmd, read_timeout=20)[:700]
                    except Exception as e:
                        facts[key] = f"ERROR: {e}"
            facts["status"] = "collected"
            print("✅")
        except NetmikoAuthenticationException:
            facts.update({"status": "auth_failed",  "error": "Auth failure"})
            print("❌ auth")
        except NetmikoTimeoutException:
            facts.update({"status": "timeout",      "error": "Timeout"})
            print("⏱  timeout")
        except Exception as e:
            facts.update({"status": "failed",        "error": str(e)})
            print(f"❌ {e}")
        device_facts[name] = facts

    ok = sum(1 for f in device_facts.values() if f.get("status") == "collected")
    print(f"  Done: {ok}/{len(device_facts)} collected")
    state["device_facts"] = device_facts
    return state


# ── Agent 2: Backup ───────────────────────────────────────────
def backup_agent(state: NetworkOpsState) -> NetworkOpsState:
    print("\n" + "═"*50)
    print("  [BACKUP AGENT] Saving configurations")
    print("═"*50)

    backup_dir = Path("backups")
    backup_dir.mkdir(exist_ok=True)
    ts = datetime.now().strftime("%Y%m%d_%H%M%S")
    backup_results = {}

    for name, params in INVENTORY.items():
        print(f"  {name}...", end=" ", flush=True)
        try:
            with ConnectHandler(**params) as conn:
                hostname = conn.find_prompt().rstrip("#>").strip()
                config   = conn.send_command("show running-config", read_timeout=60)
            filename = backup_dir / f"{hostname}_{ts}.cfg"
            filename.write_text(config)
            backup_results[name] = {
                "status":   "success",
                "hostname": hostname,
                "file":     str(filename),
                "size_kb":  round(len(config) / 1024, 1),
            }
            print(f"✅  → {filename}")
        except Exception as e:
            backup_results[name] = {"status": "error", "error": str(e)}
            print(f"❌ {e}")

    ok = sum(1 for r in backup_results.values() if r.get("status") == "success")
    print(f"  Done: {ok}/{len(backup_results)} backups saved")
    state["backup_results"] = backup_results
    return state


# ── Agent 3: Report ───────────────────────────────────────────
def report_agent(state: NetworkOpsState) -> NetworkOpsState:
    print("\n" + "═"*50)
    print("  [REPORT AGENT] Generating AI health report")
    print("═"*50)

    llm            = get_llm()
    device_facts   = state.get("device_facts", {})
    backup_results = state.get("backup_results", {})

    # Build context for AI
    facts_ctx = ""
    for name, facts in device_facts.items():
        facts_ctx += f"\n\n### {name} ({facts.get('ip','?')}) — Status: {facts.get('status')}\n"
        if facts.get("status") == "collected":
            facts_ctx += f"Hostname: {facts.get('hostname','?')}\n"
            facts_ctx += f"Interfaces:\n{facts.get('interfaces','N/A')[:400]}\n"
            facts_ctx += f"BGP:\n{facts.get('bgp','N/A')[:250]}\n"
            facts_ctx += f"OSPF:\n{facts.get('ospf','N/A')[:200]}\n"
        else:
            facts_ctx += f"Error: {facts.get('error', 'unknown')}\n"

    backup_ctx = "\n".join(
        f"  {'✅' if r.get('status')=='success' else '❌'} {nm}: "
        f"{r.get('file', r.get('error','?'))}"
        for nm, r in backup_results.items()
    )
    ok_backups = sum(1 for r in backup_results.values() if r.get("status") == "success")

    prompt = f"""Generate a professional Network Health Report in Markdown.

Timestamp: {datetime.now().strftime('%Y-%m-%d %H:%M')}
Devices assessed: {len(device_facts)}
Backups completed: {ok_backups}/{len(backup_results)}

DEVICE DATA:
{facts_ctx}

BACKUP STATUS:
{backup_ctx}

Write:
# Network Health Report
**Date:** {datetime.now().strftime('%Y-%m-%d %H:%M')}

## Overall Health Score: X/10
[one sentence based on actual data]

## Executive Summary
[2 sentences]

## Device-by-Device Assessment
[Status + key findings per device — cite actual output]

## Issues Requiring Attention
[P1/P2/P3 with specific details, or "None detected"]

## Backup Summary
[Results table]

## Recommended Actions
1. [Specific, prioritized]

Be specific. Reference real data. No placeholder text."""

    print("  Sending to AI for analysis...")
    response = llm.invoke([HumanMessage(content=prompt)])
    health_report = response.content

    ts          = datetime.now().strftime("%Y%m%d_%H%M%S")
    report_file = Path(f"health_report_{ts}.md")
    report_file.write_text(health_report)
    print(f"  ✅ Saved: {report_file}")

    state["health_report"] = health_report
    state["task_complete"] = True
    return state


# ── Build + Run Graph ─────────────────────────────────────────
def main():
    print("╔══════════════════════════════════════════════════╗")
    print("║  LangGraph Network Ops — GPT-4o Orchestration   ║")
    print("║  Agents: Inventory → Backup → Report → END      ║")
    print("╚══════════════════════════════════════════════════╝\n")

    graph = StateGraph(NetworkOpsState)
    graph.add_node("inventory", inventory_agent)
    graph.add_node("backup",    backup_agent)
    graph.add_node("report",    report_agent)
    graph.set_entry_point("inventory")
    graph.add_edge("inventory", "backup")
    graph.add_edge("backup",    "report")
    graph.add_edge("report",    END)
    ops = graph.compile()

    initial: NetworkOpsState = {
        "messages":       [HumanMessage(content="Run full network health check")],
        "device_facts":   {}, "backup_results": {},
        "health_report":  "", "errors":          [],
        "task_complete":  False,
    }

    start = datetime.now()
    final = ops.invoke(initial)
    elapsed = (datetime.now() - start).total_seconds()

    print(f"\n{'═'*50}")
    print(f"  RUN COMPLETE — {elapsed:.1f}s")
    print(f"  Devices:   {len(final['device_facts'])}")
    ok = sum(1 for r in final['backup_results'].values() if r.get('status')=='success')
    print(f"  Backups:   {ok}/{len(final['backup_results'])}")
    print("\nReport preview:")
    print("─"*50)
    lines = final["health_report"].split("\n")
    print("\n".join(lines[:35]))
    if len(lines) > 35:
        print(f"... [{len(lines)-35} more lines — see saved file]")


if __name__ == "__main__":
    main()
```

---

## When to Use What

| Approach | Best For | Complexity | Model Options |
|----------|----------|------------|---------------|
| Raw API (Ch 12–15) | Single-purpose tools | ⭐ Simple | Any |
| LangChain Agent | Interactive Q&A, tool decisions | ⭐⭐ Medium | GPT-4o / Claude / Ollama |
| LangGraph Multi-Agent | Autonomous multi-step operations | ⭐⭐⭐ Complex | GPT-4o recommended |
| Local Ollama | Private data, zero cost, air-gapped | ⭐ Simple (less capable) | Llama 3.2 / Gemma 4 |

---

## Practice Exercises

1. **Model swap:** Change `build_weather_agent("openai")` to `build_weather_agent("ollama")`. Compare Llama 3.2 vs GPT-4o response quality and speed.

2. **New Netmiko tool:** Add `check_ntp_configured(device_name)` that returns `True/False`. Ask the agent "Is NTP configured on all my devices?"

3. **Ollama for config analysis:** Point `ollama_report_generator.py` at a real backup from Chapter 9. Run both models. Which gives better analysis?

4. **LangGraph conditional route:** Add: if any device `status != "collected"`, route to a `retry_node` that tries one more time before proceeding to backup.

5. **Full pipeline:** Modify the LangGraph Report Agent to save both Markdown AND a JSON summary. The JSON should have `{device: status, issues: [...]}` per device.

---

*Previous: [Chapter 15 — Digital CX Coworker](ch15_digital_coworker.md) | Next: [Chapter 17 — AIOps and the Future](ch17_whats_next.md)*
