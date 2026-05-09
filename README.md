# 🏠 edy's Homelab

A self-hosted homelab running on a Lenovo ThinkCentre Mini, built with Proxmox VE. This repo documents every service, configuration, and lesson learned.
 
<img width="1810" height="941" alt="Image" src="https://github.com/user-attachments/assets/c4ed62bf-83b6-4a71-898c-d363d4862dc3" />


---

## 🖥️ Hardware

| Component | Details |
|---|---|
| **Machine** | Lenovo ThinkCentre Mini |
| **CPU** | Intel Core i3-6100T (Intel HD Graphics 530) |
| **RAM** | 8GB |
| **Storage** | ~100GB SSD (Proxmox + CTs) + 115GB USB Drive (Media) |
| **OS** | Proxmox VE |

---

## 🗺️ Architecture

```
Internet
    │
    ▼
Cloudflare (edyase.me)
    │
    ▼
Cloudflare Tunnel
    │
    ▼
NPM CT (192.168.1.6)
    │
    ├──► Jellyfin (192.168.1.3:8096)
    ├──► Seerr (192.168.1.5:5055)
    └──► Grafana (192.168.1.7:3000)

Home Network (192.168.1.0/24)
    │
    ├── Pi-hole + Tailscale (192.168.1.2)
    ├── Jellyfin (192.168.1.3)
    ├── Arr Stack (192.168.1.4)
    │   ├── qBittorrent :8080
    │   ├── Prowlarr :9696
    │   └── Sonarr :8989
    ├── Seerr (192.168.1.5)
    ├── NPM + Cloudflared (192.168.1.6)
    └── Grafana + Prometheus (192.168.1.7)

Media Flow:
Jellyfin Enhanced → Seerr → Sonarr → qBittorrent → USB Drive → Jellyfin
```
---

## 🌐 Network

| Setting | Value |
|---|---|
| **Router IP** | 192.168.1.1 |
| **DHCP Pool** | 192.168.1.128 - 192.168.1.254 |
| **Static IP Range** | 192.168.1.2 - 192.168.1.10 |
| **Domain** | edyase.me (Namecheap via GitHub Student Pack) |
| **DNS** | Cloudflare |

---

## 📦 Services

| CT ID | IP | Service | Port(s) |
|---|---|---|---|
| 100 | 192.168.1.2 | Pi-hole + Tailscale | 80 |
| 101 | 192.168.1.3 | Jellyfin | 8096 |
| 102 | 192.168.1.4 | qBittorrent + Prowlarr + Sonarr | 8080 / 9696 / 8989 |
| 103 | 192.168.1.5 | Seerr | 5055 |
| 104 | 192.168.1.6 | Nginx Proxy Manager + Cloudflared | 80 / 81 / 443 |
| 105 | 192.168.1.7 | Grafana + Prometheus | 3000 / 9090 |

---

## 🔧 Services Detail

### Pi-hole (CT 100)
Network-wide ad blocking via DNS filtering.


**Install:**
```bash
curl -sSL https://install.pi-hole.net | bash
```

**Config:**
- Upstream DNS: Cloudflare (1.1.1.1)
- Blocklists: OISD Big + HaGeZi Pro
- Web UI: `http://192.168.1.2/admin`

**Screenshot:**

<img width="1248" height="940" alt="Image" src="https://github.com/user-attachments/assets/103eba48-c69d-484d-a57e-ced840fbe5d7" />



---

### Tailscale (CT 100 — same as Pi-hole)
Secure remote access VPN. Runs alongside Pi-hole.

**Install:**
```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up --advertise-routes=192.168.1.0/24 --accept-routes
```

**Enable IP forwarding (Proxmox host):**
```bash
echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
echo 'net.ipv6.conf.all.forwarding = 1' >> /etc/sysctl.conf
sysctl -p
```

**Fix for unprivileged LXC (Proxmox host):**
Add to `/etc/pve/lxc/100.conf`:
```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```

**Approve subnet route:**
Go to https://login.tailscale.com/admin/machines → Edit route settings → Enable 192.168.1.0/24

---

### Jellyfin (CT 101)
Media server for anime and movies.

**Install:**
```bash
curl -fsSL https://repo.jellyfin.org/jellyfin_team.gpg.key | gpg --dearmor -o /usr/share/keyrings/jellyfin.gpg
echo "deb [signed-by=/usr/share/keyrings/jellyfin.gpg] https://repo.jellyfin.org/ubuntu noble main" | tee /etc/apt/sources.list.d/jellyfin.list
apt update && apt install -y jellyfin
```

**CT must be privileged** for Intel QuickSync hardware transcoding (i3-6100T has Intel HD 530).

**Web UI:** `http://192.168.1.3:8096`

**Plugins installed:**
- Jellyfin Enhanced (adds Seerr integration)
  - Repo: `https://raw.githubusercontent.com/n00bcodr/jellyfin-plugins/main/10.11/manifest.json`

**USB drive passthrough:**
Add to `/etc/pve/lxc/101.conf`:
```
mp0: /mnt/usb,mp=/mnt/usb
```
**Screenshot:**
<img width="1810" height="941" alt="Image" src="https://github.com/user-attachments/assets/455c1014-0819-4c7f-9cf0-011bc0402499" />


---

### Arr Stack (CT 102)
Automated media downloading stack.

#### qBittorrent
**Install:**
```bash
apt install -y qbittorrent-nox
```

**Service file** `/etc/systemd/system/qbittorrent-nox.service`:
```ini
[Unit]
Description=qBittorrent-nox
After=network.target

[Service]
Type=simple
User=qbittorrent
ExecStart=/usr/bin/qbittorrent-nox
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

**Web UI:** `http://192.168.1.4:8080`
**Download path:** `/mnt/usb/downloads`

#### Prowlarr
**Install:** Follow https://wiki.servarr.com/prowlarr/installation/linux

**Web UI:** `http://192.168.1.4:9696`

**Indexers added:**
- Nyaa (best anime torrent indexer)

#### Sonarr
**Install:**
```bash
apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 2009837CBFFD68F45BC180471F4F90DE2A9B4BF8
echo "deb https://apt.sonarr.tv/ubuntu focal main" | tee /etc/apt/sources.list.d/sonarr.list
apt update && apt install -y sonarr
```

**Web UI:** `http://192.168.1.4:8989`
**Root folder:** `/mnt/usb/media/anime`
**Download client:** qBittorrent at `192.168.1.4:8080`

**USB drive passthrough:**
Add to `/etc/pve/lxc/102.conf`:
```
mp0: /mnt/usb,mp=/mnt/usb
```

**Fix permissions for USB:**
```bash
# Get sonarr UID from CT
cat /etc/passwd | grep sonarr  # e.g. UID 995
# On Proxmox host (offset by 100000 for unprivileged CT)
chown -R 100995:100988 /mnt/usb/media
chmod -R 775 /mnt/usb/media
```

---

### Seerr (CT 103)
Media request interface. Connects Jellyfin to Sonarr/Radarr.

**Install (Docker):**
```bash
curl -fsSL https://get.docker.com | bash
mkdir -p /etc/seerr && chmod 777 /etc/seerr
docker run -d \
  --name seerr \
  --restart unless-stopped \
  -p 5055:5055 \
  -v /etc/seerr:/app/config \
  ghcr.io/seerr-team/seerr
```

**Web UI:** `http://192.168.1.5:5055`

**Connect to Jellyfin:**
- Settings → Media Server → Jellyfin → `http://192.168.1.3:8096`

**Connect to Sonarr:**
- Settings → Services → Sonarr → `http://192.168.1.4:8989`

**Jellyfin Enhanced integration:**
- In Jellyfin Enhanced plugin settings → Seerr tab
- Set URL: `http://192.168.1.5:5055`
- Set API Key from Seerr → Settings → General

---

### Nginx Proxy Manager + Cloudflare Tunnel (CT 104)
Reverse proxy with external HTTPS access via Cloudflare Tunnel. Bypasses CGNAT.

**Install NPM (Docker):**
```bash
curl -fsSL https://get.docker.com | bash
mkdir -p /opt/npm && cd /opt/npm
```

`docker-compose.yml`:
```yaml
services:
  npm:
    image: jc21/nginx-proxy-manager:latest
    restart: unless-stopped
    ports:
      - 80:80
      - 81:81
      - 443:443
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
```

```bash
docker compose up -d
```

**Web UI:** `http://192.168.1.6:81`
Default login: `admin@example.com` / `changeme`

**Install Cloudflared:**
```bash
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-public-v2.gpg | sudo tee /usr/share/keyrings/cloudflare-public-v2.gpg >/dev/null
echo 'deb [signed-by=/usr/share/keyrings/cloudflare-public-v2.gpg] https://pkg.cloudflare.com/cloudflared any main' | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt-get update && sudo apt-get install cloudflared
```

**Create tunnel:**
1. Go to https://one.dash.cloudflare.com → Networks → Tunnels
2. Create tunnel → Cloudflared → name it `homelab`
3. Run the install command provided
4. Add public hostname routes for each service

**Subdomains configured:**
| Subdomain | Service | Internal URL |
|---|---|---|
| jellyfin.edyase.me | Jellyfin | http://192.168.1.3:8096 |

**DNS records** (auto-created by Cloudflare Tunnel):
- CNAME `jellyfin` → `homelab` tunnel (Proxied)

---

### Grafana + Prometheus (CT 105)
Monitoring dashboards for all services and hosts.

**Install:**
```bash
wget -q -O /usr/share/keyrings/grafana.key https://apt.grafana.com/gpg.key
echo "deb [signed-by=/usr/share/keyrings/grafana.key] https://apt.grafana.com stable main" | tee /etc/apt/sources.list.d/grafana.list
apt update && apt install -y grafana prometheus
systemctl enable grafana-server prometheus --now
```

**Web UI:** `http://192.168.1.7:3000`
Default login: `admin` / `admin`

**Prometheus config** `/etc/prometheus/prometheus.yml`:
```yaml
scrape_configs:
  - job_name: 'proxmox'
    static_configs:
      - targets: ['192.168.1.210:9100']

  - job_name: 'pihole'
    static_configs:
      - targets: ['192.168.1.2:9100']

  - job_name: 'jellyfin'
    static_configs:
      - targets: ['192.168.1.3:9100']

  - job_name: 'arrstack'
    static_configs:
      - targets: ['192.168.1.4:9100']

  - job_name: 'seerr'
    static_configs:
      - targets: ['192.168.1.5:9100']

  - job_name: 'nginx'
    static_configs:
      - targets: ['192.168.1.6:9100']
```

**Install Node Exporter on each CT and Proxmox host:**
```bash
apt install -y prometheus-node-exporter
systemctl enable prometheus-node-exporter --now
```

**Dashboard:** Import ID `1860` (Node Exporter Full) from Grafana dashboard library.


**Screenshot:**
<img width="1512" height="866" alt="Image" src="https://github.com/user-attachments/assets/572d5274-e8d7-478b-b09d-58052564accf" />


---

## 💾 USB Drive Setup

```bash
# On Proxmox host
fdisk /dev/sdb  # create partition
mkfs.ext4 /dev/sdb1
mkdir /mnt/usb
mount /dev/sdb1 /mnt/usb

# Add to /etc/fstab for persistence
UUID=your-uuid /mnt/usb ext4 defaults 0 2

# Create media folders
mkdir -p /mnt/usb/media/anime
mkdir -p /mnt/usb/media/movies
mkdir -p /mnt/usb/downloads
```

---

## 🔒 Common Fixes

### Gateway missing after CT reboot
Edit `/etc/pve/lxc/<CTID>.conf` on Proxmox host and add `gw=192.168.1.1` to the net0 line:
```
net0: name=eth0,bridge=vmbr0,gw=192.168.1.1,ip=192.168.1.X/24,type=veth
```

### DNS missing after CT reboot
Add to `/etc/pve/lxc/<CTID>.conf`:
```
nameserver: 1.1.1.1
```

### Tailscale not running in LXC
Enable IP forwarding on Proxmox host:
```bash
echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
echo 'net.ipv6.conf.all.forwarding = 1' >> /etc/sysctl.conf
sysctl -p
```

### Sonarr can't write to USB
Fix permissions using the CT's mapped UID (offset by 100000):
```bash
# Get UID from inside CT
cat /etc/passwd | grep sonarr
# Apply on Proxmox host (add 100000 to UID)
chown -R 100995:100988 /mnt/usb/media
chmod -R 775 /mnt/usb/media
```

---

## 📋 To Do
- [ ] Radarr — movie automation
- [ ] Vaultwarden — self hosted password manager
- [ ] More Cloudflare subdomains (seerr, grafana, sonarr)
- [ ] Fix local DNS resolution for edyase.me subdomains on Arch
- [ ] Home Assistant
- [ ] Ollama — local AI

---

## Thank you for checking my homelab out!
