---
layout: page
icon: fas fa-code-branch
order: 4
title: Projects
---

## Home SOC Lab

**Stack:** Wazuh · Docker · OpenSearch · Linux · MITRE ATT&CK

A working SIEM/XDR deployment running on my home network. Wazuh server on a desktop with agents deployed across a Raspberry Pi server, laptop, and other devices. Collects real logs, maps alerts to MITRE ATT&CK techniques, monitors file integrity, and scans for CVEs.

What I learned from this project goes beyond the install guide — version mismatches that silently block agent registration, certificate generation quirks, and the difference between reading about SIEM and actually triaging alerts from your own network traffic.

[Full write-up →](/posts/home-soc-wazuh-homelab/)

---

## Automated Security News Briefing

**Stack:** n8n · Ollama (llama3.1) · Telegram API · RSS

An n8n workflow that collects the day's cybersecurity headlines from three sources (The Hacker News, Bleeping Computer, Krebs on Security), runs them through a local LLM for summarization, and delivers a curated briefing to a private Telegram channel every morning. No external AI API costs — everything runs locally on Ollama.

This was my first real n8n project and it forced me to think about error handling, rate limiting, and formatting output for a messaging platform in a way that a script wouldn't.

---

## Newsletter Image Generation Pipeline

**Stack:** Bannerbear API · n8n · Dynamic Templates

An automated pipeline that generates branded newsletter images using Bannerbear's template API. Three templates (news, tips, alerts) with a cyberpunk aesthetic — dark background, green/cyan terminal-style typography, dynamic fields for headlines, categories, and descriptions.

Handles text rendering, image composition, and webhook callbacks for delivery status. Designed to integrate with a content workflow for automated social media posting.

---

## Self-Hosted Home Server

**Stack:** Raspberry Pi 4 · Docker · Portainer · n8n · Gitea · Vaultwarden · Caddy

A Raspberry Pi running 14+ Docker services that forms the backbone of my home automation and development infrastructure. Includes self-hosted Git (Gitea), password management (Vaultwarden), workflow automation (n8n), monitoring (Portainer), reverse proxy with auto SSL, and more.

This is the kind of project that doesn't have a single finished moment — it evolves every time I find a new service worth self-hosting or hit a limitation that needs a workaround.

---

## Personal AI Agent Infrastructure

**Stack:** Hermes Agent (Nous Research) · OpenRouter · Ollama · Custom Skills

A CLI-based AI assistant integrated into my workflow that goes beyond chat. Manages cron jobs, SSHes into servers, queries APIs, reads and writes to an Obsidian vault, and runs system diagnostics — all through natural language. Uses a multi-model setup: lightweight local models for routine tasks via Ollama, and routed reasoning through OpenRouter for complex analysis.

Built entirely on open-source tooling. No cloud subscriptions, no vendor lock-in.

---

## CTF Write-up Series

**Platforms:** TryHackMe · HackTheBox

A growing collection of penetration testing write-ups covering everything from EternalBlue (MS17-010) exploitation to Linux privilege escalation via SUID path hijacking. Each write-up documents the full kill chain — recon, exploitation, post-exploitation, and the thinking behind each decision.

[Browse write-ups →](/categories/ctf/)