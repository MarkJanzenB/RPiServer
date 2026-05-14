# Raspberry Pi Home Server Setup

Complete workflow for installing Pi-hole + Unbound + Tailscale on Raspberry Pi 4/5.

## Hardware
- Raspberry Pi 4 or 5
- SD Card (32GB+)
- Ethernet connection recommended

## Initial Setup

### 1. Flash Raspberry Pi OS
```bash
# Using Raspberry Pi Imager or balenaEtcher
# Download from: https://www.raspberrypi.com/software/
# Select: Raspberry Pi OS (64-bit) Lite or Desktop
```

### 2. Enable SSH & Configure WiFi (Headless)
Create `ssh` file and `wpa_supplicant.conf` on boot partition:

**wpa_supplicant.conf:**
```conf
country=US
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1

network={
    ssid="YourNetworkName"
    psk="YourPassword"
}
```

**SSH:**
```bash
touch /Volumes/boot/ssh
```

### 3. First Boot & Update
```bash
ssh pi@raspberrypi.local
# default password: raspberry

# Update system
sudo apt update && sudo apt upgrade -y
sudo reboot
```

### 4. Disable Wi-Fi (Ethernet-only)
```bash
# Delete Wi-Fi connection profile
nmcli con show
sudo nmcli con delete <wifi-connection-name>

# Optional: hard block Wi-Fi at kernel level
echo 'dtoverlay=disable-wifi' | sudo tee -a /boot/config.txt
```

### 5. Set Static IP

**Method A — Raspberry Pi OS (dhcpcd):**
```bash
sudo nano /etc/dhcpcd.conf
```

Add at end:
```conf
interface eth0
static ip_address=192.168.254.99/24
static routers=192.168.254.254
static domain_name_servers=192.168.254.254
```

**Method B — Debian 13+ (NetworkManager — used in this guide):**
```bash
# Check router gateway first
ip route show default

# Use nmtui (interactive) or nmcli (direct)
# Find your connection name first:
nmcli con show

# Set static IP via nmcli (replace UUID from above)
sudo nmcli con mod <uuid> \
  ipv4.method manual \
  ipv4.addresses 192.168.254.99/24 \
  ipv4.gateway 192.168.254.254 \
  ipv4.dns "1.1.1.1,8.8.8.8" \
  ipv4.ignore-auto-dns yes

# Apply
sudo nmcli con up <uuid>
```

> **Important:** Find your router's gateway first — common values are `192.168.254.254`, `192.168.1.1`, `192.168.0.1`. Run `ip route` and look for the `default via <ip>` line. The old guide used `192.168.254.1` which is wrong for most GlobeAtHome routers.

---

## Install Services

### 1. Install Tailscale
```bash
curl -fsSL https://tailscale.com/install.sh | sh

# Enable and start
sudo systemctl enable --now tailscaled

# Authenticate (run on Pi, open URL)
sudo tailscale up --operator=mark
```

### 2. Install Unbound
```bash
sudo apt install -y unbound

# Create port configuration
echo "server:
    port: 5335" | sudo tee /etc/unbound/unbound.conf.d/port.conf

# Restart
sudo systemctl restart unbound

# Verify
sudo ss -tlnp | grep 5335
```

### 3. Install Pi-hole (Manual Method)

```bash
# Install dependencies
sudo apt install -y curl git lsb-release ca-certificates jq

# Download and run installer
curl -sSL https://install.pi-hole.net | bash

# Follow prompts:
# - Static IP: Use current IP (e.g. 192.168.254.99)
# - Upstream DNS: Google/Cloudflare (will change later)
# - Install web interface: Yes
# - Log queries: Yes
```

---

## Configuration

### 1. Fix Pi-hole Config (Post-Install)

Edit `/etc/pihole/pihole.toml`:

```toml
[dns]
upstreams = [ "127.0.0.1#5335" ]  # Unbound on port 5335
interface = "eth0"
listeningMode = "ALL"              # Accept all origins (Tailscale)
```

```bash
# Fix permissions
sudo touch /etc/pihole/versions
sudo chown pihole:pihole /etc/pihole/versions
sudo chmod 644 /etc/pihole/versions
sudo chown pihole:pihole /etc/pihole/pihole-FTL.db
sudo chmod 644 /etc/pihole/pihole-FTL.db

# Restart
sudo systemctl restart pihole-FTL
```

### 2. Set Web Password
```bash
sudo pihole setpassword '320240123'
```

### 3. Enable Tailscale DNS

**In Tailscale Admin Console** (https://login.tailscale.com/admin):
1. Go to DNS settings
2. Add nameserver: `<homeserver-tailscale-ip>` (e.g. `100.111.98.30`)
3. Remove old/offline nameservers

### 4. System DNS → Pi-hole + Unbound

After Pi-hole is running, point the Pi itself at its own Pi-hole:

```bash
# Stop Tailscale from managing resolv.conf
sudo tailscale set --accept-dns=false

# Set system DNS to local Pi-hole via NetworkManager
sudo nmcli con mod <ethernet-uuid> ipv4.dns "127.0.0.1"
sudo nmcli con up <ethernet-uuid>

# Verify
cat /etc/resolv.conf
# Should show: nameserver 127.0.0.1 (managed by NetworkManager)
dig google.com
# Should resolve successfully
```

Now the DNS chain is: `App → 127.0.0.1 → Pi-hole → 127.0.0.1#5335 → Unbound → root servers`

---

## Network Configuration

### Router DNS Settings
Set your router's DNS to:
```
192.168.254.99
```

### Device DNS
| Device Type | DNS Server |
|-------------|-------------|
| Home network | 192.168.254.99 |
| Tailscale | 100.111.98.30 or homeserver.tailcbdea4.ts.net |

---

## Access

| Service | Local URL | Tailscale URL |
|---------|------------|---------------|
| Pi-hole Admin | http://192.168.254.99/admin | http://100.111.98.30/admin |
| Pi-hole API | http://192.168.254.99/admin/api.php | http://100.111.98.30/admin/api.php |

**Login:** `320240123`

---

## Verify Services

```bash
# Check all services
systemctl status tailscaled
systemctl status unbound
systemctl status pihole-FTL

# Test DNS resolution
dig @127.0.0.1 google.com
nslookup pi.hole 192.168.254.99

# Check ports
ss -tlnp | grep -E '53|5335|80|443'

# Pi-hole status
pihole status
```

---

## Troubleshooting

### Pi-hole web UI shows "Database not available"
```bash
sudo chown pihole:pihole /etc/pihole/pihole-FTL.db
sudo chmod 644 /etc/pihole/pihole-FTL.db
sudo systemctl restart pihole-FTL
```

### NTP errors in Pi-hole
Disable NTP in `/etc/pihole/pihole.toml`:
```toml
ntp.sync.active = false
ntp.ipv4.active = false
ntp.ipv6.active = false
```

### DNS not resolving
```bash
# Check upstream
dig @127.0.0.1 google.com

# If failing, check Unbound
systemctl status unbound
sudo ss -tlnp | grep 5335
```

---

## Backup/Restore

### Backup Pi-hole
```bash
# Database
sudo cp /etc/pihole/pihole-FTL.db ~/pihole-backup.db

# Config
sudo cp /etc/pihole/pihole.toml ~/pihole-backup.toml

# Blocklists
sudo cp /etc/pihole/adlists.list ~/adlists-backup.list
```

### Restore
```bash
# Stop FTL
sudo systemctl stop pihole-FTL

# Restore files
sudo cp ~/pihole-backup.db /etc/pihole/pihole-FTL.db
sudo cp ~/pihole-backup.toml /etc/pihole/pihole.toml
sudo cp ~/adlists-backup.list /etc/pihole/adlists.list

# Permissions
sudo chown pihole:pihole /etc/pihole/pihole-FTL.db

# Restart
sudo systemctl start pihole-FTL
```