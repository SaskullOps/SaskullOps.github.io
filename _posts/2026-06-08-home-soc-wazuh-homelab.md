---
title: "Building a home SOC lab with Wazuh"
date: 2026-06-08 12:00:00 +0100
categories:
  - Homelab
  - Blue Team
tags:
  - wazuh
  - soc
  - siem
  - docker
  - linux
  - homelab
  - defensive-security
---

This is a post I've been meaning to write since I decided to stop just reading about blue team and actually build something. The goal: a real SIEM running at home, monitoring my own machines, generating real alerts from real traffic. No sandboxed VMs, no pre-baked labs — my actual network.

Here's how I set it up, what broke, and what I learned.

## The setup

My homelab runs across three machines:

- **Pi** — Raspberry Pi 4 acting as a home server (Pi-hole, n8n, Vaultwarden, Gitea, and more running in Docker)
- **Tower** — desktop with 32 GB RAM, running Ollama for local AI models
- **Laptop** — daily driver, CachyOS (Arch-based)

The plan: Wazuh server on the tower, agents everywhere else.

## Why Wazuh

Wazuh is an open-source SIEM/XDR platform. In short, it:

- Collects logs from all your agents (Linux, Windows, Mac)
- Detects threats using rules mapped to the **MITRE ATT&CK** framework
- Monitors file integrity (FIM) — alerts when important files change
- Scans for vulnerabilities (CVEs) on connected machines
- Can take **active response** — automatically block an IP if it triggers enough alerts

It's also what a lot of companies actually use in production, which makes it useful to learn.

## Installation

Wazuh has an official Docker Compose setup for the server (single-node for a homelab is enough). The stack is three containers:

- `wazuh.manager` — receives data from agents, applies rules
- `wazuh.indexer` — OpenSearch instance that stores all events
- `wazuh.dashboard` — the web UI

```bash
git clone https://github.com/wazuh/wazuh-docker.git --branch v4.14.5 --depth 1 ~/wazuh-docker
cd ~/wazuh-docker/single-node
docker compose -f generate-indexer-certs.yml run --rm generator
docker compose up -d
```

**Important:** use the repo tag that matches the agent version you're going to install. I made the mistake of cloning `v4.12.0` and then installing `4.14.5` agents from the apt repo. The manager rejects agents newer than itself. Start with the right version.

Dashboard available at `https://<tower-ip>` — default credentials `admin / SecretPassword` (change these).

## Agents

For the Pi (Debian ARM64):

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg \
  --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg \
  --import && sudo chmod 644 /usr/share/keyrings/wazuh.gpg

echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] \
  https://packages.wazuh.com/4.x/apt/ stable main" \
  | sudo tee /etc/apt/sources.list.d/wazuh.list

sudo apt update
WAZUH_MANAGER='<tower-ip>' sudo -E apt install -y wazuh-agent
sudo systemctl enable wazuh-agent --now
```

For the laptop (Arch/CachyOS):

```bash
yay -S wazuh-agent
sudo sed -i 's/<address>MANAGER_IP<\/address>/<address><tower-ip><\/address>/' \
  /var/ossec/etc/ossec.conf
sudo systemctl enable wazuh-agent --now
```

Once both agents are running, they appear automatically in **Agents** in the dashboard.

## What to look at first

Once the agents are connected, here's the order I'd explore the dashboard:

### 1. Security Events
The main feed of everything Wazuh has detected. Filter by agent to see what's happening on each machine. You'll see authentication events, package installs, cron jobs — normal activity, but learning to read it is the point.

### 2. MITRE ATT&CK
Wazuh maps every alert to a MITRE technique. Even on a quiet home network you'll see detections — reconnaissance, credential access, lateral movement attempts. This is where it starts to feel like real SOC work.

### 3. Vulnerability Detection
Wazuh scans each agent and cross-references installed packages against the NVD database. It will find CVEs in your system. Some will be informational, some will matter. Learn to triage them.

### 4. File Integrity Monitoring
By default Wazuh monitors `/etc`, `/usr/bin`, `/usr/sbin`, and a few other paths. Any modification — file added, changed, deleted — creates an alert. Useful for detecting persistence mechanisms.

### 5. Agents → click an agent
Each agent has its own dashboard with hardware info, last keep-alive, active rules, and events. This is a good place to start investigating when something looks off.

## What broke

A few things that cost time:

- **Version mismatch** — agents 4.14.5, manager 4.12.0. The manager silently rejects registration with a vague "agent version must be lower or equal" error. Fix: always clone the matching repo tag.
- **Certificate generation** — the cert generator image `0.0.2` isn't compatible with 4.14.5. Version `0.0.4` downloads the correct cert tool from Wazuh's CDN. This is handled automatically if you use the right repo tag.
- **OpenSearch security not initialized** — after the first start, the security plugin needs to initialize before the dashboard can connect. It's automatic but takes 2-3 minutes. The dashboard shows "server not ready yet" until it finishes. Just wait.

## What's next

Now that the plumbing works, the interesting part starts:

- **Custom rules** — write detection rules for specific threats (SSH brute force patterns, unusual cron changes, etc.)
- **Active response** — configure Wazuh to block IPs automatically when they hit too many auth failures
- **Integrations** — send Wazuh alerts to a Telegram channel via the n8n automation stack I already run
- **Threat hunting** — use the dashboard to investigate something specific rather than waiting for alerts

The goal isn't to have a perfect setup. It's to have enough that I'm reading real logs every day and building intuition for what normal looks like — so that when something isn't normal, I notice.
