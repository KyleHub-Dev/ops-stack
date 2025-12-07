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

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
