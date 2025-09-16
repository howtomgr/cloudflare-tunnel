# Cloudflare Tunnel Installation Guide

Secure way to connect your applications to Cloudflare without opening inbound ports. Essential tool for exposing self-hosted services securely through Cloudflare's global network.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Supported Operating Systems](#supported-operating-systems)
3. [Installation](#installation)
4. [Configuration](#configuration)
5. [Service Management](#service-management)
6. [Troubleshooting](#troubleshooting)
7. [Security Considerations](#security-considerations)
8. [Performance Tuning](#performance-tuning)
9. [Backup and Restore](#backup-and-restore)
10. [System Requirements](#system-requirements)
11. [Support](#support)
12. [Contributing](#contributing)
13. [License](#license)
14. [Acknowledgments](#acknowledgments)
15. [Version History](#version-history)
16. [Appendices](#appendices)

## 1. Prerequisites

- Linux system (any modern distribution)
- Root or sudo access
- Cloudflare account with domain configured
- Applications running locally that need external access


## 2. Supported Operating Systems

This guide supports installation on:
- RHEL 8/9 and derivatives (CentOS Stream, Rocky Linux, AlmaLinux)
- Debian 11/12
- Ubuntu 20.04/22.04/24.04 LTS
- Arch Linux (rolling release)
- Alpine Linux 3.18+
- openSUSE Leap 15.5+ / Tumbleweed
- SUSE Linux Enterprise Server (SLES) 15+
- macOS 12+ (Monterey and later) 
- FreeBSD 13+
- Windows 10/11/Server 2019+ (where applicable)

## 3. Installation

### Ubuntu/Debian
```bash
# Download cloudflared
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64
sudo mv cloudflared-linux-amd64 /usr/local/bin/cloudflared
sudo chmod +x /usr/local/bin/cloudflared

# Verify installation
cloudflared --version
```

### RHEL/CentOS/Rocky Linux/AlmaLinux
```bash
# Download and install cloudflared
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64
sudo mv cloudflared-linux-amd64 /usr/local/bin/cloudflared
sudo chmod +x /usr/local/bin/cloudflared

# Verify installation
cloudflared --version
```

### Docker Installation
```bash
# Run cloudflared in Docker
mkdir -p ~/.cloudflared

docker run -d \
  --name cloudflared \
  --restart unless-stopped \
  -v ~/.cloudflared:/etc/cloudflared \
  cloudflare/cloudflared:latest \
  tunnel run
```

## 4. Configuration

### Tunnel Setup
```bash
# Authenticate with Cloudflare
cloudflared tunnel login

# Create tunnel
cloudflared tunnel create myserver-tunnel

# List tunnels
cloudflared tunnel list

# Create configuration file
sudo mkdir -p /etc/cloudflared
sudo tee /etc/cloudflared/config.yml > /dev/null <<EOF
# Cloudflare Tunnel Configuration

tunnel: your-tunnel-id-here
credentials-file: /etc/cloudflared/your-tunnel-id.json

# Ingress rules
ingress:
  # Home Assistant
  - hostname: homeassistant.example.com
    service: http://localhost:8123
    
  # Nextcloud
  - hostname: nextcloud.example.com
    service: http://localhost:80
    
  # Grafana
  - hostname: grafana.example.com
    service: http://localhost:3000
    
  # Jellyfin media server
  - hostname: jellyfin.example.com
    service: http://localhost:8096
    
  # Default catch-all (required)
  - service: http_status:404

# Global configuration
warp-routing:
  enabled: true
EOF

# Copy tunnel credentials
sudo cp ~/.cloudflared/your-tunnel-id.json /etc/cloudflared/
sudo chown cloudflared:cloudflared /etc/cloudflared/*
sudo chmod 600 /etc/cloudflared/*.json
```

### DNS Configuration
```bash
# Create DNS records for tunnel
cloudflared tunnel route dns myserver-tunnel homeassistant.example.com
cloudflared tunnel route dns myserver-tunnel nextcloud.example.com
cloudflared tunnel route dns myserver-tunnel grafana.example.com
cloudflared tunnel route dns myserver-tunnel jellyfin.example.com

# Verify DNS records
dig homeassistant.example.com
```

### SystemD Service
```bash
# Install as system service
sudo cloudflared service install

# Create custom service with security
sudo tee /etc/systemd/system/cloudflared.service > /dev/null <<EOF
[Unit]
Description=Cloudflare Tunnel
After=network.target

[Service]
Type=simple
User=cloudflared
Group=cloudflared
ExecStart=/usr/local/bin/cloudflared tunnel run
Restart=always
RestartSec=5s
StandardOutput=journal
StandardError=journal

# Security settings
NoNewPrivileges=true
PrivateTmp=true
ProtectHome=true
ProtectSystem=strict
ReadWritePaths=/var/log/cloudflared

[Install]
WantedBy=multi-user.target
EOF

# Create cloudflared user
sudo useradd --system --shell /bin/false cloudflared
sudo chown -R cloudflared:cloudflared /etc/cloudflared

sudo systemctl daemon-reload
sudo systemctl enable --now cloudflared
```

## Security Configuration

### Access Control with Cloudflare Access
```bash
# Configure Cloudflare Access policies via API or Dashboard
# Example: Restrict access to specific applications

# Create access policy for sensitive services
cat > access-policy.json <<EOF
{
  "name": "Admin Access Only",
  "decision": "allow",
  "include": [
    {
      "email": ["admin@example.com"]
    }
  ],
  "require": [
    {
      "any_valid_service_token": {}
    }
  ],
  "exclude": []
}
EOF

# Apply to applications via Cloudflare dashboard:
# Security > Cloudflare Access > Applications
```

### Network Security
```bash
# Configure local firewall (services now accessible only via Cloudflare)
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
# Don't open ports 80/443/8096 etc. - only accessible via tunnel

# Monitor tunnel connections
sudo tee /usr/local/bin/tunnel-monitor.sh > /dev/null <<'EOF'
#!/bin/bash
LOG="/var/log/tunnel-monitor.log"

log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a ${LOG}
}

# Check tunnel status
if systemctl is-active cloudflared >/dev/null; then
    log_message "✓ Cloudflare tunnel running"
else
    log_message "✗ Cloudflare tunnel not running"
fi

# Check tunnel connectivity
if cloudflared tunnel info myserver-tunnel >/dev/null 2>&1; then
    log_message "✓ Tunnel connectivity OK"
else
    log_message "⚠ Tunnel connectivity issues"
fi

log_message "Tunnel monitoring completed"
EOF

sudo chmod +x /usr/local/bin/tunnel-monitor.sh
echo "*/10 * * * * root /usr/local/bin/tunnel-monitor.sh" | sudo tee -a /etc/crontab
```

## Additional Resources

- [Cloudflare Tunnel Documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)
- [Cloudflare Zero Trust](https://developers.cloudflare.com/cloudflare-one/)

---

**Note:** This guide is part of the [HowToMgr](https://howtomgr.github.io) collection.