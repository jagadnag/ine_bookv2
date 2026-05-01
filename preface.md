# Preface: Why I Wrote This Book

> *"The network is the computer."*  
> — Sun Microsystems, 1984

---

Twenty years ago, I sat in a data center with a console cable in one hand and a Cisco Command Reference in the other, methodically typing the same twenty CLI commands into a brand-new router. I had done it the week before on a different router. I would do it again the week after on another.

Nobody questioned this. It was simply how networking was done.

That world is gone. Or rather — it should be.

The network engineers who will thrive in the next decade are not the ones who memorized the most `show` commands. They are the ones who learned to teach computers to do the memorizing for them.

---

## Why This Book Exists

This book is the result of twenty years of hands-on networking experience combined with years of teaching Python and network automation to engineers at every level — from CCNA students writing their first `for` loop to CCIE professionals automating multi-vendor enterprise networks.

Every example in these pages comes from **real problems I encountered in real networks**. Every tool I recommend is one I have used in production. Every pattern you will learn here has been tested against actual Cisco IOS-XE devices, not just described in theory.

This is not a theory book. It is a workshop.

You will get your hands dirty. Your scripts will break. You will fix them. And somewhere in that process you will discover that what once took you four hours to do manually across thirty devices now takes your Python script forty seconds.

That moment — the first time your code does the work instead of you — is what this book is building toward.

---

## The Three Waves

When I look back at two decades of network engineering, I can see three distinct waves:

**Wave 1 — The CLI Era (1990–2012)**  
Configuration was manual. Verification was manual. Documentation was often scribbled in Visio, accurate on the day it was drawn and gradually wrong from that day forward. Engineers who were fast typists with good memories of IOS syntax were the most valued people in the room.

**Wave 2 — The Automation Era (2012–2022)**  
Network vendors began exposing APIs. Tools like Ansible, Terraform, and Nornir appeared. Python became the lingua franca. Engineers who could write even basic scripts became enormously more productive than those who could not.

**Wave 3 — The AI Era (2022–present)**  
Large Language Models can now read your running configurations, analyze your syslog streams, generate configuration templates from plain English, and flag compliance gaps without a single hand-written rule. The engineers who learn to leverage AI as a force multiplier will operate at a level of productivity that was simply not possible before.

This book covers all three waves — teaching you the Python fundamentals, applying them to real network automation, and then wiring AI intelligence into everything you have built.

---

## Who This Book Is For

This book is for **network engineers and architects** who:

- Manage networks primarily through the CLI and want to change that
- Have heard of Python but never written a script for their network
- Have written some automation but want to level it up with AI
- Want to understand what the "AI-powered network" actually means in practice

You do **not** need a programming background. You need curiosity, patience with error messages, and the willingness to type through a few hundred examples. The muscle memory you build here will serve you for the rest of your career.

---

## What You Will Build

By the end of this book, you will have built:

| Tool | What It Does |
|------|-------------|
| Multi-device configurator | Pushes config changes to your entire fleet simultaneously |
| Config backup automation | Pulls and saves running configs from all devices, timestamped |
| Inventory report generator | SSHes to all devices, extracts structured data, exports to CSV |
| Parallel automation runner | Netmiko + Nornir, all devices in parallel |
| AI Network Troubleshooter | Collects device data, Claude analyzes it |
| AI Syslog Analyzer | Triages events, groups incidents, prioritizes alerts |
| AI Config Auditor | 22-point security check against any running config |
| Digital CX Coworker | Conversational AI with device inventory + live command execution |

---

## How to Use This Book

Read it sequentially the first time. Each chapter builds on the last. Run every code example as you read — reading without typing is like watching someone work out and expecting to get fit.

After your first read, it becomes a reference. The chapter files are designed to be searchable from VS Code. The scripts in `/scripts` are meant to be copied, adapted, and run against your own devices.

---

## A Personal Note

I have taught these skills to hundreds of network engineers. The ones who succeed all have one thing in common: they did not wait until they felt "ready" to start. They started imperfect, broke things safely in a lab, and improved through practice.

There is no perfect time to start automating your network. The right time is now. The right first script is the one that solves an actual problem you have today — even if it is just printing a list of your device hostnames from a file.

Start there. The rest follows.

---

*Next: [Introduction — The Evolution You Cannot Ignore](../introduction/introduction.md)*
