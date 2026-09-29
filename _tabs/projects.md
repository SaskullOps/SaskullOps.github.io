---
layout: page
icon: fas fa-code-branch
order: 4
title: Projects
---

## QRFleet: Fleet Inspections and Damage Reporting

**Stack:** Next.js · TypeScript · FastAPI · PostgreSQL · Supabase · Docker · Caddy · Hetzner

A web application for fleet inspections and damage reporting. An asset scan leads into a checklist with damage observations, photos and signatures. The API checks the completed inspection and stores its PDF record. The interface supports mobile and desktop workflows in five languages.

Built with a separate Python API, a browser-side offline write queue and managed database/object storage in Supabase. Docker Compose runs the application on a Hetzner VPS behind Caddy and Cloudflare. The offline queue persists pending work in IndexedDB and synchronizes it when the connection returns; the server still validates the final result.

[Architecture and stack →](/posts/qrfleet-stack-from-qr-to-inspection/) · [Live site →](https://qrfleet.com)

---

## Personal AI Agent Infrastructure

**Stack:** Hermes Agent (Nous Research) · OpenRouter · Ollama · Custom Skills

A CLI-based AI assistant integrated into my workflow that goes beyond chat. Manages cron jobs, SSHes into servers, queries APIs, reads and writes to an Obsidian vault, and runs system diagnostics, all through natural language. Uses a multi-model setup: lightweight local models for routine tasks via Ollama, and routed reasoning through OpenRouter for complex analysis.

Built entirely on open-source tooling. No cloud subscriptions, no vendor lock-in.

[How I route coding requests →](/posts/hermes-openrouter-pareto-code/)

---

## Image Generation Pipeline

**Stack:** Python · HTML/CSS · Chromium · n8n webhook

A local generator for newsletter headers, LinkedIn images and carousel slides. Python fills HTML templates and Chromium renders them to PNG. I can change the layouts in CSS rather than editing hosted templates. There's an HTTP endpoint for n8n to request images, but the output still gets a manual review before I use it.

[How I built it →](/posts/building-my-own-image-generator/)

---

## Self-Hosted Home Server

**Stack:** Raspberry Pi 4 · Docker · Portainer · n8n · Gitea · Vaultwarden · Caddy

A Raspberry Pi running 14+ Docker services that forms the backbone of my home automation and development infrastructure. Includes self-hosted Git (Gitea), password management (Vaultwarden), workflow automation (n8n), monitoring (Portainer), reverse proxy with auto SSL, and more.

This is the kind of project that doesn't have a single finished moment. It evolves every time I find a new service worth self-hosting or hit a limitation that needs a workaround.

---

## Automated Security News Briefing

**Stack:** n8n · Ollama (llama3.1) · Telegram API · RSS

An n8n workflow that collects the day's cybersecurity headlines from three sources (The Hacker News, Bleeping Computer, Krebs on Security), runs them through a local LLM for summarization, and delivers a curated briefing to a private Telegram channel every morning. No external AI API costs, everything runs locally on Ollama.

This was my first real n8n project and it forced me to think about error handling, rate limiting, and formatting output for a messaging platform in a way that a script wouldn't.

---

## Home SOC Lab

**Stack:** Wazuh · Docker · OpenSearch · Linux · MITRE ATT&CK

A personal-interest project on the side: a working SIEM/XDR deployment running on my home network. Wazuh server on a desktop with agents deployed across a Raspberry Pi server, laptop, and other devices. Collects real logs, maps alerts to MITRE ATT&CK techniques, monitors file integrity, and scans for CVEs.

What I learned from this project goes beyond the install guide: version mismatches that silently block agent registration, certificate generation quirks, and the difference between reading about SIEM and actually triaging alerts from your own network traffic.

[Full write-up →](/posts/home-soc-wazuh-homelab/)

---

## CTF Write-up Series

**Platforms:** TryHackMe · HackTheBox

A handful of penetration testing write-ups I publish when I feel like it, covering everything from EternalBlue (MS17-010) exploitation to Linux privilege escalation via SUID path hijacking. Each write-up documents the full kill chain: recon, exploitation, post-exploitation, and the thinking behind each decision.

[Browse write-ups →](/categories/ctf/)
