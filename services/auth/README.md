# Zitadel Auth-Server Setup

Dies ist die Dokumentation für den Aufbau des zentralen Authentifizierungs-Servers.

**Stack:**

  * **Hardware:** Hetzner Cloud VPS (CX23 - 2 vCPU, 4GB RAM)
  * **Server:** auth-debian-4gb-nbg1-1
  * **OS:** Debian 13 "Trixie" (Stable)
  * **Runtime:** Docker Engine (Official)
  * **Proxy:** Traefik v3 (via Docker Socket)
  * **App:** Zitadel + PostgreSQL 17

-----

## Schritt 1: Server Initialisierung (Hetzner)

1.  Wähle im Hetzner Panel das Image **Debian 13** aus.
2.  Füge deinen **SSH-Key** hinzu (Logge dich niemals mit Passwort ein).
3.  Erstelle den Server.

### 1.1 Server Setup

Führe die **Server Setup** Schritte aus der Haupt-README (`/README.md`) aus. Diese beinhalten:
*   System Updates
*   Installation von Tools (curl, git, nano, htop, fail2ban)
*   Docker Installation

-----

## Schritt 2: Repository Setup

Wir klonen das gesamte Repository nach `/opt/ops-stack`. Da das Repo klein ist, ist dies der einfachste Weg.

```bash
cd /opt
git clone https://github.com/KyleHub-Dev/ops-stack.git
cd ops-stack/services/auth
```

## Schritt 3: Konfiguration

### 3.1 Ordnerstruktur

Erstelle die lokalen Ordner für persistente Daten (diese werden nicht mit git synchronisiert):

```bash
mkdir -p letsencrypt
mkdir -p data/postgres
```

### 3.2 Environment Variablen

Kopiere die Beispiel-Konfiguration und passe sie an:

```bash
cp .env.example .env
nano .env
```

### 3.3 Permissions setzen

Traefik benötigt eine leere Datei mit restriktiven Rechten, sonst startet der Container nicht.

```bash
touch ./letsencrypt/acme.json
chmod 600 ./letsencrypt/acme.json
```

-----

## Schritt 4: Cloudflare & Firewall Konfiguration

Um den Server "stealthy" zu machen:

1.  **Cloudflare DNS:** Setze einen A-Record für deine Subdomain auf die Server-IP. Proxy Status: **Orange (Proxied)**.
2.  **Cloudflare SSL:** Setze SSL/TLS auf **"Full (Strict)"**.
3.  **Hetzner Cloud Firewall:**
      * Erstelle eine Firewall im Hetzner Panel.
      * **Inbound:**
          * TCP 22 (SSH): Nur deine eigene IP.
          * TCP 80 & 443 (Web): **Nur Cloudflare IPs erlauben** (Liste: [https://www.cloudflare.com/ips-v4](https://www.cloudflare.com/ips-v4)).
      * Wende die Firewall auf den Server an.

-----

## Schritt 5: Start

```bash
cd /opt/ops-stack/services/auth
docker compose up -d
```

Warte ca. 1-2 Minuten beim ersten Start (Datenbank-Initialisierung). Prüfe die Logs mit:

```bash
docker compose logs -f zitadel
```

Sobald Zitadel läuft, erreichst du es unter: `https://auth.deinedomain.de/ui/console`

-----

## Schritt 6: Backups & Updates

### Backup Script

Erstelle `/opt/ops-stack/services/auth/backup.sh` und lasse es via Cronjob (`crontab -e`) täglich laufen.

```bash
#!/bin/bash
BACKUP_DIR="/opt/ops-stack/services/auth/backups"
mkdir -p $BACKUP_DIR
docker exec zitadel_db pg_dump -U zitadel -d zitadel > "$BACKUP_DIR/db_backup_$(date +%F).sql"
# Lösche Backups älter als 7 Tage
find $BACKUP_DIR -type f -name "*.sql" -mtime +7 -delete
```

### Updates

Um auf die neuesten Versionen von Debian 13 oder Docker Images zu aktualisieren:

```bash
apt update && apt upgrade -y
cd /opt/ops-stack/services/auth
docker compose pull
docker compose up -d
docker image prune -f
```