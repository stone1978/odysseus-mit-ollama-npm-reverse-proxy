Odysseus AI Workspace – Installation mit Ollama + Nginx Proxy Manager (Reverse Proxy)
Diese Anleitung beschreibt die vollständige Installation und Konfiguration von Odysseus (Docker Compose) mit:

Ollama (läuft direkt auf dem Host als systemd-Service)
Nginx Proxy Manager (NPM) als Reverse Proxy (HTTPS-Termination)
Optional: NVIDIA GPU via NVIDIA Container Toolkit
Optional: SearXNG extern (z. B. https://search.stonecloud.at/)
Beispiel-Setup (aus dieser Doku):

Host-IP (LAN): 172.22.222.114
Domain: odysseus.stonecloud.at
Odysseus Port: 7000
Zusätzlich liegt im Repository die PDF-Version: odysseus_installation_guide.pdf.

Datenfluss / Architektur (Kurzüberblick)
Browser → https://odysseus.stonecloud.at → NPM → http://172.22.222.114:7000 → Odysseus Container
Odysseus Container → http://host.docker.internal:11434/v1 → Ollama auf Host (systemd)
Odysseus Container → http://chromadb:8000 → ChromaDB Container (intern)
Odysseus Container → http://searxng:8080 → SearXNG Container (intern) oder extern https://search.stonecloud.at/
Inhaltsverzeichnis
Voraussetzungen
Ollama für Docker erreichbar machen (systemd Override)
NVIDIA Container Toolkit (optional, für GPU)
Odysseus installieren (Repository + .env)
Odysseus starten
Admin-Login (Initialpasswort)
Nginx Proxy Manager (Reverse Proxy) einrichten
Ollama in Odysseus einbinden
SearXNG extern einbinden
Portainer-Hinweis & Updates
Referenz: Ports, Netzwerke, Pfade
1) Voraussetzungen
Linux (z. B. Ubuntu / Linux Mint)
Docker CE + Docker Compose Plugin (docker compose)
Git (git)
Nginx Proxy Manager bereits lauffähig
(Optional) NVIDIA GPU + Treiber am Host
(Optional) Portainer
2) Ollama für Docker erreichbar machen (systemd Override)
Wenn Ollama standardmäßig nur auf 127.0.0.1:11434 lauscht, ist es aus Docker-Containern nicht erreichbar. Lösung: OLLAMA_HOST per systemd Drop‑in setzen.

2.1 Drop-in Override-Datei anlegen
bash
Copy
sudo mkdir -p /etc/systemd/system/ollama.service.d

sudo tee /etc/systemd/system/ollama.service.d/override.conf >/dev/null <<'EOF'
[Service]
Environment=
Environment="OLLAMA_HOST=0.0.0.0:11434"
EOF
Wichtig: Die erste leere Zeile Environment= ist notwendig, um alte Environment-Werte zu löschen.

2.2 systemd neu laden & Ollama neu starten
bash
Copy
sudo systemctl daemon-reload
sudo systemctl restart ollama
2.3 Verifizieren
bash
Copy
sudo systemctl cat ollama | sed -n '1,200p'
sudo systemctl show ollama -p Environment
sudo ss -lntp | grep 11434 || true
Erwartung: ss zeigt :11434 bzw. 0.0.0.0:11434.

3) NVIDIA Container Toolkit (optional, für GPU)
Damit der Odysseus-Container die Host-GPU nutzen kann:

3.1 Tools installieren
bash
Copy
sudo apt-get update
sudo apt-get install -y curl gpg
3.2 NVIDIA Repo hinzufügen
bash
Copy
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --batch --yes --dearmor \
    -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
3.3 Toolkit installieren
bash
Copy
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
3.4 Docker Runtime konfigurieren
bash
Copy
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
3.5 GPU-Passthrough testen
bash
Copy
docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
4) Odysseus installieren (Repository + .env)
4.1 Zielverzeichnis anlegen
bash
Copy
sudo mkdir -p /opt/odysseus
sudo chown -R "$USER":"$USER" /opt/odysseus
cd /opt/odysseus
4.2 Repository klonen
bash
Copy
git clone https://github.com/pewdiepie-archdaemon/odysseus.git .
4.3 Branch wählen
Empfehlung für Produktion:

bash
Copy
git checkout main
4.4 .env erstellen
bash
Copy
cp .env.example .env
nano /opt/odysseus/.env
4.5 .env – Beispielkonfiguration (auf dein Setup angepasst)
bash
Copy
#############################################
# Odysseus - Environment (.env)
#############################################

########################
# Network / Reverse Proxy
########################
# Bind Odysseus Web UI auf der Host-LAN-IP
APP_BIND=172.22.222.114
APP_PORT=7000

########################
# Auth & Security
########################
AUTH_ENABLED=true
LOCALHOST_BYPASS=false

# HTTPS wird am NPM terminiert -> Cookies müssen Secure sein
SECURE_COOKIES=true

# CORS Origins
ALLOWED_ORIGINS=http://localhost,http://127.0.0.1,https://odysseus.stonecloud.at

# Admin (Passwort leer lassen = wird beim ersten Start generiert)
ODYSSEUS_ADMIN_USER=admin
ODYSSEUS_ADMIN_PASSWORD=

########################
# Ollama (läuft am Host)
########################
OLLAMA_BASE_URL=http://host.docker.internal:11434/v1
EMBEDDING_URL=http://host.docker.internal:11434/v1/embeddings
LLM_HOST=host.docker.internal
LLM_HOSTS=host.docker.internal,172.22.222.114

########################
# Web Search (SearXNG)
########################
SEARXNG_INSTANCE=https://search.stonecloud.at/

########################
# GPU (NVIDIA)
########################
# NVIDIA Compose-Overlay aktivieren
COMPOSE_FILE=docker-compose.yml:docker/gpu.nvidia.yml

########################
# Bundled Services (loopback-only)
########################
CHROMADB_BIND=127.0.0.1
NTFY_BIND=127.0.0.1
Hinweise (Konzept):

APP_BIND=172.22.222.114 macht Odysseus im LAN unter http://172.22.222.114:7000 erreichbar (wichtig für NPM Forward).
Mit SECURE_COOKIES=true erfolgt der Login zuverlässig über HTTPS via NPM.
5) Odysseus starten
bash
Copy
cd /opt/odysseus
docker compose up -d --build
Status prüfen:

bash
Copy
docker compose ps
6) Admin-Login (Initialpasswort)
Beim ersten Start wird ein temporäres Admin-Passwort erzeugt (wenn ODYSSEUS_ADMIN_PASSWORD leer ist).

bash
Copy
cd /opt/odysseus
docker compose logs odysseus | grep -i "Temporary password"
Login über HTTPS:

URL: https://odysseus.stonecloud.at
User: admin
Passwort: aus den Logs
7) Nginx Proxy Manager (Reverse Proxy) einrichten
In NPM: Proxy Hosts → Add Proxy Host

Details Tab
Domain Names: odysseus.stonecloud.at
Scheme: http
Forward Hostname / IP: 172.22.222.114
Forward Port: 7000
Websockets Support: ON
Block Common Exploits: ON (empfohlen)
SSL Tab
SSL Certificate: Let’s Encrypt (oder bestehendes Zertifikat)
Force SSL: ON
HTTP/2 Support: ON (empfohlen)
Advanced Tab (empfohlen)
nginx
Copy
proxy_read_timeout 300s;
proxy_send_timeout 300s;
proxy_buffering off;
8) Ollama in Odysseus einbinden
8.1 Optional: Erreichbarkeit aus Docker testen
bash
Copy
docker run --rm --network odysseus_default curlimages/curl:8.8.0 \
  -sS http://host.docker.internal:11434/api/tags | head
8.2 In Odysseus hinzufügen
Odysseus UI: Settings → Add Models → Add Local Models

Typ: LLM
Endpoint: http://host.docker.internal:11434/v1
Dann Test und Add.

9) SearXNG extern einbinden
Zwei saubere Wege:

9.1 Über die UI
Odysseus: Settings → Search → SearXNG-URL auf https://search.stonecloud.at/ setzen.

9.2 Über .env + docker-compose.yml (dauerhaft)
Damit der Wert aus .env sicher verwendet wird, ändere im docker-compose.yml beim odysseus-Service:

Von:

yaml
Copy
- SEARXNG_INSTANCE=http://searxng:8080
Zu:

yaml
Copy
- SEARXNG_INSTANCE=${SEARXNG_INSTANCE:-http://searxng:8080}
Dann:

bash
Copy
cd /opt/odysseus
docker compose up -d
10) Portainer-Hinweis & Updates
Wenn der Stack via CLI erstellt wurde, zeigt Portainer ggf. Limited Control. Das ist normal.

Update-Workflow
bash
Copy
cd /opt/odysseus
git pull
docker compose up -d --build
11) Referenz: Ports, Netzwerke, Pfade
Ports (Beispiel)
Odysseus: 172.22.222.114:7000
Ollama (Host): 0.0.0.0:11434
Docker-Netzwerke
Standard: bridge (172.17.0.0/16)
Odysseus: odysseus_default (intern)
Wichtige Pfade
Odysseus: /opt/odysseus/
Konfiguration: /opt/odysseus/.env
Compose: /opt/odysseus/docker-compose.yml
Ollama Override: /etc/systemd/system/ollama.service.d/override.conf
