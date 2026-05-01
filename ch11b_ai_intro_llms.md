# Chapter 11b — How AI Works: LLMs, Tokens, Inference, and the Road from Vibe-Coding to Agents

> *"You don't need to understand how an engine works to drive a car. But if you want to build a race car, understanding the engine changes everything."*

---

## Chapter Goal

Before you write your first AI-powered network tool, you need a working mental model of how these systems actually function. Not at the PhD level — at the practitioner level. What is a token? What happens during inference? Why does context window size matter when you are injecting a router's running config into a prompt? Why can Claude Opus 4.6 analyze a complex multi-vendor network topology when GPT-3 from 2020 could barely answer "What is OSPF?"

This chapter answers all of that — and connects it to the journey you have been on since Chapter 1.

**Key Points:**
- How Large Language Models actually work: training, weights, and emergent intelligence
- Tokens, context windows, and why they matter for network automation
- Inference, temperature, and how the model "thinks"
- Why frontier models like Claude Opus 4.6 and GPT-4o are genuinely different
- Prompt engineering, tool calling, and MCP — the practitioner toolkit
- The complete arc: Python → Netmiko → AI tools → Vibe-coding → Agent orchestration

---

## How Large Language Models Work

### The Core Insight: Prediction at Scale

A Large Language Model is trained to do one thing: **predict the next token** given everything that came before it. That is not a simplification — that is literally the training objective. And yet, from that simple task applied at sufficient scale to sufficient data, something remarkable emerges: a system that understands, reasons, and generates knowledge across virtually every human domain.

Here is the key insight that most people miss: **nobody programmed these models to understand networking.** Nobody wrote rules like "if the user mentions OSPF, respond with routing protocol knowledge." The model absorbed the patterns of how networking concepts relate to each other by reading enough networking text — RFCs, Cisco documentation, Stack Overflow threads, textbooks, blog posts, forum discussions — that the relationships became encoded in billions of numerical parameters called weights.

```
TRAINING PHASE (done once, by the AI company):

┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  Input: Hundreds of billions of documents                          │
│  (every RFC, every Cisco doc, every networking forum thread,        │
│   every Stack Overflow answer, every textbook, every blog post)    │
│                                                                     │
│                         ↓  (training)                              │
│                                                                     │
│  Output: A set of ~100 billion numerical weights                   │
│  that encode the statistical patterns across all that text         │
│                                                                     │
│  Think of it as: a very compressed representation of               │
│  "how ideas relate to each other across all human writing"         │
└─────────────────────────────────────────────────────────────────────┘

INFERENCE PHASE (every time you call the API):

┌─────────────────────────────────────────────────────────────────────┐
│  Your prompt  →  Model applies weights  →  Generates next token    │
│                                          →  Appends token          │
│                                          →  Generates next token   │
│                                          →  Repeat until done      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Tokens: The Currency of AI

### What Is a Token?

A token is the unit of text that models process. Tokens are not exactly words — they are chunks of characters that the model learned to group together during training based on frequency patterns in the training data.

```
"networking"         →  1 token    (common English word)
"GigabitEthernet"   →  3 tokens   (uncommon, split: "Gigabit" + "Ether" + "net")
"OSPF"              →  1 token    (common enough as a unit)
"show ip bgp sum"   →  5 tokens   (spaces count as tokens too)
" running-config"   →  3 tokens   (leading space + "running" + "-config")
```

**Rules of thumb every practitioner needs:**

| Measurement | Approximation |
|-------------|---------------|
| 1 token | ≈ 0.75 English words |
| 1 token | ≈ 4 characters |
| 1,000 tokens | ≈ 750 words ≈ 1.5 pages |
| Typical running config | 5,000–15,000 tokens |
| Full show tech-support | 50,000–200,000 tokens |

### Why Tokens Matter in Practice

```python
# Before sending a config to the AI, estimate the token cost
running_config = Path("sw-core-01.cfg").read_text()
char_count  = len(running_config)
token_est   = char_count // 4   # rough estimate

print(f"Config: {char_count:,} chars ≈ {token_est:,} tokens")
print(f"Model context: 200,000 tokens (Claude Opus 4.6)")
print(f"Fits in context: {'YES' if token_est < 180000 else 'TRUNCATE NEEDED'}")
# Config: 24,312 chars ≈ 6,078 tokens  → YES, comfortably fits
```

---

## Context Windows: What the Model Can See

The context window is the total amount of text the model can process in a single call — your system prompt, conversation history, injected device data, and the response being generated.

```
┌─────────────────────────────────────────────────────────────────┐
│                   CONTEXT WINDOW (200K tokens)                   │
│                                                                  │
│  System Prompt          ████░░░░░░░░  ~500 tokens               │
│  Conversation History   ████████░░░░  ~5,000 tokens             │
│  Injected Device Data   ████████████  ~8,000 tokens             │
│  Room for Response      ████░░░░░░░░  ~2,000 tokens             │
│                                                                  │
│  Unused capacity:  ~184,500 tokens — you could inject           │
│  dozens of device configs simultaneously                         │
└─────────────────────────────────────────────────────────────────┘

Model Context Window Sizes (as of 2025):
  Claude Opus 4.6:   200,000 tokens  ← Can see an entire network's configs
  GPT-4o:            128,000 tokens
  Gemini 2.0 Flash:  1,000,000 tokens ← Can see an entire datacenter
  Llama 3.2 (local): 128,000 tokens
```

**Why this matters for you:** Modern frontier models can hold an entire router's running config, multiple show command outputs, several syslog batches, and a full conversation history — all within a single context window. This is why the AI troubleshooter in Chapter 13 can give coherent analysis across eleven different show commands: it sees them all simultaneously, just like a senior engineer would read a printed show tech-support.

---

## Inference: How the Model Generates Text

When you call the API, the model does not look up answers in a database. It generates each token sequentially, with every previously generated token influencing the probability distribution for the next one.

```
Your prompt: "The BGP neighbor is in ___"

Step 1: Model processes all prompt tokens through all 100B weights
Step 2: Produces probability over every possible next token:
        "Active"    → 41%
        "Idle"      → 22%
        "down"      → 18%
        "Established" → 9%
        ...

Step 3: Samples from distribution → selects "Active"

Step 4: "The BGP neighbor is in Active ___"
Step 5: Next distribution:
        " state"    → 67%
        ","         → 18%
        " mode"     → 8%
        ...

This loop continues until the model produces <end_of_sequence>
```

### Temperature: Controlling Creativity vs. Consistency

Temperature controls randomness in the sampling step:

```python
# For network analysis tools — use low temperature
# Consistent, deterministic, reliable
client.messages.create(
    model="claude-opus-4-6",
    max_tokens=2048,
    # temperature defaults to ~0.7 if not set
    # For structured JSON output and analysis, go lower:
    messages=[...]
)

# Low temperature (0.0–0.3): deterministic, great for:
#   - JSON output, structured reports, config audits
#   - Situations where you need the same answer every time

# High temperature (0.7–1.0): creative, varied, good for:
#   - Brainstorming, documentation writing, varied suggestions
#   - When you want different phrasings each run
```

---

## Prompt Engineering: The Practitioner's Toolkit

Prompt engineering is the art of writing instructions that consistently elicit accurate, useful, structured responses from an LLM. After the model itself, this is the most important skill in applied AI.

```
THE THREE LEVELS (covered in Chapter 12):

Level 1 — Role-Task-Format (R-T-F):
  "You are a [role]. [Task description]. Format as [structure]."
  Best for: Quick one-off analysis, simple structured output

Level 2 — Structured Context Injection:
  System prompt (stable persona) + User message (dynamic device data)
  Best for: Device health assessments, operational analysis

Level 3 — RAG + Chain-of-Thought:
  Inject knowledge documents + instruct step-by-step reasoning
  Best for: Lifecycle assessment, compliance audits, complex multi-factor decisions
```

### Key Prompt Engineering Principles for Network Automation

```python
# Principle 1: Be specific about the device context
# BAD:
"Analyze this network device"

# GOOD:
f"""
Device: {hostname} ({model})
IOS-XE: {version}
Uptime: {uptime}
CPU (5-min): {cpu}%
Down interfaces: {', '.join(down_intfs)}
"""

# Principle 2: Request structured output for programmatic use
"Return ONLY valid JSON with no markdown fences:
{
  'risk_level': 'CRITICAL|HIGH|MEDIUM|LOW',
  'issues': [...],
  'actions': [...]
}"

# Principle 3: Use Chain-of-Thought for complex analysis
"Think step by step:
STEP 1 — Is this platform EoL? What are the dates?
STEP 2 — Are there known vulnerabilities in this IOS version?
STEP 3 — Combine into overall risk score.
STEP 4 — Prioritized actions with specific commands."
```

---

## Tool Calling: Giving AI Hands

The raw Anthropic API returns text. Tool calling — also called function calling — lets the model request the execution of Python functions you define, inspect the results, and continue reasoning. This is what transforms a text-generating model into an agent that can act.

```
WITHOUT TOOL CALLING:

User: "What interfaces are down on sw-core-01?"
AI:   "I would need you to run 'show ip interface brief' and share the output."
↑ Not very useful. The AI knows what to do but can't do it.

WITH TOOL CALLING:

User: "What interfaces are down on sw-core-01?"
AI:   [internally decides to call run_show_command("sw-core-01", "show ip int brief")]
Tool: [Netmiko SSHes to device, returns raw output]
AI:   [reads output, analyzes it]
AI:   "GigabitEthernet0/1 and GigabitEthernet0/7 are down. GigabitEthernet0/1 went
       down 3 hours ago based on the syslog correlation. Likely causes: ..."
↑ The AI did the work.
```

The `@tool` decorator pattern from LangChain (Chapter 16) is how you implement this. The tool's docstring is what the model reads to decide when and how to call it — write them carefully.

---

## MCP: Model Context Protocol

MCP (Model Context Protocol) is an open standard — developed by Anthropic — that standardizes how AI models connect to external tools and data sources. Think of it as a universal plugin system for AI assistants.

```
TRADITIONAL APPROACH:
  Each tool integration requires custom code
  Claude + Netmiko   → custom glue code
  Claude + Ansible   → different custom glue code
  Claude + NetBox    → yet more custom glue code

MCP APPROACH:
  Standardized protocol that any AI client understands
  Claude ──── MCP ──── Netmiko Server
  Claude ──── MCP ──── Ansible Server
  Claude ──── MCP ──── NetBox Server

  Write the server once, any MCP-compatible client uses it
```

For network engineers, MCP means:
- You can build a "Cisco Network MCP Server" and connect it to Claude Desktop, VS Code Copilot, or any other MCP client
- Others can share MCP servers for common tools (there is a growing ecosystem)
- Your automation tools become reusable across AI clients, not locked to one

---

## Why Frontier Models Are Genuinely Different

### The Capability Jump Is Not Incremental

```
MODEL GENERATION     →  NETWORK CAPABILITY

GPT-3 (2020):          "What is OSPF?" → Reasonable explanation
ChatGPT (2022):         Explain OSPF, give examples, compare to EIGRP
GPT-4 (2023):           Read a pasted config, identify OSPF issues
Claude 3 Opus (2024):   Analyze full show tech, correlate events, write fix,
                         explain business impact, suggest prevention
Claude Opus 4.6 (2025): Orchestrate multi-step investigation across devices,
                         write and test automation code, reason about novel
                         topologies it has never seen, maintain expert-level
                         accuracy across a 4-hour troubleshooting session
```

The jump from GPT-3 to frontier models is not "20% better at answering questions." It is a qualitative change in reasoning capability — the ability to hold complex multi-variable problems in context, reason about them systematically, and produce accurate analysis for novel situations.

### What Frontier Models Know About Networking

These models were trained on essentially everything ever published about networking:

- Every Cisco IOS/IOS-XE/NX-OS command reference page
- Every RFC (1,000+ documents)
- Thousands of network engineering textbooks and study guides
- Years of Cisco Community, NetworkEngineering.StackExchange, Reddit r/networking
- Vendor whitepapers, design guides, deployment guides for every major platform
- Network engineering blogs, conference talks, certification study materials

A frontier model has, in some meaningful sense, absorbed more networking knowledge than any single human engineer could read in a lifetime. When you ask Claude Opus 4.6 about a specific BGP confederation topology edge case, it is drawing on synthesized knowledge from thousands of similar discussions.

**The practical implication:** The bottleneck is no longer "does the AI know enough about networking?" It is "do *you* know enough to direct the AI toward the right problem and evaluate whether its answer is correct?"

### The Shift in What Is Scarce

```
WHAT FRONTIER MODELS HAVE COMMODITIZED:
  ✗ Knowing CLI syntax by memory
  ✗ Finding the right Cisco documentation page
  ✗ Writing standard boilerplate automation code
  ✗ Pattern-matching common error messages
  ✗ Generating configuration templates

WHAT FRONTIER MODELS CANNOT REPLACE:
  ✓ Understanding why your specific network behaves the way it does
  ✓ Knowing which constraints are organizational vs. technical
  ✓ Making risk/impact judgments for changes in your environment
  ✓ Deciding what the network should look like, not just how it works
  ✓ Building trust with business stakeholders about technical risk
  ✓ Recognizing when AI analysis is wrong because you know the context
```

Deep protocol expertise does not become less valuable. It becomes the foundation on which all AI capability rests.

---

## The Complete Journey: Python → Netmiko → AI → Vibe-Coding → Agents

Now that you understand how AI works, the arc of this book makes complete sense:

```
SECTION 1 — Python Fundamentals (Ch 1–5)
Goal: Write and understand code

  You learn: variables, lists, dicts, loops, functions,
  classes, files, exceptions

  Why it matters: Every AI library, every automation tool,
  every agent framework is Python. You cannot evaluate
  AI-generated code without understanding the language.

    ↓

SECTION 2 — Network Automation (Ch 6–11)
Goal: Apply Python to real devices

  You learn: Netmiko, Nornir, Genie, Git, CI/CD

  Why it matters: These are the tools your AI agents will
  use. An agent that SSHes to a device and runs show
  commands is just Netmiko wrapped in an AI decision loop.
  Understanding Netmiko means understanding what the agent
  is actually doing.

    ↓

SECTION 3 — AI Integration (Ch 11b–17)
Goal: Wire AI intelligence into everything you built

  Stage 3a: Understanding AI (this chapter)
  → What are tokens? How does inference work?
  → Why frontier models reason differently?

  Stage 3b: Building AI tools (Ch 12–15)
  → LLM APIs, prompt engineering, the AI troubleshooter,
    syslog analyzer, config auditor, Digital CX Coworker

  Stage 3c: Vibe-Coding
  → Using AI to build tools faster: describe → review → refine
  → The workflow: prompt → AI generates → you evaluate → iterate
  → Possible BECAUSE you understand what you are evaluating

  Stage 3d: Agent Frameworks (Ch 16)
  → LangChain tool-calling agents, LangGraph multi-agent systems
  → Weather agent → Netmiko agent → 3-agent network ops system

  Stage 3e: The Orchestrator (Ch 17)
  → AIOps, agentic operations, the future of network engineering
  → Defining what agents do, reviewing outcomes, handling escalations
```

### The Vibe-Coding Bridge

Vibe-coding deserves specific attention because it is misunderstood. It is not "having AI write all the code." It is a development methodology where:

1. You describe what you want in plain English
2. AI generates a first version
3. You *evaluate* it using your understanding from Sections 1 and 2
4. You give AI feedback on what is wrong or incomplete
5. AI refines
6. You test against a real device
7. Repeat until it works

The critical step is 3 — evaluation. An engineer who completed Section 2 of this book knows immediately whether a Netmiko script is handling auth exceptions correctly, whether the Genie parsing will work for their IOS version, whether the file-writing logic will produce correctly named backup files. An engineer without that background cannot evaluate — and therefore cannot vibe-code reliably.

This is the reason you learned Python and Netmiko before touching the AI chapters. Not to write every line manually forever. To build the judgment that makes AI collaboration competent.

---

## The Orchestrator Principle: Stated Plainly

We said it in the introduction and it bears repeating with the full context you now have:

> **An engineer who does the work of an agent will get replaced by an agent soon. But the engineer who becomes the orchestrator of AI agents will thrive in the next era.**

After everything in this chapter, here is what that means technically:

**The work of an agent** is: defined procedures, repeatable actions, structured data collection, pattern matching, rule execution. Anything that can be described precisely enough to automate is agent work. SSHing to forty switches is agent work. Pulling configs on a schedule is agent work. Checking VLAN consistency against a policy is agent work.

**The work of an orchestrator** is: defining what agents should do, setting the scope of autonomous action, evaluating agent outputs, handling exceptions that fall outside defined scope, designing how agents coordinate, updating agent policies based on outcomes. None of this can be automated away — it is the control plane for the automation.

The journey from agent work to orchestrator work is the journey of upskilling. Python. Netmiko. Git. CI/CD. LangChain. LangGraph. Prompt engineering. Agent architecture. Each skill moves you further along that spectrum.

---

## Chapter Summary

Large Language Models predict tokens from weights learned during training on vast text corpora. Intelligence emerges from scale — nobody programmed these models to understand networking; they absorbed the patterns. Context windows define what the model can see in a single call; frontier models like Claude Opus 4.6 can hold entire router configs plus conversation history simultaneously. Temperature controls determinism. Tool calling gives AI the ability to act, not just answer. MCP standardizes how AI connects to external systems.

Frontier models have encoded essentially all public networking knowledge. The bottleneck is now your ability to direct them, evaluate their output, and build the systems that deploy them. Deep protocol expertise remains essential — it becomes the lens through which you judge everything the AI produces.

The journey from Python fundamentals to agent orchestration is the arc of this book. Each section builds the foundation for the next. The vibe-coding methodology connects AI-assisted development to that foundation. The orchestrator principle names where this all leads.

The agents are being built. The only question is: are you building them, or waiting?

---

*Previous: [Chapter 11 — Git and CI/CD](../section2/ch11_git_cicd.md) | Next: [Chapter 12 — LLM APIs and Prompt Engineering](ch12_llm_api_prompts.md)*
