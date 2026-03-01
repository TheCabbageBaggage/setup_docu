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
  landingpage:
    image: nginx:stable-alpine
    container_name: landingpage-service
    restart: unless-stopped
    ports:
      - "3000:80"  # Deine Seite ist dann über Port 8080 erreichbar
    volumes:
      # Ersetze den Pfad links vom Doppelpunkt durch deinen absoluten OMV-Pfad
      - /www:/usr/share/nginx/html:ro
```

---

### 3.2 Nextcloud & MariaDB
Your private cloud for files, contacts, and calendars.
```yaml
services:
  nextcloud-db:
    image: mariadb:10.6
    restart: always
    command: --transaction-isolation=READ-COMMITTED --binlog-format=ROW
    volumes:
      - /nextcloud/nextcloud_db:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=secure_mysqlroot_password
      - MYSQL_PASSWORD=secure_mysql_password
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud

  nextcloud-app:
    image: nextcloud:latest
    restart: always
    ports:
      - 8080:80
    depends_on:
      - nextcloud-db
    volumes:
      - /nextcloud/nextcloud_config:/var/www/html
      - /nextcloud/nextcloud_data:/var/www/html/data
    environment:
      - MYSQL_HOST=nextcloud-db
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_PASSWORD=secure_mysql_password
      - NEXTCLOUD_ADMIN_USER=admin
      - NEXTCLOUD_ADMIN_PASSWORD=secure_admin_password
      - NEXTCLOUD_TRUSTED_DOMAINS=cloud.lkohl.duckdns.org 192.168.68.62
```

---

### 3.3 Pi-hole
Network-wide ad-blocking and DNS management.
```yaml
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "8083:80/tcp" # Web-Oberfläche auf Port 8083
    environment:
      TZ: 'Europe/Berlin'
      WEBPASSWORD: 'lJ2&0Zk6Chv4IPi!' # Passwort für das Pi-hole Admin Panel
      FTLCONF_LOCAL_IPV4: '192.168.68.62'
    volumes:
      - '/srv/dev-disk-by-uuid-lkohl/appdata/pihole/config:/etc/pihole'
      - '/srv/dev-disk-by-uuid-lkohl/appdata/pihole/dnsmasq.d:/etc/dnsmasq.d'
    cap_add:
      - NET_ADMIN # Erforderlich für DHCP und Netzwerk-Manipulation
    restart: unless-stopped
```

---

### 3.4 Vaultwarden
Your private, self-hosted password manager.
```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: always
    environment:
      - SIGNUPS_ALLOWED=false   # Sobald dein Account erstellt ist, auf 'false' setzen!
    volumes:
      - /vaultwarden/:/data    # Hier wird dein Pfad auf /dev/sda1 genutzt
    ports:
      - 8081:80
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
```yaml
services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    restart: unless-stopped
    ports:
      - "3001:8080" # Web-Oberfläche auf Port 3001
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434/
    volumes:
      - /srv/dev-disk-by-uuid-lkohl/appdata/open-webui:/app/data
      - /nextcloud/nextcloud_data/'Linus Kohl'/files:/app/nextcloud_files:ro
    depends_on:
      - ollama
```

---

### 3.7 Cloudflare DDNS
Updates your home IP address automatically for your domain.
```yaml
services:
  cloudflare-ddns:
    image: oznu/cloudflare-ddns:latest
    container_name: cloudflare-ddns
    restart: unless-stopped
    environment:
      - API_KEY=l1E6SGvNXt0w1NVIh87mNJICFJpHx4wg5SKXHT3B
      - ZONE=cabbagebaggage.net
      - PROXIED=true # Das aktiviert den Cloudflare-Schutz (orange Wolke)
```
---

### 3.8 Obsidian
My personal note server
```yaml
services:
  obsidian:
    image: lscr.io/linuxserver/obsidian:latest
    container_name: obsidian
    privileged: true # Oft nötig für die Fuse-Dateisystem-Interaktionen von Obsidian
    environment:
      - PUID=1000 # Ersetze dies durch die ID deines OMV-Nutzers
      - PGID=100 # Meistens 'users' Gruppe in OMV
      - TZ=Europe/Berlin
      - DOCKER_MODS=linuxserver/mods:universal-package-install # Optional für Zusatzpakete
      # --- ABSICHERUNG START ---
      - CUSTOM_USER=dein_name        # Dein gewünschter Benutzername
      - PASSWORD=dein_starkes_pw     # Dein Passwort für den Web-Login
      - AUTH_CONFIG=                 # Leer lassen, um Standard-Auth zu erzwingen
      # --- ABSICHERUNG ENDE ---
    volumes:
      - /sharedfolders/AppData/obsidian/config:/config
      - /sharedfolders/AppData/obsidian/vaults:/vaults
    ports:
      - 3002:3000 # HTTP Web-Interface (KasmVNC)
      - 3003:3001 # HTTPS Web-Interface
    restart: unless-stopped
```
---

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
   * `cloud.cabbagebaggage.net` -> `nextcloud:80`
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
