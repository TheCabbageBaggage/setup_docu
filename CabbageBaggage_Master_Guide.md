# 📦 CabbageBaggage: The Ultimate Home Lab Guide (2026)

This guide documents the transition from a basic OMV setup to a professional, domain-secured AI, Cloud, and Network infrastructure.

---

## 🛠️ Chapter 1: Hardware & Base System
* **Host:** 2014 Desktop PC, Intel i3-4330 @ 3,50 GHz | 8GB RAM | 250GB SSD
* **OS:** OpenMediaVault (OMV) 8.
* **Storage:** External HDD/SSD for data, formatted as `ext4`.
* **Networking:** TP-Link Deco Mesh System (Static IP for Server: `192.168.68.62`).

---

## 💾 Chapter 2: OMV Setup & Extensions
Before deploying containers, we need to prepare the environment.

### 2.1 Installing OMV-Extras & Docker
1. **Run the install script via SSH:**
   ```bash
   wget -O - https://github.com/OpenMediaVault-Plugin-Developers/installScript/raw/master/install | sudo bash
   ```
2. **Enable Plugins:** In the OMV WebUI, go to **System > Plugins** and install `openmediavault-compose`.
3. **Set Directory Permissions:** To prevent persistence issues, ensure the Docker user has ownership:
   ```bash
   # Replace YOUR_UUID with your actual disk UUID
   sudo chown -R 1000:1000 /srv/dev-disk-by-uuid-YOUR_UUID/appdata
   sudo chmod -R 775 /srv/dev-disk-by-uuid-YOUR_UUID/appdata
   ```

---

## 🚀 Chapter 3: Docker Services Configuration
Each service is deployed via **Services > Compose > Files** in OMV. All services share a common network: `cabbage-net`.

### 3.1 Nginx Proxy Manager (NPM)
The \"Traffic Controller\" for SSL and subdomains.
```yaml
services:
  npm:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: nginx-proxy-manager
    restart: unless-stopped
    ports:
      - '80:80'
      - '81:81'
      - '443:443'
    volumes:
      - /srv/dev-disk-by-uuid-YOUR_UUID/appdata/npm/data:/data
      - /srv/dev-disk-by-uuid-YOUR_UUID/appdata/npm/letsencrypt:/etc/letsencrypt
    networks:
      - cabbage-net

networks:
  cabbage-net:
    driver: bridge
```

---

### 3.2 Nextcloud & MariaDB
Your private cloud for files, contacts, and calendars.
```yaml
services:
  nextcloud-db:
    image: mariadb:10.11
    container_name: nextcloud-db
    restart: unless-stopped
    command: --transaction-isolation=READ-COMMITTED --binlog-format=ROW
    volumes:
      - /srv/dev-disk-by-uuid-YOUR_UUID/appdata/nextcloud/db:/var/lib/mysql
    environment:
      - MARIADB_ROOT_PASSWORD=YOUR_DB_ROOT_PASSWORD
      - MARIADB_PASSWORD=YOUR_DB_PASSWORD
      - MARIADB_DATABASE=nextcloud
      - MARIADB_USER=nextcloud
    networks:
      - cabbage-net

  nextcloud:
    image: nextcloud:latest
    container_name: nextcloud
    restart: unless-stopped
    ports:
      - '8080:80'
    depends_on:
      - nextcloud-db
    volumes:
      - /srv/dev-disk-by-uuid-YOUR_UUID/appdata/nextcloud/data:/var/www/html
    environment:
      - MYSQL_PASSWORD=YOUR_DB_PASSWORD
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_HOST=nextcloud-db
    networks:
      - cabbage-net
```

---

### 3.3 Pi-hole
Network-wide ad-blocking and DNS management.
```yaml
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    restart: unless-stopped
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "8081:80" # Web UI moved to 8081 because NPM uses 80
    environment:
      - TZ=Europe/Berlin
      - WEBPASSWORD=YOUR_ADMIN_PASSWORD
    volumes:
      - /srv/dev-disk-by-uuid-YOUR_UUID/appdata/pihole/config:/etc/pihole
      - /srv/dev-disk-by-uuid-YOUR_UUID/appdata/pihole/dnsmasq:/etc/dnsmasq.d
    networks:
      - cabbage-net
```

---

### 3.4 Vaultwarden
Your private, self-hosted password manager.
```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    environment:
      - SIGNUPS_ALLOWED=true # Set to false after creating your account
    volumes:
      - /srv/dev-disk-by-uuid-YOUR_UUID/appdata/vaultwarden:/data
    networks:
      - cabbage-net
```

---

### 3.5 Ollama (AI Backend)
Handles the local execution of Large Language Models.
```yaml
services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    volumes:
      - /srv/dev-disk-by-uuid-lkohl/appdata/ollama:/root/.ollama
    ports:
      - "11434:11434"
```

---

### 3.6 Open WebUI (AI Frontend)
The user interface for your local AI.
```
# OpenClaw — Docker Compose Configuration
# Part of CabbageBaggage Homelab Stack
# Full setup guide: see openclaw_setup_guide.md
#
# ⚠️  This file contains NO secrets.
#     Secrets (gateway token, API keys, bot token) live in .env — never commit .env to git.
#
# Usage:
#   cd /root/openclaw
#   export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
#   ./docker-setup.sh          # First-time setup (runs onboarding wizard)
#   docker compose up -d       # Subsequent starts

version: '3.8'

services:

  openclaw-gateway:
    image: ${OPENCLAW_IMAGE:-ghcr.io/openclaw/openclaw:latest}
    container_name: openclaw-openclaw-gateway-1
    restart: unless-stopped
    network_mode: "host"   # Required: OpenClaw binds to LAN interface
    environment:
      - OPENCLAW_GATEWAY_BIND=lan
      # OPENCLAW_GATEWAY_TOKEN and provider API keys are injected via .env (auto-generated by docker-setup.sh)
    volumes:
      - ~/.openclaw:/home/node/.openclaw          # Config, memory, sessions
      - ~/openclaw/workspace:/home/node/.openclaw/workspace   # Agent workspace / files
    healthcheck:
      test: ["CMD", "node", "-e", "fetch('http://127.0.0.1:18789/healthz').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"]
      interval: 30s
      timeout: 5s
      start_period: 15s

  openclaw-cli:
    image: ${OPENCLAW_IMAGE:-ghcr.io/openclaw/openclaw:latest}
    container_name: openclaw-cli
    network_mode: "service:openclaw-gateway"  # Shares network namespace with gateway
    volumes:
      - ~/.openclaw:/home/node/.openclaw
      - ~/openclaw/workspace:/home/node/.openclaw/workspace
    entrypoint: ["node", "dist/index.js"]
    profiles:
      - cli   # Only starts when explicitly called: docker compose run --rm openclaw-cli <command>
    depends_on:
      - openclaw-gateway

# Notes:
# - No PostgreSQL needed — OpenClaw stores state in ~/.openclaw (JSON files)
# - No socat relay needed — network_mode host exposes port 18789 directly
# - Dashboard: http://192.168.68.62:18789  (or via SSH tunnel for HTTPS requirement)
# - Telegram bot: @clowie_claw_bot
# - Ollama URL (set via CLI): http://192.168.68.62:11434
```

### 3.7 Cloudflare DDNS
Updates your home IP address automatically for your domain.
```yaml
services:
  cloudflare-ddns:
    image: oznu/cloudflare-ddns:latest
    container_name: cloudflare-ddns
    restart: unless-stopped
    environment:
      - API_KEY=YOUR_CLOUDFLARE_API_TOKEN
      - ZONE=cabbagebaggage.net
      - PROXIED=true
    networks:
      - cabbage-net
```
### 3.8 OpenClaw
Runs OpenClaw on your private data
```yaml
version: '3.8'

services:
  openclaw:
    image: ghcr.io/openclaw/openclaw:latest
    container_name: openclaw-app
    restart: always
    network_mode: "host"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://claw_user:claw_password@127.0.0.1:5432/openclaw
      # Wir nutzen hier die lokale IP statt der Domain
      - BACKEND_URL=http://192.168.68.62:3005 
    volumes:
      - /sharedfolders/AppData/openclaw/config:/home/node/.openclaw

  # Das Socat-Relay macht die App im LAN/WLAN unter Port 3005 sichtbar
  relay:
    image: alpine/socat
    container_name: openclaw-relay
    restart: always
    network_mode: "host"
    # Lauscht auf ALLEN lokalen Schnittstellen am Port 3005
    command: tcp-listen:3005,fork,reuseaddr tcp-connect:127.0.0.1:18789
    depends_on:
      - openclaw

  db:
    image: postgres:15-alpine
    container_name: openclaw-db
    restart: always
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_USER=claw_user
      - POSTGRES_PASSWORD=claw_password
      - POSTGRES_DB=openclaw
    volumes:
      - /sharedfolders/AppData/openclaw/db:/var/lib/postgresql/data
---
```
## 🌐 Chapter 4: Domain & Networking Setup

### 4.1 Cloudflare DNS
1. **Nameservers:** Point your GoDaddy domain nameservers to Cloudflare.
2. **A-Record:** Create a record for `@` pointing to `1.1.1.1` (proxied).
3. **CNAMEs:** Create records for `ai`, `cloud`, `pass`, and `pihole` pointing to `@`.

### 4.2 Deco Router Settings (NAT Forwarding)
Forward the following ports to your ThinkPad (`192.168.68.62`):
* **External 80** -> **Internal 80**
* **External 443** -> **Internal 443**

---

## 🛡️ Chapter 5: Security & Subdomains (NPM)
1. **SSL Certificate:** Add a \"Let's Encrypt\" certificate using **DNS Challenge** (Cloudflare) for `*.cabbagebaggage.net`.
2. **Proxy Hosts:**
   * `ai.cabbagebaggage.net` -> `open-webui:8080`
   * `cloud.cabbagebaggage.net` -> `nextcloud:
   * `pass.cabbagebaggage.net` -> `vaultwarden:80`
   * `pihole.cabbagebaggage.net` -> `pihole:80`

---

## 🧠 Chapter 6: AI Automation & Multi-User RAG

### 6.1 Automating Embeddings
Pull the embedding model manually to enable Document Search (RAG):
```bash
docker exec -it ollama ollama pull mxbai-embed-large
```
In Open WebUI **Settings > Documents**, select `mxbai-embed-large` as the \"Embedding Model\".

### 6.2 Nextcloud Assistant
Connect the Nextcloud Assistant app to your Ollama API (`http://ollama:11434`). This allows each user to chat with their own files privately.

---

## 🖨️ Chapter 7: Troubleshooting (The Printer Ghost)
If your HP LaserJet prints garbled text:
1. Access the printer's Web Interface via its IP.
2. Disable **SNMP**, **WSD**, and **LPD** services.
3. This stops random network discovery scans from being interpreted as print jobs.
