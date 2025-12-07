# <img src="logo.png" alt="KyleHub Logo" width="40" height="40" align="center"> KyleHub Ops Stack

Infrastructure as Code repository for the KyleHub network. This repository contains container orchestration configurations, setup scripts, and service definitions.

## Documentation

For comprehensive documentation, please visit our [Docusaurus docs](https://docs.kylehub.dev) (coming soon).

## Server Setup

Standard commands for a fresh installation (Debian 13 / Ubuntu 24.04):

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install essential tools
sudo apt install -y curl wget git nano htop fail2ban

# Configure Fail2ban (protect SSH)
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo systemctl enable fail2ban
sudo systemctl start fail2ban

# Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
```

## SSH & User Management

### Connecting to your VPS
If you added an SSH key during creation, connect using the `root` user:
```bash
ssh root@<your-ip>
```
*Note: If you are asked for a password, ensure you are using `root@` and not your local username.*

### Creating a Sudo User
It is recommended to create a non-root user for daily operations.

1. **Create user:**
   ```bash
   adduser kyle
   ```
2. **Add to sudo group:**
   ```bash
   usermod -aG sudo kyle
   ```
3. **Setup SSH for new user:**
   Run these commands as `root`:
   ```bash
   # Create .ssh directory
   mkdir /home/kyle/.ssh
   chmod 700 /home/kyle/.ssh

   # Create authorized_keys
   touch /home/kyle/.ssh/authorized_keys
   chmod 600 /home/kyle/.ssh/authorized_keys

   # Set ownership
   chown -R kyle:kyle /home/kyle/.ssh
   ```
4. **Add your Public Key:**
   On your local machine, get your public key:
   ```bash
   cat ~/.ssh/id_ed25519.pub
   ```
   On the server (as root), add it to the new user's config:
   ```bash
   echo "YOUR_PUBLIC_KEY_CONTENT" >> /home/kyle/.ssh/authorized_keys
   ```

## Structure

- `services/` - Service definitions and configurations

## Security & Hardening

To prevent accidental leakage of secrets (API keys, passwords, private keys), we use a multi-layered approach.

### 1. Gitignore
We have configured `.gitignore` to exclude:
- `.env` files (Environment variables)
- `*.pat` files (Zitadel Personal Access Tokens)
- `*.key`, `*.pem` (SSL/SSH keys)
- `data/`, `backups/` (Persistent data)

### 2. Pre-Commit Hooks (Recommended)
We use [pre-commit](https://pre-commit.com/) with **Gitleaks** to automatically scan your staged changes for secrets *before* you commit.

**Setup:**
1. Install pre-commit:
   ```bash
   pip install pre-commit
   # OR on macOS
   brew install pre-commit
   ```
2. Install the hooks:
   ```bash
   pre-commit install
   ```
Now, every time you run `git commit`, Gitleaks will scan your changes. If it finds a secret, the commit will be blocked.

### 3. File Permissions
Ensure your configuration files on the server are secure:
```bash
# Secure .env files
chmod 600 services/auth/.env

# Secure ACME (Let's Encrypt) storage
chmod 600 services/auth/letsencrypt/acme.json
```

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
