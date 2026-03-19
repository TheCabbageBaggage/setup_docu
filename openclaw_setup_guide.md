# 🦞 OpenClaw Setup Guide — CabbageBaggage Homelab

## Overview

OpenClaw is a self-hosted personal AI agent (Gateway + CLI) that connects to messaging platforms like Telegram. This guide documents the full setup process on an OMV homelab server.

- **Host:** `192.168.68.62`
- **Gateway Port:** `18789`
- **Telegram Bot:** `@clowie_claw_bot`

---

## Prerequisites

- Docker & Docker Compose installed (via OMV-Extras)
- Ollama running on port `11434`
- Existing Telegram bot token from `@BotFather`
- Anthropic API key from [console.anthropic.com](https://console.anthropic.com) *(optional, for cloud AI)*

---

## Chapter 1: Installation via Git

OpenClaw must be installed from the official repository. **Do not** use a hand-written `docker-compose.yml` — the repo's own `docker-setup.sh` handles everything.

```bash
cd /root
git clone https://github.com/openclaw/openclaw.git
cd openclaw
```

---

## Chapter 2: Running the Setup Script

Use the pre-built image to avoid a slow local build on low-end hardware:

```bash
export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
./docker-setup.sh
```

The script will:
1. Pull the Docker image
2. Fix config directory permissions
3. Start the gateway container
4. Launch the interactive onboarding wizard

---

## Chapter 3: Onboarding Wizard — Step by Step

### 3.1 Security Warning
Accept the security disclaimer. For a single-user homelab setup this is safe.

### 3.2 Onboarding Mode
Select **Quickstart**.

### 3.3 AI Provider
- **Option A — Anthropic (cloud):** Enter your `sk-ant-...` API key from [console.anthropic.com](https://console.anthropic.com).
  > ⚠️ Claude.ai Pro and the Anthropic API are **separate products** with separate billing. Your Pro subscription does not include API access.
- **Option B — Ollama (local, free):** Enter `http://192.168.68.62:11434` as the base URL.
  > ⚠️ Do **not** use `http://127.0.0.1:11434` — the container cannot reach the host loopback address.

### 3.4 Search Provider
Skip for now. Can be added later via Gemini (free tier at [aistudio.google.com](https://aistudio.google.com)).

### 3.5 Initial Skills
Skip for now. Skills can be installed later with:
```bash
docker compose run --rm openclaw-cli skills install <name>
```

### 3.6 Node Package Manager
Select **pnpm** (consistent with OpenClaw internals).

### 3.7 Homebrew
Skip. Not relevant on a Linux server.

### 3.8 Hooks
Recommended selections:
- ✅ `command-logger` — logs agent actions for debugging
- ✅ `session-memory` — agent retains context across sessions
- ❌ `boot-md` — not needed
- ❌ `bootstrap-extra-files` — not needed

### 3.9 Dashboard Ready
The wizard prints a dashboard link with your token:
```
http://127.0.0.1:18789/#token=<YOUR_TOKEN>
```
Save this token securely — it is also stored in `/root/openclaw/.env`.

---

## Chapter 4: Fixing the Gateway (LAN Access)

By default, OpenClaw blocks non-localhost access to the Control UI. Fix this for LAN access:

```bash
cd /root/openclaw

# Allow Host-header origin fallback (simpler fix for LAN)
docker compose run --rm openclaw-cli config set \
  gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback true

docker compose restart openclaw-gateway
```

---

## Chapter 5: Accessing the Dashboard

### Via SSH Tunnel (HTTP, no certificate needed)
On your local PC:
```bash
ssh -N -L 18789:127.0.0.1:18789 root@192.168.68.62
```
Then open in browser:
```
http://localhost:18789/#token=<YOUR_TOKEN>
```

### Via NPM + HTTPS (permanent setup)
In Nginx Proxy Manager, create a proxy host:
- **Domain:** `claw.cabbagebaggage.net`
- **Forward:** `192.168.68.62:18789`
- **SSL:** Use existing `*.cabbagebaggage.net` wildcard cert

Then update the allowed origins:
```bash
docker compose run --rm openclaw-cli config set \
  gateway.controlUi.allowedOrigins '["https://claw.cabbagebaggage.net"]'
docker compose restart openclaw-gateway
```

> ⚠️ OpenClaw requires HTTPS for WebSocket connections. The SSH tunnel workaround uses `localhost` which browsers treat as a secure context.

---

## Chapter 6: Device Pairing

On first browser visit you will see "pairing required". Approve your device:

```bash
cd /root/openclaw

# List pending devices
docker compose run --rm openclaw-cli devices list

# Approve by request ID
docker compose run --rm openclaw-cli devices approve <requestId>
```

---

## Chapter 7: Telegram Setup

### 7.1 Create a Bot
1. Open Telegram and message `@BotFather`
2. Send `/newbot`
3. Follow the prompts — save the token

### 7.2 Connect the Bot
```bash
cd /root/openclaw
docker compose run --rm openclaw-cli channels add \
  --channel telegram \
  --token "<YOUR_BOT_TOKEN>"
```

### 7.3 Add Your Telegram User ID to Allowlist
Find your numeric ID by messaging `@userinfobot` on Telegram, then:
```bash
docker compose run --rm openclaw-cli config set \
  channels.telegram.allowFrom '["<YOUR_TELEGRAM_USER_ID>"]'

docker compose restart openclaw-gateway
```

Now message `/start` to your bot on Telegram.

---

## Chapter 8: Connecting Ollama (Local AI)

After onboarding, point OpenClaw to your local Ollama instance:

```bash
docker compose run --rm openclaw-cli config set \
  models.providers.ollama.baseUrl "http://192.168.68.62:11434"
```

Recommended models for low-end hardware (i3-4330, 8GB RAM):
- `llama3.2:3b`
- `qwen2.5:3b`

Pull a model:
```bash
docker exec -it ollama ollama pull llama3.2:3b
```

---

## Chapter 9: Security Notes

- **Never expose port `18789` directly to the internet** without Cloudflare Access or equivalent
- **Keep the gateway token secret** — treat it like a password
- **Token is stored at:** `/root/openclaw/.env` — add to `.gitignore`
- Run periodic audits:
  ```bash
  docker compose run --rm openclaw-cli security audit --deep
  ```

---

## Chapter 10: Useful Commands

| Action | Command |
|--------|---------|
| Start gateway | `docker compose up -d openclaw-gateway` |
| Stop all | `docker compose down` |
| View logs | `docker logs openclaw-openclaw-gateway-1 --tail 50` |
| Get dashboard token | `docker compose run --rm openclaw-cli dashboard --no-open` |
| List devices | `docker compose run --rm openclaw-cli devices list` |
| Health check | `curl http://192.168.68.62:18789/healthz` |
| Update image | `docker compose pull && docker compose up -d` |
