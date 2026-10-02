# Rocky Linux 9.4 Server Setup & Security Documentation

This internal documentation covers the system initialization, containerization setup using **Podman**, security hardening via **SELinux**, reverse proxy configuration using **Caddy**, and brute-force prevention with **Fail2ban**.

---

## 1. Package Management & Podman Installation

Podman is utilized as a daemonless, rootless container engine replacement for Docker.

```bash
# Update system repositories
sudo dnf update -y

# Install Podman
sudo dnf install podman -y

# Verify installation
podman --version
```

### Rootless User Configuration
Ensure your user account can allocate subordinate user/group IDs for unprivileged container isolation:
```bash
# Check existing configuration
cat /etc/subuid
cat /etc/subgid

# If your user is not listed, add subuids and subgids manually
sudo usermod --add-subuids 100000-165535 --add-subgids 100000-165535 $(whoami)
```

---

## 2. Podman Compose Setup

To orchestrate multi-container setups using regular compose configuration files, install `podman-compose` from the EPEL repository.

```bash
# Enable Extra Packages for Enterprise Linux (EPEL) repo
sudo dnf install -y epel-release

# Install Podman Compose
sudo dnf install -y podman-compose

# Verify installation
podman-compose --version
```

---

## 3. Install Karakeep Application

To run `karakeep` application using podman follow this instructions:

1. Create a new directory to host the compose file and env variables.
   ```bash
   mkdir -p /app/karakeep-app
   cd /app/mkdir karakeep-app
   ```
2. Download the docker compose file provided here directly into your new directory.
   ```bash
   wget https://raw.githubusercontent.com/karakeep-app/karakeep/main/docker/docker-compose.yml
   ```
3. Populate the environment variables
   > You should change the random strings
   > 
   >> You can use `openssl rand -base64 36` in a separate terminal window to generate the random strings.
   > Then put the result in `.env` below
   ```ini
   KARAKEEP_VERSION=0.33.2
   NEXTAUTH_SECRET=super_random_string
   MEILI_MASTER_KEY=another_random_string
   NEXTAUTH_URL=http://localhost:8080
   ```
4. Update `docker-compose.yml` and change default port to `8080:8080`
5. Spin up your multi-container environment, navigate to your compose configuration directory and run:
   ```bash
   podman compose up -d
   ```
   > [!NOTE]
   > Podman requires fully qualified image names like `image: docker.io/library/ubuntu` inside compose manifests to prevent interactive prompt hangs).*6. 
   
---

## 3. SELinux & Custom Storage Security

Rocky Linux 9.4 runs with SELinux enabled. To avoid application lockouts or permission issues, execute the following steps to safely transitionalize or target custom configurations.

### Safe SELinux Permissive Transition (If currently Disabled)
1. Open the configuration file:
   ```bash
   sudo nano /etc/selinux/config
   ```
2. Modify or verify the target state:
   ```text
   SELINUX=enforcing
   ```
3. Trigger a background filesystem relabel and reboot:
   ```bash
   sudo touch /.autorelabel
   sudo reboot
   ```

### Custom Working Directory Configuration (`/app`)
Configure the system to permanently authorize container reads and writes to the custom `/app` directory:
```bash
# Create directory and assign local user ownership
sudo chown -R $(whoami):$(whoami) /app

# Add permanent SELinux file context label for containers
sudo semanage fcontext -a -t container_file_t "/app(/.*)?"

# Apply the security label immediately
sudo restorecon -R -v /app

# Verify the assignment
ls -dZ /app
```

> [!NOTE] 
> For local directory bindings inside compose volumes, append the `:Z` flag to control proper container isolation: `- ./data:/app/data:Z`.

---

## 4. Network Security & Custom Port Definitions

To route production services and prevent lockouts, update your active rules in `firewalld` and inform SELinux policy constraints regarding your non-standard ports.

### Custom SSH Port Authorization
```bash
# Install SELinux management utilities
sudo dnf install -y policycoreutils-python-utils

# Bind custom port to the ssh_port_t policy (Replace CUSTOM_PORT)
sudo semanage port -a -t ssh_port_t -p tcp CUSTOM_PORT

# Verify the port map
sudo semanage port -l | grep ssh
```

### App Port Activation (Port 8080)
```bash
# Explicitly authorize standard web operations or container targets on port 8080
sudo semanage port -a -t http_port_t -p tcp 8080
```

### Firewalld Adjustments
Ensure your external ingress network paths are clear for application targets:
```bash
# Open application port 8080 and standard web channels
sudo firewall-cmd --add-port=8080/tcp --permanent
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --add-service=https --permanent

# Reload configurations to apply live
sudo firewall-cmd --reload
```

---

## 5. Reverse Proxy with Caddy (SSL Automation)

Caddy handles transparent SSL generation and terminates traffic externally on ports 80/443 before forwarding to the loopback target (`localhost:8080`).

### Installation
```bash
# Enable the official Caddy Copr repository
sudo dnf install -y dnf-plugins-core
sudo dnf copr enable @caddy/caddy -y

# Install Caddy binary
sudo dnf install -y caddy
```

### SELinux Service Interconnect Permission
Allow the Caddy reverse proxy service worker to establish upstream connections over internal network layers:
```bash
sudo setsebool -P httpd_can_network_connect 1
```

### Deployment Configuration (`/etc/caddy/Caddyfile`)
```text
app.yourdomain.com {
    reverse_proxy localhost:8080
}
```

### Process Management
```bash
# Enable and start the daemon
sudo systemctl enable --now caddy

# Monitor runtime and ACME certificate verification logs
sudo systemctl status caddy
```

---

## 6. Brute-Force Hardening via Fail2ban

Fail2ban analyzes auth failures and dynamically creates `firewalld` blocks.

### Installation
```bash
sudo dnf install -y fail2ban fail2ban-firewalld
```

### Operational Rule Profiles (`/etc/fail2ban/jail.local`)
Create an override configuration file to secure your custom SSH infrastructure:
```ini
[DEFAULT]
bantime  = 3600
findtime = 600
maxretry = 5
banaction = firewallcmd-ipset
backend = systemd

[sshd]
enabled = true
port    = CUSTOM_PORT
logpath = %(sshd_log)s
backend = systemd
```

### Service Engine Administration
```bash
# Enable and initialize the tracking daemon
sudo systemctl enable --now fail2ban

# Query live monitoring state and jail statistics
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

---
*End of Documentation.*

