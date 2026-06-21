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
The error is a **permissions problem** on `/etc/pihole/versions`. Fix it:

— Fix the permissions
```bash
sudo chown -R pihole:pihole /etc/pihole
sudo chmod -R 755 /etc/pihole
```

 — Set the password
```bash
sudo pihole setpassword
```
> The command changed in Pi-hole v6 — it's `setpassword` now, not `-a -p`

 — Verify Pi-hole is running
```bash
sudo pihole status
```

---

If step 1 didn't fully fix it:
```bash
# Check who owns the file
ls -la /etc/pihole/

# Nuclear fix — reset all pihole file ownership
sudo chown -R pihole:pihole /etc/pihole /opt/pihole /var/log/pihole* 2>/dev/null
sudo systemctl restart pihole-FTL
```

Then retry `sudo pihole setpassword`.

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
`autoreconf` is missing. Install the autotools first:

```bash
sudo apt install -y autoconf automake libtool
```

Then retry the bootstrap and build:

```bash
cd /tmp
git clone https://github.com/snort3/libdaq.git
cd libdaq
./bootstrap
./configure
make -j$(nproc)
sudo make install
sudo ldconfig
```

You should see `configure` generate successfully and `make` compile without errors. Paste any output if it still fails.

### 3. Install Snort 3
Missing `libpcre2`. Install it and retry:

Install pcre2 + any other likely missing deps in one shot
```bash
sudo apt install -y libpcre2-dev libpcre3-dev libsafec-dev uuid-dev libmnl-dev
```


Then immediately continue with Snort3 (don't reboot):
```bash
cd /tmp
git clone https://github.com/snort3/snort3.git
cd snort3
./configure_cmake.sh --prefix=/usr/local
cd build
make -j$(nproc)
sudo make install
sudo ldconfig
snort --version
```

> **Tip:** Next time, build in a persistent directory like `~/src` instead of `/tmp` so reboots don't wipe your work:
> ```bash
> mkdir -p ~/src && cd ~/src
> ```

---

> CMake checks dependencies one at a time, so you may hit one more missing lib. If it fails again, just paste the next `CMake Error` line and I'll give you the exact `apt install` for it.

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

The config has a `RULE_PATH` variable that isn't defined. Fix the `snort.lua`:

**Step 1 — Rewrite the config correctly**
```bash
sudo tee /etc/snort/snort.lua << 'EOF'
-- Basic Snort 3 config

HOME_NET = '192.168.1.0/24'
EXTERNAL_NET = '!$HOME_NET'

ips =
{
  rules = [[
    include /etc/snort/rules/local.rules
  ]],
  enable_builtin_rules = true,
}

alert_fast = { file = true }
EOF
```

**Step 2 — Test the config**
```bash
sudo snort -c /etc/snort/snort.lua --daq-dir /usr/local/lib/daq -T
```

You should see `Snort successfully validated the configuration` at the end.

**Step 3 — Restart the service**
```bash
sudo systemctl restart snort
sudo systemctl status snort
```

---

> **Note:** In Snort 3, `RULE_PATH` is not a built-in variable — rules must be included with a full path using the `rules` key inside the `ips` block, not `include = RULE_PATH .. '/local.rules'`. That was the root cause of the error.
Good progress — config loads now. Two remaining issues: `$HOME_NET` undefined in the rules file, and `eth0` may not be your interface name. Fix both:

**Step 1 — Fix the rule (use literal IP, not variable)**
```bash
sudo tee /etc/snort/rules/local.rules << 'EOF'
alert icmp any any -> 192.168.1.0/24 any (msg:"ICMP Test"; sid:1000001; rev:1;)
EOF
```

**Step 2 — Check your actual interface name**
```bash
ip link show
```
It's likely `ens33`, `enp0s3`, or similar — not `eth0` on Ubuntu 22.04.

**Step 3 — Test config (should pass cleanly now)**
```bash
sudo snort -c /etc/snort/snort.lua --daq-dir /usr/local/lib/daq -T
```

**Step 4 — Update the systemd service with the correct interface**
```bash
# Replace eth0 with your actual interface, e.g. ens33
sudo sed -i 's/-i eth0/-i ens33/' /etc/systemd/system/snort.service
sudo systemctl daemon-reload
```

**Step 5 — Start and verify**
```bash
sudo systemctl restart snort
sudo systemctl status snort
```

---

> **Note:** Snort 3 rules don't expand Lua variables like `$HOME_NET` — that variable only works inside the Lua config itself, not in `.rules` files loaded via the `rules` block. Use literal CIDRs in rule files, or define them with `HOME_NET` in the `ips` block directly.
