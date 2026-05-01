# Chapter 15 — The Digital CX Coworker: Your AI Network Engineering Partner

## Chapter Goal

Build the Digital CX Coworker — a persistent, conversational AI agent that knows your network inventory, remembers conversation context, and can execute live show commands on demand. Add a Streamlit web interface to make it accessible to your entire team.

---

## What Makes a Coworker Different from a Script

A script runs and forgets. A coworker remembers.

- **Script:** "Analyze device X." → Output → Exits. Next run has no memory.
- **Coworker:** "My BGP dropped." → Context builds. "The neighbor is 10.0.0.2." → AI remembers. "It shows Hold Timer Expired." → AI connects all three messages and reasons about the complete picture.

Memory is implemented by passing the full conversation history with every API call.

---

## The Full CX Coworker

```python
#!/usr/bin/env python3
"""
cx_coworker.py

Digital CX Coworker — conversational AI network assistant.

Features:
  - Full session memory (conversation history)
  - Device inventory awareness  
  - Live show command execution via Netmiko
  - Safety: only executes 'show' commands
  - Save session to JSON

Usage:
  python cx_coworker.py               # Interactive CLI
  python cx_coworker.py --save-ui     # Create Streamlit UI file
"""

import anthropic, os, json, sys
from getpass import getpass
from datetime import datetime
from pathlib import Path

try:
    from netmiko import ConnectHandler
    from netmiko.exceptions import NetmikoAuthenticationException, NetmikoTimeoutException
    NETMIKO_AVAILABLE = True
except ImportError:
    NETMIKO_AVAILABLE = False


# ─────────────────────────────────────────────────────────────
# INVENTORY — Update with your devices
# ─────────────────────────────────────────────────────────────

DEFAULT_INVENTORY = {
    "sw-core-01": {
        "device_type": "cisco_ios",
        "ip":          "198.18.1.11",
        "username":    "cisco",
        "password":    "cisco",
        "role":        "Core Switch",
        "model":       "C9300-48P",
    },
    "sw-core-02": {
        "device_type": "cisco_ios",
        "ip":          "198.18.1.12",
        "username":    "cisco",
        "password":    "cisco",
        "role":        "Core Switch",
        "model":       "C9300-48P",
    },
}


# ─────────────────────────────────────────────────────────────
# COWORKER CLASS
# ─────────────────────────────────────────────────────────────

class CXCoworker:
    """Digital CX Coworker — AI network assistant with device access."""

    SYSTEM_TEMPLATE = """You are a Digital CX Coworker — a senior Cisco network engineer 
with 20 years of enterprise network experience. You are a knowledgeable, direct colleague who:

EXPERTISE:
- Cisco IOS-XE, NX-OS, SD-WAN, Catalyst 9000, ASR, ISR, CSR platforms
- BGP, OSPF, EIGRP, STP, VRF, VXLAN, QoS, SD-WAN configuration
- Network automation: Python, Netmiko, Ansible, pyATS, Genie
- Cisco lifecycle: EoL, PSIRT, Field Notices, Smart Licensing
- Troubleshooting methodology and structured problem-solving

DEVICE INVENTORY (devices you can access):
{inventory_summary}

CAPABILITIES:
1. Answer technical networking questions with expert detail
2. Analyze show command output or config snippets the user shares
3. Generate correct, production-ready IOS-XE configuration
4. Create step-by-step troubleshooting runbooks
5. Assess security posture from configuration snippets
6. Execute show commands on inventory devices when asked

COMMAND EXECUTION PROTOCOL:
When asked to run a command, respond with exactly this on its own line:
EXECUTE: <device_name> | <command>

Example: EXECUTE: sw-core-01 | show ip bgp summary

I will execute the command and return the output to you for analysis.
SAFETY: Never request config commands via EXECUTE — show commands only.

COMMUNICATION STYLE:
- Direct and specific — no filler
- Networking terminology used correctly
- When uncertain, say so and suggest how to verify
- IOS-XE configs formatted correctly with proper indentation"""

    def __init__(self, inventory=None, verbose=False):
        self.client        = anthropic.Anthropic()
        self.inventory     = inventory or DEFAULT_INVENTORY
        self.history       = []
        self.verbose       = verbose
        self.session_start = datetime.now()
        self.commands_run  = 0

        inv_summary = "\n".join(
            f"  - {name}: {info.get('role','?')} | {info['ip']} | {info.get('model','?')}"
            for name, info in self.inventory.items()
        )
        self.system = self.SYSTEM_TEMPLATE.format(inventory_summary=inv_summary)

    def chat(self, user_message):
        """Process a message and return AI response."""
        self.history.append({"role": "user", "content": user_message})

        response = self.client.messages.create(
            model="claude-opus-4-6",
            max_tokens=2048,
            system=self.system,
            messages=self.history
        )

        reply = response.content[0].text

        if "EXECUTE:" in reply:
            reply = self._handle_execution(reply)
        else:
            self.history.append({"role": "assistant", "content": reply})

        return reply

    def _handle_execution(self, reply):
        """Find EXECUTE: lines, run the commands, feed results back."""
        for line in reply.split("\n"):
            if line.strip().startswith("EXECUTE:"):
                _, details = line.split(":", 1)
                parts = details.split("|")
                if len(parts) == 2:
                    device_name = parts[0].strip()
                    command     = parts[1].strip()
                    output      = self._run_command(device_name, command)
                    # Feed output back to AI as a new user message
                    return self.chat(
                        f"Command `{command}` output from `{device_name}`:\n```\n{output}\n```\n"
                        f"Analyze this output and continue your response."
                    )
        # No valid EXECUTE found — just store and return
        self.history.append({"role": "assistant", "content": reply})
        return reply

    def _run_command(self, device_name, command):
        """Execute a show command on a named device."""
        if not NETMIKO_AVAILABLE:
            return "[Netmiko not installed]"

        device = self.inventory.get(device_name)
        if not device:
            return f"[Device '{device_name}' not found in inventory. Available: {list(self.inventory.keys())}]"

        # Safety check — only show commands
        if not command.strip().lower().startswith("show"):
            return f"[BLOCKED: Only 'show' commands allowed via AI execution. Got: '{command}']"

        try:
            params = {k: v for k, v in device.items()
                      if k in ("device_type", "ip", "username", "password")}
            if self.verbose:
                print(f"\n  [Executing on {device_name}]: {command}")
            with ConnectHandler(**params) as conn:
                output = conn.send_command(command, read_timeout=30)
                self.commands_run += 1
                return output
        except NetmikoAuthenticationException:
            return f"[Auth failed for {device_name}]"
        except NetmikoTimeoutException:
            return f"[Timeout connecting to {device_name}]"
        except Exception as e:
            return f"[Error: {e}]"

    def get_stats(self):
        """Return session statistics."""
        duration = datetime.now() - self.session_start
        return {
            "duration_minutes":      round(duration.total_seconds() / 60, 1),
            "messages_exchanged":    len(self.history),
            "commands_executed":     self.commands_run,
            "devices_in_inventory":  len(self.inventory),
        }

    def save_session(self, filename=None):
        """Save conversation to JSON."""
        if not filename:
            ts = self.session_start.strftime("%Y%m%d_%H%M%S")
            filename = f"coworker_session_{ts}.json"
        data = {
            "session_start": self.session_start.isoformat(),
            "stats":         self.get_stats(),
            "history":       self.history,
        }
        Path(filename).write_text(json.dumps(data, indent=2))
        print(f"Session saved: {filename}")

    def clear_history(self):
        """Clear conversation history."""
        self.history = []
        print("Conversation history cleared.")


# ─────────────────────────────────────────────────────────────
# STREAMLIT UI CODE (saved as a separate file)
# ─────────────────────────────────────────────────────────────

STREAMLIT_UI = '''#!/usr/bin/env python3
"""
cx_coworker_ui.py — Streamlit web interface for the CX Coworker.
Run with: streamlit run cx_coworker_ui.py
"""

import streamlit as st
from cx_coworker import CXCoworker

st.set_page_config(
    page_title="CX Digital Coworker",
    page_icon="🤖",
    layout="wide"
)

st.title("🤖 CX Digital Coworker")
st.caption("AI-powered Cisco network assistant | Powered by Claude + Netmiko")

# Initialize in session state — persists across reruns
if "coworker" not in st.session_state:
    st.session_state.coworker  = CXCoworker()
if "messages" not in st.session_state:
    st.session_state.messages = []

# ── Sidebar ──
with st.sidebar:
    st.header("📋 Device Inventory")
    for name, info in st.session_state.coworker.inventory.items():
        with st.expander(f"🖥️ {name}"):
            st.write(f"**IP:** {info['ip']}")
            st.write(f"**Role:** {info.get('role', 'Unknown')}")
            st.write(f"**Model:** {info.get('model', 'Unknown')}")

    st.divider()
    stats = st.session_state.coworker.get_stats()
    col1, col2 = st.columns(2)
    col1.metric("Messages", stats["messages_exchanged"])
    col2.metric("Commands Run", stats["commands_executed"])

    st.divider()
    if st.button("🗑️ Clear History"):
        st.session_state.coworker.clear_history()
        st.session_state.messages = []
        st.rerun()

    if st.button("💾 Save Session"):
        st.session_state.coworker.save_session()
        st.success("Session saved!")

# ── Chat Display ──
for msg in st.session_state.messages:
    with st.chat_message(msg["role"]):
        st.markdown(msg["content"])

# ── Input ──
if prompt := st.chat_input("Ask anything about your network..."):
    st.session_state.messages.append({"role": "user", "content": prompt})
    with st.chat_message("user"):
        st.markdown(prompt)

    with st.chat_message("assistant"):
        with st.spinner("Thinking..."):
            response = st.session_state.coworker.chat(prompt)
        st.markdown(response)

    st.session_state.messages.append({"role": "assistant", "content": response})
'''


# ─────────────────────────────────────────────────────────────
# CLI INTERFACE
# ─────────────────────────────────────────────────────────────

def run_cli(coworker):
    """Run interactive CLI session."""
    print("\n" + "╔" + "═"*58 + "╗")
    print("║" + "  🤖 CX Digital Coworker — Cisco Network AI".center(58) + "║")
    print("║" + f"  {len(coworker.inventory)} devices in inventory".center(58) + "║")
    print("║" + "  Commands: save | stats | devices | clear | exit".center(58) + "║")
    print("╚" + "═"*58 + "╝\n")

    while True:
        try:
            user_input = input("You: ").strip()
        except (KeyboardInterrupt, EOFError):
            print("\nGoodbye!")
            break

        if not user_input:
            continue

        if user_input.lower() == "exit":
            print("Goodbye! Session ended.")
            break
        elif user_input.lower() == "save":
            coworker.save_session()
            continue
        elif user_input.lower() == "stats":
            stats = coworker.get_stats()
            print(f"\n📊 Session Stats:")
            for k, v in stats.items():
                print(f"   {k}: {v}")
            print()
            continue
        elif user_input.lower() == "devices":
            print("\n📋 Device Inventory:")
            for name, info in coworker.inventory.items():
                print(f"   {name}: {info.get('role','?')} | {info['ip']}")
            print()
            continue
        elif user_input.lower() == "clear":
            coworker.clear_history()
            continue

        print("\n🤖 Coworker: ", end="", flush=True)
        response = coworker.chat(user_input)
        print(response)
        print()


# ─────────────────────────────────────────────────────────────
# MAIN
# ─────────────────────────────────────────────────────────────

def main():
    if "--save-ui" in sys.argv:
        Path("cx_coworker_ui.py").write_text(STREAMLIT_UI.strip())
        print("Created: cx_coworker_ui.py")
        print("Run web UI with: streamlit run cx_coworker_ui.py")
        return

    coworker = CXCoworker(verbose=True)
    run_cli(coworker)


if __name__ == "__main__":
    main()
```

### Running the Web UI

```bash
# Save the UI file
python cx_coworker.py --save-ui

# Install Streamlit
pip install streamlit

# Launch the web interface
streamlit run cx_coworker_ui.py
```

Opens at `http://localhost:8501` — a full chat interface accessible to your entire team.

---

*Previous: [Chapter 14](ch14_ai_syslog_auditor.md) | Next: [Chapter 16 — What Comes Next](ch16_whats_next.md)*
