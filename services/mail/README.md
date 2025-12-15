# Proton Mail Bridge & Local SMTP Relay Setup

This guide details how to set up a secure, headless email sending infrastructure using Proton Mail Bridge and Postfix.

## Important Usage Note: Production Use & Limits

You are using a Proton Mail Pro/Business plan, which offers significantly higher sending limits and custom domain support compared to free accounts. However, please note:

*   **Transactional vs. Bulk:** Proton Mail is optimized for secure business communication. While it can handle transactional emails (password resets, notifications) for standard business operations, it is not a dedicated bulk emailing service.
*   **Best Practice:** Ensure your sending volume remains consistent and avoids "spam-like" bursts. If your application scales to massive volumes (e.g., tens of thousands of emails daily), dedicated Transactional Email Service Providers (Postmark, SendGrid) are recommended to ensure deliverability and reputation management.
*   **Custom Domains:** This setup fully supports your custom domain (e.g., `noreply@yourdomain.com`). Ensure this address is configured as an alias in your Proton account.

---

## Architecture

To ensure reliability and compatibility, we use a 3-Tier Architecture:

1.  **Application (e.g., Zitadel):** Connects to `localhost:25` (standard, unencrypted). Simple and fast.
2.  **Middleware (Postfix):**
    *   Listens on `localhost:25`.
    *   Queues mail (prevents data loss if Bridge is busy).
    *   Translates simple traffic into the complex encrypted authentication required by the Bridge.
3.  **Proton Mail Bridge:**
    *   Listens on `localhost:1025`.
    *   Encrypts mail and sends it to Proton servers.

```mermaid
graph LR
    A[Zitadel / Apps] -- SMTP (No Auth) --> B(Postfix Relay :25)
    B -- SMTP (Auth + TLS) --> C(Proton Mail Bridge :1025)
    C -- Encrypted Tunnel --> D[Proton Mail Cloud]
```

---

## Phase 1: Headless Proton Mail Bridge Setup

Since we are on a headless Linux server, we cannot use the GUI. We must use `gpg` and `pass` to handle credentials securely.

### 1. Install Dependencies
```bash
# Ubuntu/Debian
sudo apt update && sudo apt install -y pass gnupg
```

### 2. Download & Install Bridge
Download the latest `.deb` from the [Proton Bridge Release Page](https://proton.me/mail/bridge).
```bash
wget https://proton.me/download/bridge/protonmail-bridge_3.12.0-1_amd64.deb -O proton-bridge.deb
sudo dpkg -i proton-bridge.deb
rm proton-bridge.deb
```

### 3. Create Isolated Service User
We run the bridge as a restricted user (`proton`) for security.
```bash
# Create user with no login shell
sudo useradd -r -s /bin/false proton

# Create home and log directories
sudo mkdir -p /home/proton /var/log/proton-bridge
sudo chown -R proton:proton /home/proton /var/log/proton-bridge
```

### 4. Setup Key Store (GPG & Pass)
Execute these commands specifically as the `proton` user to initialize the headless keychain.

```bash
# Switch to proton user context temporarily
sudo -u proton bash

# 1. Generate GPG key (Leave passphrase EMPTY for headless auto-start)
gpg --batch --passphrase '' --quick-gen-key 'ProtonMail Bridge' default default never

# 2. Initialize pass database
pass init "ProtonMail Bridge"

# 3. Exit back to root/sudo user
exit
```

### 5. Login & Configure Bridge
Run the bridge CLI interactively **once** to login.

```bash
sudo -u proton protonmail-bridge --cli
```

**Inside the CLI:**
1.  Type `login`.
2.  Enter your Proton Mail username (e.g., `admin@yourdomain.com`) and password.
3.  Type `info` to see your configuration.
4.  **IMPORTANT:** Note down the **SMTP Username** (usually your email) and **SMTP Password** (a generated string, *not* your login password). You will need these for Postfix.
5.  Type `exit`.

### 6. Daemonize (Systemd Service)
Create a service to keep the bridge running.

`sudo nano /etc/systemd/system/proton-bridge.service`

```ini
[Unit]
Description=ProtonMail Bridge
After=network.target

[Service]
Type=simple
User=proton
ExecStart=/usr/bin/protonmail-bridge --noninteractive
StandardOutput=append:/var/log/proton-bridge/bridge.log
StandardError=append:/var/log/proton-bridge/error.log
Restart=always

[Install]
WantedBy=multi-user.target
```

**Start the service:**
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now proton-bridge
```

---

## Phase 2: Middleware (Postfix Relay)

We use Postfix to "bridge the bridge". It accepts mail easily and handles the hard work of talking to Proton.

### 1. Install Postfix
During installation, select **"Satellite system"**.
*   **System mail name:** (e.g., `hostname.yourdomain.com`)
*   **SMTP relay host:** `[127.0.0.1]:1025`

```bash
sudo apt install -y postfix libsasl2-modules
```

### 2. Configure Authentication
Create the password map file. Replace with the credentials you got from the Bridge CLI (Step 5 above).

`sudo nano /etc/postfix/sasl_passwd`

```text
[127.0.0.1]:1025    admin@yourdomain.com:YOUR_GENERATED_BRIDGE_PASSWORD
```

**Secure and Hash the file:**
```bash
sudo chmod 600 /etc/postfix/sasl_passwd
sudo postmap /etc/postfix/sasl_passwd
```

### 3. Configure `main.cf`
Edit `/etc/postfix/main.cf` to ensure it relays correctly. Ensure these lines exist and are modified:

```bash
# RELAY CONFIGURATION
# The brackets [] prevent DNS lookups (crucial for localhost)
relayhost = [127.0.0.1]:1025

# AUTHENTICATION
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous

# TLS SETTINGS (Must be 'may' to accept Bridge's self-signed cert)
smtp_tls_security_level = may

# NETWORK SECURITY
# Only allow this server (localhost) to send mail
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128
inet_interfaces = loopback-only
```

### 4. Restart Postfix
```bash
sudo systemctl restart postfix
```

---

## Phase 3: Integration (Zitadel & Others)

Now that Postfix is listening on port 25, integrating applications is easy. They don't need to know about Proton Mail, encryption, or complex auth. They just see a standard open SMTP server.

### General Configuration for ANY App
*   **SMTP Host:** `127.0.0.1` (or the Docker host IP if running in container)
*   **SMTP Port:** `25`
*   **TLS/SSL:** `None` / `False` (Security is handled by local loopback trust)
*   **Username:** (Leave Empty)
*   **Password:** (Leave Empty)
*   **Sender Address:** **MUST** match an active address or alias in your Proton account (e.g., `noreply@yourdomain.com`).

### Specific: Zitadel Configuration
If configuring via the Zitadel Console or `defaults.yaml`:

```yaml
SMTPConfiguration:
  Host: "172.17.0.1" # Use 'host.docker.internal' or the Docker Bridge Gateway IP if Zitadel is in Docker
  Port: 25
  SenderName: "Zitadel Identity"
  SenderAddress: "noreply@yourdomain.com" # Must match authenticated user/alias
  TLS: false
```

### How to Extend to Other Apps
Follow the "General Configuration" above.
*   **GitLab:** Set `gitlab_rails['smtp_address'] = "127.0.0.1"` and `smtp_port = 25`.
*   **Grafana:** Set `[smtp] host = 127.0.0.1:25`, `skip_verify = true`.
*   **Cron/System Alerts:** They typically default to `localhost:25`, so they will start working automatically.

---

## Testing

1.  **Test Postfix Relay:**
    ```bash
    echo "This is a test email." | mail -s "Test Subject" your-personal-email@gmail.com
    ```

2.  **Check Logs:**
    *   **Postfix:** `tail -f /var/log/mail.log`
    *   **Bridge:** `tail -f /var/log/proton-bridge/bridge.log`

3.  **Troubleshooting:**
    *   **Error:** *Client host rejected: Access denied* -> Check `mynetworks` in `main.cf`.
    *   **Error:** *Certificate verification failed* -> Ensure `smtp_tls_security_level = may` in `main.cf`.
