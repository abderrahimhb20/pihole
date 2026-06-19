Here's a complete guide to installing **Pi-hole** and **Snort** on Ubuntu Server 22.04.

---

## Pi-hole Installation

### 1. Update the system
```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Set a static IP (recommended before Pi-hole)
```bash
sudo nano /etc/netplan/00-installer-config.yaml
```
```yaml
network:
  ethernets:
    eth0:
      dhcp4: no
      addresses: [192.168.1.10/24]
      gateway4: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
  version: 2
```
```bash
sudo netplan apply
```

### 3. Install Pi-hole
```bash
curl -sSL https://install.pi-hole.net | bash
```
The interactive installer will walk you through:
- Choosing your network interface
- Selecting an upstream DNS provider (e.g. Cloudflare, Google)
- Enabling the web admin interface
- Installing the web server (lighttpd)

### 4. Set the admin password
```bash
pihole -a -p
```

### 5. Access the dashboard
Open a browser: `http://<your-server-ip>/admin`

---

## Snort 3 Installation

### 1. Install dependencies
```bash
sudo apt install -y build-essential libpcap-dev libpcre3-dev \
  libdumbnet-dev bison flex zlib1g-dev liblzma-dev \
  openssl libssl-dev libnghttp2-dev libluajit-5.1-dev \
  pkg-config cmake git
```

### 2. Install DAQ (Data Acquisition library)
```bash
cd /tmp
git clone https://github.com/snort3/libdaq.git
cd libdaq
./bootstrap
./configure
make
sudo make install
sudo ldconfig
```

### 3. Install Snort 3
```bash
cd /tmp
git clone https://github.com/snort3/snort3.git
cd snort3
./configure_cmake.sh --prefix=/usr/local --enable-tcmalloc
cd build
make -j$(nproc)
sudo make install
sudo ldconfig
```

### 4. Verify installation
```bash
snort --version
```

### 5. Create directory structure
```bash
sudo mkdir -p /etc/snort/rules
sudo mkdir -p /var/log/snort
sudo touch /etc/snort/rules/local.rules
```

### 6. Create a basic config (`/etc/snort/snort.lua`)
```bash
sudo nano /etc/snort/snort.lua
```
```lua
-- Basic Snort 3 config
HOME_NET = '192.168.1.0/24'
EXTERNAL_NET = '!$HOME_NET'

ips =
{
  include = RULE_PATH .. '/local.rules',
  enable_builtin_rules = true,
}

alert_fast = { file = true }
```

### 7. Add a test rule
```bash
echo 'alert icmp any any -> $HOME_NET any (msg:"ICMP Test"; sid:1000001; rev:1;)' \
  | sudo tee /etc/snort/rules/local.rules
```

### 8. Test the config
```bash
sudo snort -c /etc/snort/snort.lua --daq-dir /usr/local/lib/daq -T
```

### 9. Run Snort
```bash
sudo snort -c /etc/snort/snort.lua -i eth0 -A alert_fast -l /var/log/snort
```

### 10. Create a systemd service
```bash
sudo nano /etc/systemd/system/snort.service
```
```ini
[Unit]
Description=Snort3 IDS
After=network.target

[Service]
ExecStart=/usr/local/bin/snort -c /etc/snort/snort.lua -i eth0 -A alert_fast -l /var/log/snort -D
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now snort
sudo systemctl status snort
```

---

## Integration Tip

Point your router's **DNS to the Pi-hole IP** so it filters DNS queries network-wide. Snort monitors traffic on the **same interface** (`eth0`) — together they give you DNS-level ad/tracker blocking plus network intrusion detection on a single Ubuntu server.

Let me know if you want to add community Snort rules (Emerging Threats), set up logging to a SIEM, or configure Pi-hole with custom blocklists.
