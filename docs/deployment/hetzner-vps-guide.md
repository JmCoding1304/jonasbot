# Hetzner VPS Deployment Guide for OpenClaw

Complete guide to deploy OpenClaw on Hetzner Cloud with security hardening and Cloudflare protection.

## Table of Contents

1. [Hetzner Account Setup](#1-hetzner-account-setup)
2. [VPS Creation](#2-vps-creation)
3. [Initial Server Security](#3-initial-server-security)
4. [Cloudflare Setup (Free Tier)](#4-cloudflare-setup-free-tier)
5. [Install OpenClaw](#5-install-openclaw)
6. [Configure OpenClaw](#6-configure-openclaw)
7. [Reverse Proxy Setup](#7-reverse-proxy-setup-nginx)
8. [SSL/TLS with Cloudflare](#8-ssltls-with-cloudflare)
9. [Systemd Service](#9-systemd-service)
10. [Firewall Configuration](#10-firewall-configuration)
11. [Monitoring & Maintenance](#11-monitoring--maintenance)
12. [Security Checklist](#12-security-checklist)

---

## 1. Hetzner Account Setup

### Create Account

1. Go to [https://accounts.hetzner.com/signUp](https://accounts.hetzner.com/signUp)
2. Enter email and create password
3. Verify email address
4. Complete identity verification (may require ID upload)
5. Add payment method (credit card or PayPal)

### Access Cloud Console

1. Go to [https://console.hetzner.cloud/](https://console.hetzner.cloud/)
2. Create a new project (e.g., "openclaw-prod")
3. Note your project ID for future reference

---

## 2. VPS Creation

### Choose Server Specs

For OpenClaw, recommended minimum specs:

| Use Case            | Server Type | vCPU | RAM  | Storage | Monthly Cost |
| ------------------- | ----------- | ---- | ---- | ------- | ------------ |
| Light (personal)    | CX22        | 2    | 4GB  | 40GB    | ~€4.50       |
| Medium (small team) | CX32        | 4    | 8GB  | 80GB    | ~€9.00       |
| Heavy (production)  | CX42        | 8    | 16GB | 160GB   | ~€18.00      |

### Create Server

1. Click **"Add Server"** in Hetzner Cloud Console

2. **Location**: Choose closest to your users
   - Falkenstein (Germany) - EU
   - Nuremberg (Germany) - EU
   - Helsinki (Finland) - EU
   - Ashburn (US East)
   - Hillsboro (US West)

3. **Image**: Ubuntu 24.04 LTS

4. **Type**: Select based on needs (CX22 for personal use)

5. **Networking**:
   - ✅ Public IPv4
   - ✅ Public IPv6
   - ❌ Private networks (optional, for multi-server setups)

6. **SSH Keys**:

   ```bash
   # On your local machine, generate if you don't have one:
   ssh-keygen -t ed25519 -C "openclaw-vps"

   # Copy public key:
   cat ~/.ssh/id_ed25519.pub
   ```

   - Click "Add SSH Key" and paste your public key
   - **CRITICAL**: Do NOT enable password authentication

7. **Volumes**: Skip (not needed initially)

8. **Firewalls**: Create new firewall (configure later)

9. **Backups**: ✅ Enable (~20% extra cost, worth it)

10. **Name**: `openclaw-prod-1`

11. Click **"Create & Buy now"**

### Note Your Server IP

After creation, note:

- IPv4: `xxx.xxx.xxx.xxx`
- IPv6: `xxxx:xxxx:xxxx:xxxx::1`

---

## 3. Initial Server Security

### First Login

```bash
# SSH into your server
ssh root@YOUR_SERVER_IP
```

### Create Non-Root User

```bash
# Create user for openclaw
adduser openclaw
usermod -aG sudo openclaw

# Copy SSH key to new user
mkdir -p /home/openclaw/.ssh
cp ~/.ssh/authorized_keys /home/openclaw/.ssh/
chown -R openclaw:openclaw /home/openclaw/.ssh
chmod 700 /home/openclaw/.ssh
chmod 600 /home/openclaw/.ssh/authorized_keys

# Test login in new terminal before proceeding
# ssh openclaw@YOUR_SERVER_IP
```

### Harden SSH

```bash
# Edit SSH config
nano /etc/ssh/sshd_config

# Change/add these lines:
Port 2222                          # Non-standard port
PermitRootLogin no                 # Disable root login
PasswordAuthentication no          # Key-only auth
PubkeyAuthentication yes
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
X11Forwarding no
AllowUsers openclaw                # Only allow openclaw user

# Restart SSH
systemctl restart sshd
```

**IMPORTANT**: Test new SSH config in a NEW terminal before closing current session:

```bash
ssh -p 2222 openclaw@YOUR_SERVER_IP
```

### System Updates

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install essential security tools
sudo apt install -y \
  fail2ban \
  ufw \
  unattended-upgrades \
  apt-listchanges \
  logwatch \
  rkhunter

# Enable automatic security updates
sudo dpkg-reconfigure -plow unattended-upgrades
```

### Configure Fail2ban

```bash
# Create local config
sudo nano /etc/fail2ban/jail.local
```

Add:

```ini
[DEFAULT]
bantime = 1h
findtime = 10m
maxretry = 3
ignoreip = 127.0.0.1/8 ::1

[sshd]
enabled = true
port = 2222
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 24h
```

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

---

## 4. Cloudflare Setup (Free Tier)

Cloudflare provides free:

- DDoS protection
- SSL/TLS certificates
- CDN caching
- IP hiding (your real server IP is hidden)
- Rate limiting (limited on free tier)

### Create Cloudflare Account

1. Go to [https://dash.cloudflare.com/sign-up](https://dash.cloudflare.com/sign-up)
2. Create account with email
3. Verify email

### Add Your Domain

1. Click **"Add a Site"**
2. Enter your domain (e.g., `yourdomain.com`)
3. Select **Free** plan
4. Cloudflare will scan existing DNS records

### Update Nameservers

1. Cloudflare provides two nameservers (e.g., `ada.ns.cloudflare.com`)
2. Go to your domain registrar
3. Replace nameservers with Cloudflare's
4. Wait 24-48 hours for propagation (usually faster)

### Configure DNS Records

Add these DNS records in Cloudflare:

| Type | Name | Content          | Proxy Status              |
| ---- | ---- | ---------------- | ------------------------- |
| A    | @    | YOUR_SERVER_IP   | ✅ Proxied (orange cloud) |
| A    | api  | YOUR_SERVER_IP   | ✅ Proxied                |
| AAAA | @    | YOUR_SERVER_IPv6 | ✅ Proxied                |
| AAAA | api  | YOUR_SERVER_IPv6 | ✅ Proxied                |

**CRITICAL**: Keep proxy enabled (orange cloud) to hide your real IP.

### SSL/TLS Settings

1. Go to **SSL/TLS** → **Overview**
2. Set encryption mode to **Full (strict)**
3. Go to **Edge Certificates**:
   - ✅ Always Use HTTPS
   - ✅ Automatic HTTPS Rewrites
   - Minimum TLS Version: TLS 1.2

### Security Settings

1. Go to **Security** → **Settings**:
   - Security Level: Medium
   - Challenge Passage: 30 minutes
   - Browser Integrity Check: ✅ On

2. Go to **Security** → **Bots**:
   - Bot Fight Mode: ✅ On (free tier)

### Firewall Rules (Free: 5 rules)

Go to **Security** → **WAF** → **Custom rules**:

**Rule 1: Block Bad Bots**

```
(cf.client.bot) and not (cf.bot_management.verified_bot)
Action: Block
```

**Rule 2: Block High-Risk Countries** (optional)

```
(ip.geoip.country in {"CN" "RU" "KP"})
Action: Challenge
```

**Rule 3: Protect API Endpoint**

```
(http.request.uri.path contains "/v1/") and (not http.request.method in {"POST" "GET"})
Action: Block
```

### Page Rules (Free: 3 rules)

1. **Cache static assets**:
   - URL: `*yourdomain.com/static/*`
   - Cache Level: Cache Everything

2. **Bypass cache for API**:
   - URL: `*yourdomain.com/v1/*`
   - Cache Level: Bypass

---

## 5. Install OpenClaw

### Install Node.js

```bash
# Install Node.js 22 via NodeSource
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs

# Verify
node --version  # Should be v22.x
npm --version
```

### Install OpenClaw

```bash
# Install globally
sudo npm install -g openclaw

# Verify installation
openclaw --version
```

### Create Directories

```bash
# Create OpenClaw directories with proper permissions
mkdir -p ~/.openclaw/{workspace,credentials,agents}
chmod 700 ~/.openclaw
chmod 700 ~/.openclaw/credentials
```

---

## 6. Configure OpenClaw

### Generate Secure Tokens

```bash
# Generate gateway token
export OPENCLAW_GATEWAY_TOKEN=$(openssl rand -base64 32)
echo "Gateway Token: $OPENCLAW_GATEWAY_TOKEN"
# SAVE THIS TOKEN SECURELY

# Generate webhook secret (if using webhooks)
export WEBHOOK_SECRET=$(openssl rand -base64 32)
echo "Webhook Secret: $WEBHOOK_SECRET"
```

### Create Environment File

```bash
# Create secure env file
sudo mkdir -p /etc/openclaw
sudo nano /etc/openclaw/env
```

Add:

```bash
# OpenClaw Environment Variables
# SECURITY: This file should have 600 permissions

# Authentication (REQUIRED - choose one)
ANTHROPIC_API_KEY=sk-ant-your-key-here

# Gateway Security (REQUIRED)
OPENCLAW_GATEWAY_TOKEN=your-generated-token-here

# Optional: Webhook secret
OPENCLAW_WEBHOOK_SECRET=your-webhook-secret

# Optional: Override state directory
# OPENCLAW_STATE_DIR=/var/lib/openclaw
```

```bash
# Secure the env file
sudo chmod 600 /etc/openclaw/env
sudo chown openclaw:openclaw /etc/openclaw/env
```

### Create Configuration File

```bash
nano ~/.openclaw/openclaw.json
```

Copy the cost-optimized config with security hardening:

```json5
// OpenClaw Production Configuration for VPS
// Security-hardened for Hetzner + Cloudflare deployment
{
  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace",

      // Cost-optimized model routing
      model: {
        primary: "anthropic/claude-haiku-4-5",
        fallbacks: ["anthropic/claude-sonnet-4-5", "anthropic/claude-opus-4-5"],
      },

      // Model aliases + prompt caching
      models: {
        "anthropic/claude-opus-4-5": {
          alias: "opus",
          params: { cacheRetention: "short" },
        },
        "anthropic/claude-sonnet-4-5": {
          alias: "sonnet",
          params: { cacheRetention: "short" },
        },
        "anthropic/claude-haiku-4-5": { alias: "haiku" },
      },

      // Heartbeat keeps cache warm
      heartbeat: {
        every: "4m",
        model: "anthropic/claude-haiku-4-5",
        target: "last",
      },

      // Context pruning synced with cache
      contextPruning: {
        mode: "cache-ttl",
        ttl: "5m",
        keepLastAssistants: 3,
      },

      timeoutSeconds: 300,
      maxConcurrent: 3,
    },
  },

  // Gateway security settings
  gateway: {
    mode: "local",
    port: 18789,
    bind: "loopback", // CRITICAL: Only bind to localhost, nginx handles external

    auth: {
      mode: "token",
      // Token loaded from environment variable
    },

    // Control UI settings
    controlUi: {
      enabled: true,
      basePath: "/openclaw",
      allowedOrigins: ["https://yourdomain.com", "https://api.yourdomain.com"],
      dangerouslyDisableDeviceAuth: false,
    },

    // Trust Cloudflare proxy IPs
    trustedProxies: [
      "127.0.0.1",
      // Cloudflare IPv4 ranges (update periodically)
      "173.245.48.0/20",
      "103.21.244.0/22",
      "103.22.200.0/22",
      "103.31.4.0/22",
      "141.101.64.0/18",
      "108.162.192.0/18",
      "190.93.240.0/20",
      "188.114.96.0/20",
      "197.234.240.0/22",
      "198.41.128.0/17",
      "162.158.0.0/15",
      "104.16.0.0/13",
      "104.24.0.0/14",
      "172.64.0.0/13",
      "131.0.72.0/22",
    ],
  },

  // Logging with redaction
  logging: {
    level: "info",
    file: "/var/log/openclaw/openclaw.log",
    consoleLevel: "warn",
    redactSensitive: "tools",
  },

  // Session management
  session: {
    scope: "per-sender",
    reset: {
      mode: "daily",
      atHour: 4,
      idleMinutes: 60,
    },
  },

  // Channels (customize as needed)
  channels: {
    whatsapp: {
      dmPolicy: "pairing",
      allowFrom: ["+1234567890"], // Replace with your number
    },
  },

  // Webhook security
  hooks: {
    enabled: true,
    path: "/hooks",
    // Token loaded from OPENCLAW_WEBHOOK_SECRET env var
  },
}
```

```bash
# Secure config file
chmod 600 ~/.openclaw/openclaw.json
```

---

## 7. Reverse Proxy Setup (nginx)

### Install nginx

```bash
sudo apt install -y nginx
```

### Create nginx Configuration

```bash
sudo nano /etc/nginx/sites-available/openclaw
```

Add:

```nginx
# OpenClaw reverse proxy configuration
# Optimized for Cloudflare + security

# Rate limiting zones
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
limit_req_zone $binary_remote_addr zone=ws_limit:10m rate=5r/s;
limit_conn_zone $binary_remote_addr zone=conn_limit:10m;

# Upstream for OpenClaw gateway
upstream openclaw_gateway {
    server 127.0.0.1:18789;
    keepalive 32;
}

server {
    listen 80;
    listen [::]:80;
    server_name yourdomain.com api.yourdomain.com;

    # Redirect HTTP to HTTPS (Cloudflare handles SSL)
    # But we still want internal HTTPS for Full (strict) mode
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name yourdomain.com api.yourdomain.com;

    # Cloudflare Origin Certificate (see section 8)
    ssl_certificate /etc/ssl/cloudflare/cert.pem;
    ssl_certificate_key /etc/ssl/cloudflare/key.pem;

    # SSL settings
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Only allow Cloudflare IPs (updated list)
    # Get latest from: https://www.cloudflare.com/ips/
    set_real_ip_from 173.245.48.0/20;
    set_real_ip_from 103.21.244.0/22;
    set_real_ip_from 103.22.200.0/22;
    set_real_ip_from 103.31.4.0/22;
    set_real_ip_from 141.101.64.0/18;
    set_real_ip_from 108.162.192.0/18;
    set_real_ip_from 190.93.240.0/20;
    set_real_ip_from 188.114.96.0/20;
    set_real_ip_from 197.234.240.0/22;
    set_real_ip_from 198.41.128.0/17;
    set_real_ip_from 162.158.0.0/15;
    set_real_ip_from 104.16.0.0/13;
    set_real_ip_from 104.24.0.0/14;
    set_real_ip_from 172.64.0.0/13;
    set_real_ip_from 131.0.72.0/22;
    # Cloudflare IPv6
    set_real_ip_from 2400:cb00::/32;
    set_real_ip_from 2606:4700::/32;
    set_real_ip_from 2803:f800::/32;
    set_real_ip_from 2405:b500::/32;
    set_real_ip_from 2405:8100::/32;
    set_real_ip_from 2a06:98c0::/29;
    set_real_ip_from 2c0f:f248::/32;
    real_ip_header CF-Connecting-IP;

    # Connection limits
    limit_conn conn_limit 20;

    # API endpoint with rate limiting
    location /v1/ {
        limit_req zone=api_limit burst=20 nodelay;

        proxy_pass http://openclaw_gateway;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts for long-running requests
        proxy_connect_timeout 60s;
        proxy_send_timeout 300s;
        proxy_read_timeout 300s;

        # Body size limit
        client_max_body_size 20M;
    }

    # WebSocket endpoint
    location /ws {
        limit_req zone=ws_limit burst=10 nodelay;

        proxy_pass http://openclaw_gateway;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 3600s;
        proxy_read_timeout 3600s;
    }

    # Control UI
    location /openclaw/ {
        limit_req zone=api_limit burst=10 nodelay;

        proxy_pass http://openclaw_gateway;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Webhooks endpoint
    location /hooks/ {
        limit_req zone=api_limit burst=50 nodelay;

        proxy_pass http://openclaw_gateway;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        client_max_body_size 5M;
    }

    # Health check (no rate limit)
    location /health {
        proxy_pass http://openclaw_gateway;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
    }

    # Block everything else
    location / {
        return 404;
    }

    # Deny access to sensitive files
    location ~ /\. {
        deny all;
    }
}
```

### Enable Configuration

```bash
# Enable site
sudo ln -s /etc/nginx/sites-available/openclaw /etc/nginx/sites-enabled/

# Remove default site
sudo rm /etc/nginx/sites-enabled/default

# Test configuration
sudo nginx -t

# Reload nginx
sudo systemctl reload nginx
```

---

## 8. SSL/TLS with Cloudflare

### Create Cloudflare Origin Certificate

1. Go to Cloudflare Dashboard → **SSL/TLS** → **Origin Server**
2. Click **"Create Certificate"**
3. Settings:
   - Private key type: RSA (2048)
   - Hostnames: `yourdomain.com`, `*.yourdomain.com`
   - Certificate Validity: 15 years (maximum)
4. Click **"Create"**
5. **IMPORTANT**: Copy and save both the certificate and private key

### Install Certificate on Server

```bash
# Create directory
sudo mkdir -p /etc/ssl/cloudflare

# Create certificate file
sudo nano /etc/ssl/cloudflare/cert.pem
# Paste the Origin Certificate

# Create key file
sudo nano /etc/ssl/cloudflare/key.pem
# Paste the Private Key

# Secure permissions
sudo chmod 600 /etc/ssl/cloudflare/key.pem
sudo chmod 644 /etc/ssl/cloudflare/cert.pem
```

### Test nginx with SSL

```bash
sudo nginx -t
sudo systemctl restart nginx
```

---

## 9. Systemd Service

### Create Service File

```bash
sudo nano /etc/systemd/system/openclaw.service
```

Add:

```ini
[Unit]
Description=OpenClaw Gateway
Documentation=https://docs.openclaw.ai
After=network.target

[Service]
Type=simple
User=openclaw
Group=openclaw
WorkingDirectory=/home/openclaw

# Load environment
EnvironmentFile=/etc/openclaw/env

# Start command
ExecStart=/usr/bin/node /usr/lib/node_modules/openclaw/dist/index.js gateway run
ExecReload=/bin/kill -HUP $MAINPID

# Restart policy
Restart=always
RestartSec=10
StartLimitIntervalSec=60
StartLimitBurst=3

# Security hardening
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=read-only
ReadWritePaths=/home/openclaw/.openclaw
ReadWritePaths=/var/log/openclaw

# Resource limits
MemoryMax=1G
CPUQuota=80%

# Logging
StandardOutput=append:/var/log/openclaw/openclaw.log
StandardError=append:/var/log/openclaw/error.log
SyslogIdentifier=openclaw

[Install]
WantedBy=multi-user.target
```

### Create Log Directory

```bash
sudo mkdir -p /var/log/openclaw
sudo chown openclaw:openclaw /var/log/openclaw
sudo chmod 755 /var/log/openclaw
```

### Enable and Start Service

```bash
# Reload systemd
sudo systemctl daemon-reload

# Enable on boot
sudo systemctl enable openclaw

# Start service
sudo systemctl start openclaw

# Check status
sudo systemctl status openclaw

# View logs
sudo journalctl -u openclaw -f
```

---

## 10. Firewall Configuration

### Configure UFW

```bash
# Reset UFW
sudo ufw --force reset

# Default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH (custom port)
sudo ufw allow 2222/tcp comment 'SSH'

# Allow HTTP/HTTPS (for nginx)
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'

# Enable UFW
sudo ufw enable

# Check status
sudo ufw status verbose
```

### Hetzner Cloud Firewall

Also configure firewall in Hetzner Cloud Console:

1. Go to **Firewalls** → **Create Firewall**
2. Add inbound rules:

| Protocol | Port | Source     | Description     |
| -------- | ---- | ---------- | --------------- |
| TCP      | 2222 | Your IP/32 | SSH access      |
| TCP      | 80   | 0.0.0.0/0  | HTTP (redirect) |
| TCP      | 443  | 0.0.0.0/0  | HTTPS           |

3. Apply to your server

---

## 11. Monitoring & Maintenance

### Log Rotation

```bash
sudo nano /etc/logrotate.d/openclaw
```

Add:

```
/var/log/openclaw/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 640 openclaw openclaw
    sharedscripts
    postrotate
        systemctl reload openclaw > /dev/null 2>&1 || true
    endscript
}
```

### Health Check Script

```bash
nano ~/health-check.sh
```

Add:

```bash
#!/bin/bash
# OpenClaw health check script

WEBHOOK_URL="https://your-monitoring-webhook"  # Optional: Discord/Slack webhook

check_service() {
    if ! systemctl is-active --quiet openclaw; then
        echo "OpenClaw service is down!"
        systemctl restart openclaw
        # curl -X POST -H "Content-Type: application/json" -d '{"content":"OpenClaw restarted"}' "$WEBHOOK_URL"
    fi
}

check_endpoint() {
    if ! curl -sf http://localhost:18789/health > /dev/null; then
        echo "OpenClaw health endpoint failed!"
        systemctl restart openclaw
    fi
}

check_disk() {
    USAGE=$(df -h /home | awk 'NR==2 {print $5}' | sed 's/%//')
    if [ "$USAGE" -gt 80 ]; then
        echo "Disk usage critical: ${USAGE}%"
    fi
}

check_service
check_endpoint
check_disk
```

```bash
chmod +x ~/health-check.sh

# Add to crontab
crontab -e
# Add: */5 * * * * /home/openclaw/health-check.sh >> /var/log/openclaw/health.log 2>&1
```

### Update Script

```bash
nano ~/update-openclaw.sh
```

Add:

```bash
#!/bin/bash
# Update OpenClaw safely

set -e

echo "Backing up config..."
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak

echo "Stopping service..."
sudo systemctl stop openclaw

echo "Updating OpenClaw..."
sudo npm update -g openclaw

echo "Starting service..."
sudo systemctl start openclaw

echo "Checking health..."
sleep 5
if systemctl is-active --quiet openclaw; then
    echo "Update successful!"
else
    echo "Update failed, rolling back..."
    sudo systemctl start openclaw
fi
```

```bash
chmod +x ~/update-openclaw.sh
```

---

## 12. Security Checklist

### Pre-Deployment

- [ ] SSH key-only authentication enabled
- [ ] Root login disabled
- [ ] SSH on non-standard port
- [ ] Fail2ban configured and running
- [ ] UFW firewall enabled
- [ ] Hetzner Cloud Firewall configured
- [ ] Automatic security updates enabled

### OpenClaw Configuration

- [ ] `OPENCLAW_GATEWAY_TOKEN` set (strong random value)
- [ ] Gateway bound to `loopback` only
- [ ] `trustedProxies` configured with Cloudflare IPs
- [ ] `allowedOrigins` whitelist configured
- [ ] `dangerouslyDisableDeviceAuth: false`
- [ ] Logging redaction enabled
- [ ] Sensitive files have 600 permissions

### Cloudflare Configuration

- [ ] SSL mode set to "Full (strict)"
- [ ] Always Use HTTPS enabled
- [ ] Bot Fight Mode enabled
- [ ] Origin certificate installed
- [ ] WAF rules configured
- [ ] DNS records proxied (orange cloud)

### Nginx Configuration

- [ ] Rate limiting configured
- [ ] Connection limits set
- [ ] Real IP from Cloudflare configured
- [ ] Security headers added
- [ ] SSL properly configured

### Monitoring

- [ ] Health check script running
- [ ] Log rotation configured
- [ ] Disk space monitoring
- [ ] Service restart alerting (optional)

---

## Quick Reference Commands

```bash
# Service management
sudo systemctl start openclaw
sudo systemctl stop openclaw
sudo systemctl restart openclaw
sudo systemctl status openclaw

# View logs
sudo journalctl -u openclaw -f
tail -f /var/log/openclaw/openclaw.log

# Check nginx
sudo nginx -t
sudo systemctl reload nginx

# Check firewall
sudo ufw status
sudo fail2ban-client status sshd

# Update Cloudflare IPs (run monthly)
curl -s https://www.cloudflare.com/ips-v4 > /tmp/cf-ips.txt
curl -s https://www.cloudflare.com/ips-v6 >> /tmp/cf-ips.txt
```

---

## Troubleshooting

### Gateway won't start

```bash
# Check logs
sudo journalctl -u openclaw -n 50

# Verify config syntax
node -e "require('/home/openclaw/.openclaw/openclaw.json')"

# Check permissions
ls -la ~/.openclaw/
```

### 502 Bad Gateway

```bash
# Check if OpenClaw is running
sudo systemctl status openclaw

# Check if port is listening
ss -tlnp | grep 18789

# Check nginx logs
sudo tail -f /var/log/nginx/error.log
```

### Rate limiting too aggressive

Adjust nginx rate limits:

```nginx
limit_req zone=api_limit burst=50 nodelay;  # Increase burst
```

### Cloudflare connection issues

1. Verify origin certificate is valid
2. Check SSL mode is "Full (strict)"
3. Verify nginx is listening on 443
4. Check Cloudflare → Origin connection in dashboard

---

## Cost Summary

| Service                | Monthly Cost     |
| ---------------------- | ---------------- |
| Hetzner CX22           | ~€4.50           |
| Hetzner Backups (+20%) | ~€0.90           |
| Cloudflare (Free tier) | €0               |
| Domain (~€10/year)     | ~€0.83           |
| **Total**              | **~€6.23/month** |

Plus API costs (with cost-optimized config):

- Light usage: $5-15/month
- Medium usage: $15-30/month
- Heavy usage: $30-50/month
