# DidattixVM
# Project Work – Infrastruttura su VM `registro`

Questo repository documenta in modo tecnico, formale e completo l’infrastruttura realizzata per il project work: una macchina virtuale Ubuntu Server 22.04.5 LTS eseguita su VirtualBox, configurata in modo sicuro, con uno stack Docker che include servizi web, database, monitoring, logging, LLM e un’API REST di monitoring.  
Sono inclusi anche firewall, script di monitoraggio, backup automatico e cronjob notturni.

---

## 1. Obiettivi

- Creare una VM Linux dedicata, stabile e riproducibile, con Ubuntu Server 22.04.5 LTS.  
- Configurare accesso remoto sicuro solo tramite chiave SSH.  
- Installare Docker Engine e Docker Compose v2 e utilizzarli per orchestrare i servizi.  
- Eseguire container per:
  - Web server PHP + Apache
  - Database MySQL + phpMyAdmin
  - Stack di osservabilità (Prometheus, Node Exporter, Loki, Grafana, Promtail)
  - Stack LLM (Ollama + Open WebUI)
  - API REST di monitoring sviluppata in Node.js  
- Aggiungere firewall UFW, script di monitoraggio periodico, backup automatico e generazione di report via cron.

---

## 2. Infrastruttura VirtualBox

### 2.1 Specifiche della VM

- Hypervisor: **Oracle VirtualBox**
- Nome macchina: **`registro`**
- Sistema operativo guest: **Ubuntu Server 22.04.5 LTS (64‑bit)**
- RAM: **16 GB (16384 MB)**
- CPU: **8 vCPU**
- Disco: **VDI dinamico**, dimensione consigliata ≥ 40–60 GB
- Memoria video: **80 MB**
- Scheda grafica: **VMSVGA**, **accelerazione 3D** abilitata

Dati di sistema (`hostnamectl`):

- Static hostname: `registro`
- Operating System: Ubuntu 22.04.5 LTS
- Kernel: Linux 5.15.0-161-generic
- Architecture: x86‑64
- Virtualization: oracle
- Hardware Vendor: innotek GmbH
- Hardware Model: VirtualBox

Indirizzo IP server (rete LAN): **10.10.13.244**

### 2.2 Configurazione rete VirtualBox

Scheda **Rete → Scheda 1**:

- Modalità di collegamento: **Scheda con bridge**
- Interfaccia host: scheda fisica principale (es. *Realtek Gaming 2.5GbE Family Controller*)
- Tipo scheda: **Intel PRO/1000 MT Desktop**
- Modalità promiscua: **Permetti tutto**
- Cavo virtuale: **collegato**

### 2.3 Creazione VM (riassunto step)

1. Avviare VirtualBox, cliccare **Nuova**.  
2. Nome: `registro`, Tipo: `Linux`, Versione: `Ubuntu (64-bit)`.  
3. Assegnare 16384 MB di RAM e 8 vCPU.  
4. Creare un disco **VDI dinamico** (≥ 40–60 GB).  
5. In **Sistema**, abilitare I/O APIC e virtualizzazione hardware.  
6. In **Schermo**, impostare 80 MB video, VMSVGA, accelerazione 3D.  
7. In **Rete**, impostare la scheda in bridge come sopra.  
8. In **Archiviazione**, montare l’ISO di Ubuntu Server 22.04.5 LTS come unità ottica.

---

## 3. Installazione di Ubuntu Server

1. Avviare la VM `registro` dall’ISO.  
2. Dal menu di boot scegliere **Install Ubuntu Server**.  
3. Configurare lingua, layout tastiera e fuso orario.  
4. Configurare la rete (DHCP o IP statico, in produzione utilizzato `10.10.13.244`).  
5. Partizionare il disco scegliendo l’opzione guidata **“Use entire disk”** sul VDI.  
6. Creare l’utente amministrativo (es. `admin`) e impostare l’hostname **`registro`**.  
7. Nella sezione dei servizi aggiuntivi, selezionare **OpenSSH server**.  
8. Completare l’installazione, riavviare la VM e rimuovere l’ISO.  

Verifica:

hostnamectl
uname -a
lsb_release -a

text

---

## 4. Configurazione rete sulla VM

Verifica IP:

ip a

text

Nel progetto la macchina è raggiungibile all’indirizzo:

10.10.13.244

text

Se necessario, l’IP può essere reso statico modificando il file Netplan in `/etc/netplan/`.

---

## 5. Aggiornamento sistema e strumenti base

Subito dopo l’installazione:

sudo apt update
sudo apt upgrade -y
sudo apt install vim htop curl git -y

text

---

## 6. Gestione utenti e sicurezza SSH

### 6.1 Utente di sistema e gruppi

Utente principale:

- **`registro`**

Estratto utenti di sistema (solo rilevanti): `root`, account di servizio standard (daemon, www-data, syslog, sshd, lxd, dnsmasq, vnstat, postfix, prometheus, ollama, …), più l’utente interattivo `registro`.

Gruppi dell’utente `registro`:

registro adm cdrom sudo dip plugdev lxd docker ollama

text

Significato:

- `sudo`: amministrazione completa della macchina.
- `docker`: esecuzione e gestione container senza `sudo`.
- `lxd`: gestione eventuali container LXD.
- `adm`: accesso ai log di sistema.

### 6.2 Directory `.ssh` e permessi

ls -ld /home/registro/.ssh

drwx------ 2 registro registro 4096 Dec 5 08:59 /home/registro/.ssh
text

Contenuto tipico:

-rw------- authorized_keys
-rw------- known_hosts
-rw-r--r-- known_hosts.old

text

- Directory `.ssh` con permessi `700`.  
- File `authorized_keys` con permessi `600`.

### 6.3 Autenticazione SSH a chiave

Sul client amministrativo:

ssh-keygen -t ed25519 -C "registro"

oppure:
ssh-keygen -t rsa -b 4096 -C "registro"
text

Copia della chiave pubblica verso la VM:

ssh-copy-id -i ~/.ssh/id_ed25519.pub registro@10.10.13.244

text

Oppure, manualmente:

mkdir -p ~/.ssh
chmod 700 ~/.ssh
echo "CONTENUTO_CHIAVE_PUBBLICA" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

text

### 6.4 Hardening del demone SSH

Modificare `/etc/ssh/sshd_config`:

PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin no

text

Validare e riavviare:

sudo sshd -t # nessun output = configurazione valida
sudo systemctl restart ssh

text

Verifica delle porte:

sudo ss -tulpn | grep ssh

tcp LISTEN 0 128 0.0.0.0:22
tcp LISTEN 0 128 [::]:22
text

Test finale di accesso:

ssh -i ~/.ssh/id_ed25519 registro@10.10.13.244

text

---

## 7. Firewall – UFW

### 7.1 Installazione e abilitazione

sudo apt install ufw -y
sudo ufw enable

text

### 7.2 Regole configurate

Consenti SSH
sudo ufw allow OpenSSH

Blocca servizi non utilizzati
sudo ufw deny 23 # Telnet
sudo ufw deny 3389 # RDP

Verifica configurazione
sudo ufw status numbered

text

Stato risultante:

| Servizio | Porta | Azione |
|----------|-------|--------|
| SSH      | 22    | ALLOW  |
| Telnet   | 23    | DENY   |
| RDP      | 3389  | DENY   |
| Altro    | –     | DENY (default) |

---

## 8. Installazione Docker e Docker Compose

### 8.1 Installazione Docker Engine

sudo apt update
sudo apt install ca-certificates curl gnupg lsb-release -y

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg |
sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg]
https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" |
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

text

Abilitare e avviare il servizio:

sudo systemctl enable docker
sudo systemctl start docker
systemctl status docker

text

Il servizio `docker.service` risulta **active (running)** con processo principale `dockerd` e vari processi `docker-proxy` per le porte mappate.

### 8.2 Versioni Docker / Compose

docker --version
docker compose version

text

- Docker: `Docker version 28.2.2, build 28.2.2-0ubuntu1~22.04.1`
- Docker Compose: `Docker Compose version v2.40.3`

### 8.3 Utilizzo di Docker senza `sudo`

sudo usermod -aG docker $USER

logout / login
docker run --rm hello-world

text

---

## 9. Stack Docker in esecuzione

### 9.1 Contenitori attivi

Output sintetico di `docker ps`:

- **Applicazione web + DB**
  - `sito-docker-web-1` – Image `php:8.3-apache`  
    - Porte: `0.0.0.0:8080->80/tcp`, `[::]:8080->80/tcp`
  - `sito-docker-db-1` – Image `mysql:8`  
    - Porte interne: `3306/tcp`, `33060/tcp`
  - `sito-docker-phpmyadmin-1` – Image `phpmyadmin/phpmyadmin`  
    - Porte: `0.0.0.0:8081->80/tcp`, `[::]:8081->80/tcp`

- **LLM**
  - `llm` – Image `ollama/ollama:latest`  
    - Porte: `0.0.0.0:11434->11434/tcp`, `[::]:11434->11434/tcp`
  - `webui` – Image `ghcr.io/open-webui/open-webui:main`  
    - Porte: `0.0.0.0:3500->8080/tcp`, `[::]:3500->8080/tcp`

- **Monitoring / Logging**
  - `prometheus` – Image `prom/prometheus:latest`  
    - Porte: `0.0.0.0:9090->9090/tcp`, `[::]:9090->9090/tcp`
  - `node-exporter` – Image `prom/node-exporter:latest`  
    - Porte: `0.0.0.0:9100->9100/tcp`, `[::]:9100->9100/tcp`
  - `loki` – Image `grafana/loki:2.8.2`  
    - Porte: `0.0.0.0:3100->3100/tcp`, `[::]:3100->3100/tcp`
  - `grafana` – Image `grafana/grafana:latest`  
    - Porte: `0.0.0.0:3000->3000/tcp`, `[::]:3000->3000/tcp`

### 9.2 Immagini e reti Docker

Immagini (`docker images`) principali:

- `php:8.3-apache`
- `mysql:8`
- `phpmyadmin/phpmyadmin:latest`
- `ghcr.io/open-webui/open-webui:main`
- `ollama/ollama:latest`
- `grafana/grafana:latest`
- `grafana/loki:2.8.2`
- `grafana/promtail:latest`
- `prom/prometheus:latest`
- `prom/node-exporter:latest`

Reti (`docker network ls`):

- `sito-docker_default` – bridge per web + db + phpMyAdmin
- `llm-docker_default` – bridge per Ollama + Open WebUI
- `02-docker_default` – bridge per monitoring/logging/LLM
- `bridge`, `host`, `none` – reti standard Docker

---

## 10. File `docker-compose.yml` (schema logico)

version: "3.9"

services:
web:
image: php:8.3-apache
container_name: sito-docker-web-1
ports:
- "8080:80"
depends_on:
- db
networks:
- sito

db:
image: mysql:8
container_name: sito-docker-db-1
environment:
- MYSQL_ROOT_PASSWORD=changeme
- MYSQL_DATABASE=registro
volumes:
- db_data:/var/lib/mysql
networks:
- sito

phpmyadmin:
image: phpmyadmin/phpmyadmin:latest
container_name: sito-docker-phpmyadmin-1
ports:
- "8081:80"
environment:
- PMA_HOST=db
depends_on:
- db
networks:
- sito

ollama:
image: ollama/ollama:latest
container_name: llm
ports:
- "11434:11434"
networks:
- llm

open-webui:
image: ghcr.io/open-webui/open-webui:main
container_name: webui
ports:
- "3500:8080"
depends_on:
- ollama
networks:
- llm

prometheus:
image: prom/prometheus:latest
container_name: prometheus
ports:
- "9090:9090"
volumes:
- ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
networks:
- monitor

node-exporter:
image: prom/node-exporter:latest
container_name: node-exporter
ports:
- "9100:9100"
networks:
- monitor

loki:
image: grafana/loki:2.8.2
container_name: loki
ports:
- "3100:3100"
volumes:
- ./loki-config.yaml:/etc/loki/loki.yaml:ro
- ./loki-data:/loki
- ./loki-data/wal:/wal
networks:
- monitor

promtail:
image: grafana/promtail:latest
container_name: promtail
volumes:
- ./promtail-config.yaml:/etc/promtail/config.yml:ro
- /var/log:/var/log:ro
- /home/registro/project-work:/project-work:ro
networks:
- monitor

grafana:
image: grafana/grafana:latest
container_name: grafana
ports:
- "3000:3000"
volumes:
- ./grafana-data:/var/lib/grafana
- ./grafana-provisioning:/etc/grafana/provisioning
networks:
- monitor

networks:
sito:
llm:
monitor:

volumes:
db_data:

text

---

## 11. Configurazione Prometheus / Loki / Promtail / Grafana

- **Prometheus**  
  File: `/home/registro/project-work/02-docker/prometheus.yml`  
  Montato in Compose come:  
  `./prometheus.yml:/etc/prometheus/prometheus.yml:ro`  
  Contiene i job di scraping (es. `node-exporter`, `prometheus`, altri target).

- **Loki**  
  File: `loki-config.yaml`  
  Montato come:  
  `./loki-config.yaml:/etc/loki/loki.yaml:ro`  
  Volumi: `./loki-data:/loki`, `./loki-data/wal:/wal` per la persistenza dei log.

- **Promtail**  
  File: `promtail-config.yaml`  
  Montato come:  
  `./promtail-config.yaml:/etc/promtail/config.yml:ro`  
  Volumi aggiuntivi:  
  - `/var/log:/var/log:ro`  
  - `/home/registro/project-work:/project-work:ro`  
  Promtail raccoglie log di sistema e log del project work e li inoltra a Loki.

- **Grafana**  
  Directory: `grafana-data/`, `grafana-provisioning/`  
  - `grafana-data/grafana.db` contiene configurazioni e dashboard in SQLite.  
  - `grafana-provisioning/` contiene eventuali datasources e provisioning.  

---

## 12. Monitoraggio periodico (script + cron)

### 12.1 Struttura directory

/opt/project/
├── scripts/
│ ├── monitor.sh
│ └── generate-report.sh
├── logs/
│ └── system.log
└── reports/
└── report-YYYY-MM-DD.txt

text

Creazione:

sudo mkdir -p /opt/project/scripts /opt/project/logs /opt/project/reports
sudo chown -R registro:registro /opt/project

text

### 12.2 Script `monitor.sh`

#!/bin/bash

LOGFILE="/opt/project/logs/system.log"

echo "=== $(date) ===" >> "$LOGFILE"

echo "CPU load:" >> "$LOGFILE"
uptime >> "$LOGFILE"

echo "RAM:" >> "$LOGFILE"
free -h >> "$LOGFILE"

echo "Disk usage:" >> "$LOGFILE"
df -h >> "$LOGFILE"

echo "Network usage:" >> "$LOGFILE"
ifconfig >> "$LOGFILE"

echo -e "\n\n" >> "$LOGFILE"

text

Permessi:

sudo chmod +x /opt/project/scripts/monitor.sh

text

### 12.3 Cronjob ogni 10 minuti

sudo crontab -e

text

Aggiungere:

*/10 * * * * /opt/project/scripts/monitor.sh

text

Ogni 10 minuti vengono registrate in `/opt/project/logs/system.log` le informazioni di CPU, RAM, disco e rete.

---

## 13. Automazione notturna: report + email

### 13.1 Script `generate-report.sh`

#!/bin/bash

REPORT_DIR="/opt/project/reports"
LOGFILE="/opt/project/logs/system.log"

mkdir -p "$REPORT_DIR"

REPORT="${REPORT_DIR}/report-$(date +%F).txt"

echo "REPORT GENERATO IL: $(date)" > "$REPORT"
echo "" >> "$REPORT"

tail -n 50 "$LOGFILE" >> "$REPORT"

text

Permessi:

sudo chmod +x /opt/project/scripts/generate-report.sh

text

### 13.2 Cronjob notturno (02:00)

sudo crontab -e

text

Aggiungere:

0 2 * * * /opt/project/scripts/generate-report.sh

text

Ogni notte alle 02:00 viene creato un file `report-YYYY-MM-DD.txt` con le ultime 50 righe del log di sistema.

### 13.3 Invio email del report (opzionale)

Installare il client mail:

sudo apt install mailutils -y

text

Aggiungere in fondo a `generate-report.sh`:

MAIL_TO="email_destinatario@example.com"
mail -s "Report giornaliero VM registro" "$MAIL_TO" < "$REPORT"

text

---

## 14. Backup automatico (bonus “Backup automatico”)

### 14.1 Script di backup

Percorso:

/home/registro/project-work/backup/backup.sh

text

Struttura:

/home/registro/project-work/backup/
├── backup.sh
├── logs/
│ └── cron.log
└── archive/
└── backup-YYYYMMDD.tar.gz (esempi)

text

Lo script `backup.sh` crea archivi compressi nelle directory `archive/` contenenti file di configurazione, log e altri artefatti del project work. Gli output vengono registrati in `backup/logs/cron.log`.

Permessi:

chmod +x /home/registro/project-work/backup/backup.sh

text

### 14.2 Cronjob di backup (01:00)

sudo crontab -e

text

Riga attiva:

0 1 * * * /home/registro/project-work/backup/backup.sh >> /home/registro/project-work/backup/logs/cron.log 2>&1

text

---

## 15. API REST di monitoring (bonus “API monitoring”)

### 15.1 API Node.js

L’API REST di monitoring è implementata in Node.js ed eseguita in background tramite PM2.

- Percorso file server:  
  `/home/registro/project-work/backend/src/server.js`
- Porta di ascolto: **3443**
- Comando del processo:  
  `node /home/registro/project-work/backend/src/server.js`

Gestione con PM2:

pm2 list
pm2 info monitoring-api
pm2 restart monitoring-api
pm2 logs monitoring-api

text

Questa API fornisce endpoint REST per esporre informazioni di stato / metriche del sistema e soddisfa il bonus “API monitoring”.  
L’eventuale file `api.py` in `/home/registro/project-work/04-monitoring/scripts/api.py` è solo sperimentale e non utilizzato in produzione.

---

## 16. Comandi operativi principali

- Avvio stack Docker:

docker compose up -d

text

- Arresto stack:

docker compose down

text

- Elenco container:

docker ps

text

- Log di un container:

docker logs -f nome_container

text

- Stato Docker:

systemctl status docker

text

- Stato firewall:

sudo ufw status numbered

text

- Cronjob di root:

sudo crontab -l

text

- Gestione API REST (PM2):

pm2 list
pm2 restart monitoring-api
pm2 logs monitoring-api

text

Questo README è il documento **definitivo e ufficiale** del project work: copre tutti i moduli (Setup, Docker, Sicurezza, Monitoraggio, Automazione) e i bonus implementati (Docker LLM, Grafana/Loki/Promtail, API REST di monitoring, backup automatico).
