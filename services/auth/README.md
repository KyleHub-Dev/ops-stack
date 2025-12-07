# Zitadel Auth-Server Setup

This is the documentation for setting up the central authentication server.

**Stack:**

  * **Hardware:** Hetzner Cloud VPS (CX23 - 2 vCPU, 4GB RAM)
  * **Server:** auth-debian-4gb-nbg1-1
  * **OS:** Debian 13 "Trixie" (Stable)
  * **Runtime:** Docker Engine (Official)
  * **Proxy:** Traefik v3 (via Docker Socket)
  * **App:** Zitadel + PostgreSQL 17

-----

## Step 1: Server Initialization (Hetzner)

1.  Select the **Debian 13** image in the Hetzner Panel.
2.  Add your **SSH Key** (Never log in with a password).
3.  Create the server.

### 1.1 Server Setup

Execute the **Server Setup** steps from the main README (`/README.md`). These include:
*   System Updates
*   Installation of tools (curl, git, nano, htop, fail2ban)
*   Docker Installation

-----

## Step 2: Repository Setup

We clone the entire repository to `/opt/ops-stack`. Since the repo is small, this is the easiest way.

```bash
cd /opt
git clone https://github.com/KyleHub-Dev/ops-stack.git
cd ops-stack/services/auth
```

## Step 3: Configuration

### 3.1 Directory Structure

Create local directories for persistent data (these are not synchronized with git):

```bash
mkdir -p letsencrypt
mkdir -p data/postgres
```

### 3.2 Environment Variables

Copy the example configuration and adjust it:

```bash
cp .env.example .env
nano .env
```

### 3.3 Set Permissions

Traefik requires an empty file with restrictive permissions, otherwise the container will not start.

```bash
touch ./letsencrypt/acme.json
chmod 600 ./letsencrypt/acme.json
```

-----

## Step 4: Cloudflare & Firewall Configuration

To make the server "stealthy":

1.  **Cloudflare DNS:** Set an A-Record for your subdomain to the server IP. Proxy Status: **Orange (Proxied)**.
2.  **Cloudflare SSL:** Set SSL/TLS to **"Full (Strict)"**.
3.  **Hetzner Cloud Firewall:**
      * Create a Firewall in the Hetzner Panel.
      * **Inbound:**
          * TCP 22 (SSH): Only your own IP.
          * TCP 80 & 443 (Web): **Allow only Cloudflare IPs** (List: [https://www.cloudflare.com/ips-v4](https://www.cloudflare.com/ips-v4)).
      * Apply the Firewall to the server.

-----

## Step 5: Start

```bash
cd /opt/ops-stack/services/auth
docker compose up -d
```

Wait approx. 1-2 minutes for the first start (Database initialization). Check the logs with:

```bash
docker compose logs -f zitadel
```

Once Zitadel is running, you can reach it at: `https://auth.yourdomain.com/ui/console`

-----

## Step 6: Backups & Updates

### Backup Script

Create `/opt/ops-stack/services/auth/backup.sh` and run it daily via Cronjob (`crontab -e`).

```bash
#!/bin/bash
BACKUP_DIR="/opt/ops-stack/services/auth/backups"
mkdir -p $BACKUP_DIR
docker exec zitadel_db pg_dump -U zitadel -d zitadel > "$BACKUP_DIR/db_backup_$(date +%F).sql"
# Delete backups older than 7 days
find $BACKUP_DIR -type f -name "*.sql" -mtime +7 -delete
```

### Updates

To update to the latest versions of Debian 13 or Docker Images:

```bash
apt update && apt upgrade -y
cd /opt/ops-stack/services/auth
docker compose pull
docker compose up -d
docker image prune -f
```