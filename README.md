# 📘 The Intelligent Network Engineer
## Automating Network Infrastructure with Python and AI

> *From CLI Commands → Python Automation → AI-Powered Digital Coworkers*

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Python](https://img.shields.io/badge/Python-3.8%2B-blue)](https://www.python.org/)
[![Netmiko](https://img.shields.io/badge/Netmiko-4.3%2B-orange)](https://github.com/ktbyers/netmiko)

---

## 🎯 What This Book Is About

For two decades, network engineers have managed infrastructure one CLI command at a time. This book is about the transformation that is already underway — from manual, device-by-device operations to **programmatic automation** and **AI-augmented intelligence**.

By the time you finish this book, you will:

- Write Python scripts that SSH to network devices, collect structured data, and push configuration changes across entire fleets
- Build automation pipelines using industry-standard tools (Netmiko, Nornir, Genie)
- Create AI-powered tools that analyze your network, triage syslog events, and audit configurations
- Deploy a **Digital CX Coworker** — a conversational AI agent that knows your network and can query devices on demand

**This is not a theory book.** Every concept is demonstrated with working code. Every pattern comes from real-world network automation experience.

---

## 📚 How to Read This Book

Each chapter is a self-contained Markdown file designed to be read in VS Code with the preview pane open. Run every code example as you read — the muscle memory matters as much as the concepts.

```bash
# Clone the repo
git clone https://github.com/YOUR_USERNAME/intelligent-network-engineer.git
cd intelligent-network-engineer

# Open in VS Code
code .

# Install all dependencies
pip install -r requirements.txt
```

---

## 🗂️ Book Structure

### Front Matter

| File | Description |
|------|-------------|
| [Preface](preface/preface.md) | Why this book exists — 20 years of network engineering distilled |
| [Introduction](introduction/introduction.md) | The three waves of network engineering (CLI → Automation → AI) |

---

### 📘 Section 1 — Python Foundation for Network Engineers
*The building blocks. Every concept here is applied directly to networking.*

| Chapter | Title | Key Concepts |
|---------|-------|-------------|
| [Ch 01](section1/ch01_why_python.md) | Why Network Engineers Need Python | The cost of manual ops, automation mindset, Python vs other languages |
| [Ch 02](section1/ch02_data_types.md) | Python Data Types: The Building Blocks | Strings, lists, dicts, booleans — all through a networking lens |
| [Ch 03](section1/ch03_control_flow.md) | Control Flow, Loops, and Functions | if/elif, for loops, while, functions, list comprehensions |
| [Ch 04](section1/ch04_files_errors.md) | Files, Error Handling, and First Scripts | File I/O, try/except, environment variables, production patterns |
| [Ch 05](section1/ch05_classes_modules.md) | Classes, Modules, and Reusable Code | OOP, NetworkDevice class, module organization |

---

### 📗 Section 2 — Network Automation with Python and Netmiko
*Applying everything from Section 1 to real Cisco devices.*

| Chapter | Title | Lab Script |
|---------|-------|-----------|
| [Ch 06](section2/ch06_netmiko_intro.md) | Netmiko: SSHing to Your Network at Python Speed | `1-netmiko-show.py`, `2-netmiko-config.py` |
| [Ch 07](section2/ch07_multi_device.md) | Multi-Device Automation and File-Driven Inventory | `3-netmiko-config.py`, `4-netmiko-config.py` |
| [Ch 08](section2/ch08_production_scripts.md) | Production Scripts: Exceptions, Env Vars, and Show Commands | `5-netmiko-final.py`, `6-netmiko-final.py`, `7-netmiko-final.py` |
| [Ch 09](section2/ch09_backup_reports.md) | Automated Backup and Inventory Reports | `8-netmiko_backup.py`, `9-netmiko-report.py`, `10-netmiko-report.py` |
| [Ch 10](section2/ch10_nornir_parallel.md) | Nornir: Parallel Automation at Scale | `11-nornir-basics.py`, `12-nornir-backup.py` |
| [Ch 11](section2/ch11_git_cicd.md) | Git, CI/CD, and Treating Your Network as Code | Git workflow, GitLab CI, pipeline stages |

---

### 📕 Section 3 — AI-Powered Network Automation
*The AI era. Using LLMs to reason about your network.*

| Chapter | Title | What You Build |
|---------|-------|---------------|
| [Ch 12](section3/ch12_llm_api_prompts.md) | LLM APIs and Prompt Engineering | First API call, 3 levels of prompting, JSON output |
| [Ch 13](section3/ch13_ai_troubleshooter.md) | The AI Network Troubleshooter | Netmiko + Claude: collect → analyze → diagnose |
| [Ch 14](section3/ch14_ai_syslog_auditor.md) | AI Syslog Analyzer and Config Auditor | Syslog triage, 22-point security audit |
| [Ch 11b](section3/ch11b_ai_intro_llms.md) | How AI Works: LLMs, Tokens, and the Vibe-to-Agent Journey | LLM primer, tokens, inference, frontier models, orchestrator principle |
| [Ch 15](section3/ch15_digital_coworker.md) | The Digital CX Coworker | Conversational AI agent + Streamlit web UI |
| [Ch 16](section3/ch16_langchain_langgraph.md) | LangChain & LangGraph: Multi-Agent Automation (GPT-4o + Ollama) | Weather→Netmiko→LangGraph 3-agent; local Llama 3.2 + Gemma 4 |
| [Ch 17](section3/ch17_whats_next.md) | AIOps, Agentic Ops, and the Future Network Engineer | The orchestrator principle, AIOps closed-loop, 24-month roadmap |

---

### 📎 Appendix

| File | Description |
|------|-------------|
| [Appendix A](appendix/appendix_a_quick_reference.md) | Python + Netmiko + Anthropic API Quick Reference |
| [Appendix B](appendix/appendix_b_glossary.md) | Glossary of Terms |

---

## ⚙️ Setup

### Prerequisites

- Python 3.8 or higher
- pip
- A terminal / VS Code
- (Section 3) An Anthropic API key from [console.anthropic.com](https://console.anthropic.com)

### Install Dependencies

```bash
pip install -r requirements.txt
```

### Set Credentials (Never Hardcode These)

```bash
# Linux / Mac — add to ~/.bashrc or ~/.zshrc
export NETMIKO_USERNAME="cisco"
export NETMIKO_PASSWORD="cisco"
export ANTHROPIC_API_KEY="sk-ant-..."

# Windows PowerShell
$env:NETMIKO_USERNAME = "cisco"
$env:NETMIKO_PASSWORD = "cisco"
$env:ANTHROPIC_API_KEY = "sk-ant-..."
```

### Lab Devices

Sections 2 and 3 reference the Cisco dCloud lab (jagadnag/labato_1010):

```
Device IPs:  198.18.1.11, 198.18.1.12
Username:    cisco
Password:    cisco
```

You can substitute any Cisco IOS/IOS-XE device. Update the IPs in your `device_list` file.

---

## 📁 Repository Structure

```
intelligent-network-engineer/
│
├── README.md                     ← You are here
├── requirements.txt              ← All Python dependencies
│
├── preface/
│   └── preface.md
│
├── introduction/
│   └── introduction.md
│
├── section1/                     ← Python Fundamentals
│   ├── ch01_why_python.md
│   ├── ch02_data_types.md
│   ├── ch03_control_flow.md
│   ├── ch04_files_errors.md
│   └── ch05_classes_modules.md
│
├── section2/                     ← Network Automation
│   ├── ch06_netmiko_intro.md
│   ├── ch07_multi_device.md
│   ├── ch08_production_scripts.md
│   ├── ch09_backup_reports.md
│   ├── ch10_nornir_parallel.md
│   └── ch11_git_cicd.md
│
├── section3/                     ← AI-Powered Automation
│   ├── ch12_llm_api_prompts.md
│   ├── ch13_ai_troubleshooter.md
│   ├── ch14_ai_syslog_auditor.md
│   ├── ch15_digital_coworker.md
│   └── ch16_whats_next.md
│
├── appendix/
│   ├── appendix_a_quick_reference.md
│   └── appendix_b_glossary.md
│
└── scripts/                      ← All runnable Python scripts
    ├── section1/                 ← Standalone examples
    ├── section2/                 ← Netmiko lab scripts
    └── section3/                 ← AI tool scripts
```

---

## 🔑 Core Philosophy

> **The network engineer who automates is not replaced by automation. They become the engineer who can do what it previously took a team to accomplish.**

This book teaches you to automate not because CLI is dead, but because automation multiplies what you can do with the knowledge you already have. The CLI will always be there. But the engineers who also know how to code it, automate it, and make AI reason about it will operate at a level of leverage that simply wasn't possible before.

---

## 📜 License

MIT License — see [LICENSE](LICENSE) for details. Use freely, attribute when sharing.

---

*Based on Python Crash Course (3rd Ed.) by Eric Matthes + jagadnag/labato_1010 lab scripts + twin-bridges/netmiko_course*
