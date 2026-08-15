# Minihyper N9 Self-Hosted Server Guide

> Turn your N9 into a file server (Samba), Git server (Gitea), and ad-blocker (Pi-hole), accessible from anywhere via Tailscale.
---

## What you'll need

- A **Minihyper N9** (or any mini PC)
- A **USB drive** (at least 4GB) for installing Ubuntu
- Another **computer** to create the install USB and follow along
- **2-3 Ethernet cables** (Cat5e or Cat6)
  - 1x XB7 to Netgear
  - 1x N9 to Netgear
  - 1x optional: main desktop to Netgear (if you want it wired)
- Your **Xfinity account** credentials (for the Xfinity app, if needed)
- Your **Netgear Nighthawk X4S** router

---

## Network overview

```
Internet
    |
    v
Xfinity XB7 (10.0.0.x, router mode, Xfinity Home cameras)
    |
    +-- Ethernet cable --> Netgear Nighthawk X4S WAN port
                              |
                              +-- N9 (wired, LAN port, 192.168.1.100)
                              |
                              +-- Your devices (WiFi or LAN, 192.168.1.x)
```

- **XB7**: stays in router mode so Xfinity Home cameras keep working. It acts as the modem + router for the Xfinity ecosystem.
- **Netgear Nighthawk X4S**: your main router. Handles WiFi, DHCP, and DNS for all your devices. DNS is pointed to Pi-hole on the N9.
- **N9**: wired to the Netgear. Runs Samba, Gitea, Pi-hole, and Tailscale.
- **Double NAT** (XB7 does NAT, Netgear does NAT) is invisible for normal use. Tailscale handles remote access without port forwarding.

---

## Step 1: Install Ubuntu Server on the N9

Ubuntu Server 24.04 LTS is recommended — most documented, most beginner-friendly.

1. On your main computer, download Ubuntu Server 24.04 LTS:
   https://ubuntu.com/download/server

2. Create a bootable USB:
   - **Windows**: use Rufus (https://rufus.ie/)
   - **macOS**: use `dd` or BalenaEtcher
   - **Linux**: use `dd` or Startup Disk Creator

3. Plug the USB into the N9, power it on, and press `F7` or `F12` repeatedly to enter the boot menu. Choose the USB drive.

4. Follow the installer:
   - Language: English
   - Keyboard: your layout
   - **Network**: connect Ethernet (preferred) or WiFi. Note the IP shown.
   - **Proxy**: leave blank
   - **Mirror**: leave default
   - **Storage**: use guided — entire disk
   - **Profile**: create a username and password. **Write these down.**
   - **Server name**: `n9-server` (or anything you like)
   - **SSH**: check **"Install OpenSSH server"** — important!
   - **Featured snaps**: uncheck everything
   - **Reboot** when done, remove the USB when prompted

5. After reboot, log in with your username and password.

6. Find the N9's IP address:
   ```bash
   ip a
   ```
   Look for an address like `192.168.1.x` under `eth0` or `enpXsY`. Write it down.

7. From your main computer, SSH into the N9:
   ```bash
   ssh yourusername@192.168.1.x
   ```
   (Replace with your actual username and IP)

> **From this point on, all commands run on the N9 via SSH.**

---

## Step 2: System updates and firewall

Update the system and set up a firewall to protect the N9:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install ufw -y
```

Open the ports we need:

```bash
sudo ufw allow OpenSSH
sudo ufw allow 445/tcp      # Samba
sudo ufw allow 139/tcp      # Samba
sudo ufw allow 3000/tcp     # Gitea
sudo ufw allow 80/tcp       # Pi-hole admin
# Replace enp2s0 with the N9's active LAN interface if different
sudo ufw allow in on enp2s0 from 192.168.1.0/24 to any port 53 proto udp
sudo ufw allow in on enp2s0 from 192.168.1.0/24 to any port 53 proto tcp
sudo ufw --force enable
```

Check it worked:

```bash
sudo ufw status
```

You should see all the ports listed as ALLOW.

---

## Step 3: Install Tailscale (secure remote access)

Tailscale creates a private encrypted network between your devices. It lets you access the N9 from anywhere — no port forwarding, no static IP, no dynamic DNS.

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

A URL will appear in the terminal. Open it in a browser on your main computer, log in with Google/Microsoft/GitHub/Apple, and authorize the device.

After that, find your N9's Tailscale IP:

```bash
tailscale ip -4
```

This prints an address like `100.x.x.x`. **Write this down — this is your N9's address from anywhere in the world.**

Test it from your main computer:

```bash
ping 100.x.x.x
```

It should respond. You're now connected through Tailscale.

> **What's happening**: Tailscale creates an encrypted WireGuard tunnel directly between your devices. It works through home routers, hotel WiFi, corporate firewalls, and cellular networks. No ports opened on your router.

Allow Pi-hole DNS queries from Tailscale clients:

```bash
sudo ufw allow in on tailscale0 to any port 53 proto udp
sudo ufw allow in on tailscale0 to any port 53 proto tcp
```

---

## Step 4: Configure systemd-resolved (Pi-hole owns DNS)

Pi-hole runs inside Docker and handles DNS. Keep systemd-resolved for per-interface DNS management, but disable its local stub listener so Pi-hole can own host port 53.

```bash
# Enable systemd-resolved (it's pre-installed but disabled on Ubuntu Server)
sudo systemctl enable --now systemd-resolved
```

Release port 53 for Pi-hole while keeping systemd-resolved enabled:

```bash
sudo mkdir -p /etc/systemd/resolved.conf.d
printf '%s\n' '[Resolve]' 'DNS=127.0.0.1' 'DNSStubListener=no' | sudo tee /etc/systemd/resolved.conf.d/pihole.conf
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
sudo systemctl restart systemd-resolved
```

Find your active ethernet interface:

```bash
ip link show | grep -E '^[0-9]:' | grep -v lo | grep -v docker | grep -v tailscale | grep -v veth | grep -v br-
```

Look for the `UP` interface — typically `enp2s0` or `enpXsY`. Replace `enp2s0` below with whatever yours is called:

```bash
# Set Pi-hole on localhost as the DNS server for this interface
sudo resolvectl dns enp2s0 127.0.0.1

# Apply DNS to all domains (".~" means "match everything")
sudo resolvectl domain enp2s0 "~."
```

Verify DNS resolution works:

```bash
resolvectl query google.com
ping -c 1 google.com
```

> **Why this approach instead of disabling systemd-resolved?** The older method (disabling systemd-resolved and hardcoding `1.1.1.1` in `/etc/resolv.conf`) bypasses per-interface DNS management. Keeping systemd-resolved enabled preserves per-interface configuration and caching, while `DNSStubListener=no` releases port 53 for Pi-hole. The `/run/systemd/resolve/resolv.conf` link points local applications at Pi-hole on `127.0.0.1`.

---

## Step 5: Install Docker (runs Gitea and Pi-hole)

Docker runs applications in isolated containers, keeping your system clean.

```bash
# Install prerequisites
sudo apt install ca-certificates curl -y

# Add Docker's official GPG key
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add Docker's repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

# Add your user to the docker group (so you don't need sudo for every command)
sudo usermod -aG docker $USER

# Apply the group change immediately
newgrp docker
```

Test that Docker works:

```bash
docker run hello-world
```

You should see a welcome message. If you do, Docker is installed correctly.

---

## Step 6: Create storage directories

Organize your data:

```bash
sudo mkdir -p /srv/files            # Your main file share
sudo mkdir -p /srv/docker/gitea     # Gitea data
sudo mkdir -p /srv/docker/pihole    # Pi-hole data

# Set ownership so you can read and write
sudo chown -R $USER:$USER /srv
```

---

## Step 7: Set up Samba (network file share)

Samba creates a network drive that you can mount on your main computer like a local folder.

```bash
sudo apt install samba -y
```

Edit the Samba config:

```bash
sudo nano /etc/samba/smb.conf
```

Scroll to the very bottom of the file and add:

```ini
[files]
   path = /srv/files
   browseable = yes
   read only = no
   guest ok = no
   valid users = yourusername
   force user = yourusername
```

> Replace `yourusername` with the username you created during Ubuntu install.

Save (`Ctrl+O`, `Enter`) and exit (`Ctrl+X`).

Set your Samba password (this can be different from your Ubuntu login password):

```bash
sudo smbpasswd -a yourusername
```

Restart Samba to apply changes:

```bash
sudo systemctl restart smbd
```

Verify the config is valid:

```bash
testparm
```

No errors means you're good.

---

## Step 8: Set up Gitea (Git server) with Docker

Gitea is a lightweight, self-hosted Git server — like your own private GitHub.

Create the Docker Compose config:

```bash
nano /srv/docker/gitea/docker-compose.yml
```

Paste this:

```yaml
services:
  gitea:
    image: gitea/gitea:latest
    container_name: gitea
    restart: always
    ports:
      - "3000:3000"
      - "222:22"
    volumes:
      - /srv/docker/gitea/data:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    environment:
      - USER_UID=1000
      - USER_GID=1000
```

Save and exit.

Start Gitea:

```bash
docker compose -f /srv/docker/gitea/docker-compose.yml up -d
```

Wait about 30 seconds, then open your browser and go to:

```
http://100.x.x.x:3000
```

(Replace with your N9's Tailscale IP.)

You'll see Gitea's setup page. Click **"Install Gitea"** at the bottom — the defaults are fine (SQLite database, etc.).

After installation, create a user account. Now you can create repos and push code:

```bash
git remote add origin http://100.x.x.x:3000/youruser/yourrepo.git
```

### SSH access (port 222)

Gitea's SSH runs on port **222** (mapped from container port 22 in the compose file). Use it to push without a password once your SSH key is added:

1. **Add your SSH public key** to Gitea: **Settings → SSH/GPG Keys → Add Key** (paste the contents of `~/.ssh/id_ed25519.pub` or `~/.ssh/id_rsa.pub`)

2. **Add the remote**:
   ```bash
   git remote add gitea ssh://git@100.x.x.x:222/youruser/yourrepo.git
   ```

3. **Push to both Gitea and GitHub** — create a global alias:
   ```bash
   git config --global alias.pushall '!git remote | xargs -L1 git push'
   ```

   Now you have three options:
   | Command | What it does |
   |---|---|
   | `git push origin` | Push only to GitHub |
   | `git push gitea` | Push only to Gitea |
   | `git pushall` | Push to all remotes |

> **If SSH asks for a password**: the key wasn't added to Gitea's `authorized_keys` file. On the server, run:
> ```bash
> docker exec gitea su git -c '/usr/local/bin/gitea admin regenerate keys'
> ```

---

## Step 9: Set up Pi-hole (ad-blocker) with Docker

Pi-hole blocks ads at the DNS level — every device on your network gets ad-blocking automatically.

```bash
nano /srv/docker/pihole/docker-compose.yml
```

Paste this:

```yaml
services:
  pihole:
    image: pihole/pihole:latest
    container_name: pihole
    restart: always
    network_mode: host
    environment:
      TZ: "America/Denver"
      WEBPASSWORD: "your-admin-password"
      PIHOLE_DNS_: "1.1.1.1;8.8.8.8"
      DNSMASQ_LISTENING: "all"
    volumes:
      - /srv/docker/pihole/etc-pihole:/etc/pihole
      - /srv/docker/pihole/etc-dnsmasq.d:/etc/dnsmasq.d
    cap_add:
      - NET_ADMIN
```

> **Change these values**:
> - `TZ`: your timezone (find yours at https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)
> - `WEBPASSWORD`: a password you'll remember for the admin UI

> **Why `network_mode: host` instead of `ports:`?**
> Docker's default port mapping (`ports:`) has issues with UDP port 53 on some networks — local subnet queries can time out due to Docker's iptables filtering. Using `network_mode: host` makes Pi-hole bind directly to the host's network interfaces, bypassing Docker's filtering entirely. This ensures DNS works for both local subnet and Tailscale connections.
>
> **Note**: With host networking, Pi-hole's web UI is on port **80** (not 8080). The admin URL becomes `http://n9-ip/admin`.

Save and exit.

Start Pi-hole:

```bash
docker compose -f /srv/docker/pihole/docker-compose.yml up -d
```

Wait about 30 seconds for Pi-hole to initialize.

### Critical: Fix listening mode for Tailscale

Even with `DNSMASQ_LISTENING: "all"` in the environment, Pi-hole's internal config might still restrict DNS to "local" subnets only — which excludes Tailscale's `/32` addresses. After the first boot, fix this:

```bash
# Check the current listening mode
docker exec pihole grep "^  listeningMode" /etc/pihole/pihole.toml

# If it says "LOCAL", change it to "ALL"
docker exec pihole sed -i 's/listeningMode = "LOCAL"/listeningMode = "ALL"/' /etc/pihole/pihole.toml

# Restart Pi-hole to apply
docker restart pihole
```

Wait 15 seconds, then verify it removed `local-service`:

```bash
docker exec pihole grep "local-service" /etc/pihole/dnsmasq.conf
```

If that returns nothing, you're good. If it still shows `local-service`, the sed command didn't match — try this instead:

```bash
docker exec pihole sed -i 's/listeningMode = "LOCAL"/listeningMode = "ALL"/' /etc/pihole/pihole.toml
docker restart pihole
```

### Access the admin UI

Open your browser and go to:

```
http://100.x.x.x/admin
```

(Replace with your N9's Tailscale IP — no port number needed, it's on standard port 80.)

Log in with the password you set.

> **Note**: If you reinstalled Ubuntu/Samba on the N9, you'll need to reset the Pi-hole password:
> ```bash
> docker exec -it pihole pihole setpassword
> ```

### Verify ad blocking is working

From your main computer, test:

```bash
getent ahostsv4 doubleclick.net
```

Should return `0.0.0.0` if Pi-hole is blocking. If it returns a real IP, Pi-hole isn't being used as your DNS — see Step 10 and Step 12 for DNS configuration.

> **If Pi-hole won't start**: run `sudo ss -lntup | grep ':53'` to check port 53. `pihole-FTL` should own the TCP and UDP listeners. If `systemd-resolved` appears, confirm `/etc/systemd/resolved.conf.d/pihole.conf` contains `DNSStubListener=no`, restart systemd-resolved, then restart the Pi-hole Compose service. Do not stop systemd-resolved or delete Pi-hole volumes.

---

## Step 10: Configure the Netgear Nighthawk X4S

This is the key step that makes Pi-hole work for your whole network automatically.

### 10.1 Connect everything

```
XB7 LAN port --(Ethernet)--> Netgear WAN port
N9 --(Ethernet)--> Netgear LAN port (any of the 4)
```

> The XB7 stays in router mode — do NOT enable bridge mode. Xfinity Home cameras need the XB7's router features.

### 10.2 Log into the Netgear admin

On a computer connected to the Netgear's WiFi (default SSID/password on the router sticker), open a browser and go to:

```
http://192.168.1.1
```

Username: `admin`
Password: `password` (or the password printed on the router sticker)

> The first time you log in, it may prompt you to change the password. Do that.

### 10.3 Reserve a static IP for the N9

1. Find your N9's MAC address. On the N9 (via SSH), run:
   ```bash
   ip a
   ```
   Look for `link/ether` under your network interface (`eth0` or `enpXsY`). It looks like `a1:b2:c3:d4:e5:f6`. Write it down.

2. In Netgear admin, go to:
   **Advanced > Setup > LAN Setup > Address Reservation**

3. Click **Add**.

4. Enter:
   - MAC Address: the one you found above
   - IP Address: `192.168.1.100`
   - Name: `n9-server`

5. Click **Add** and **Apply**.

> This ensures the N9 always gets the same IP. Without this, the IP could change after a reboot and break Pi-hole DNS.

### 10.4 Set DNS to Pi-hole

1. In Netgear admin, go to:
   **Advanced > Setup > Internet Setup**

2. Scroll to **Domain Name Server (DNS) Address**.

3. Select **"Use These DNS Servers"**.

4. Enter:
   - Primary DNS: `192.168.1.100` (your N9's reserved IP)
   - Secondary DNS: `1.1.1.1` (fallback in case the N9 is down)

5. Click **Apply**.

> Now every device that connects to the Netgear (via WiFi or Ethernet) will automatically use Pi-hole for DNS. No manual setup needed per device.

### 10.5 (Optional) Disable XB7 WiFi

To avoid interference and confusion, disable the XB7's WiFi so devices only connect to the Netgear:

1. Log into XB7 admin: `http://10.0.0.1` or use the Xfinity app
2. Go to **Gateway > Connection > WiFi**
3. Disable both 2.4GHz and 5GHz radios

> Leave the XB7's Xfinity Home radio enabled — that's separate from WiFi.

### 10.6 (Advanced) Configure DNS on clients that bypass router DHCP

Some Linux setups (e.g., Arch Linux with systemd-networkd + iwd) ignore DHCP-provided DNS servers. You can set Pi-hole as the DNS resolver directly on those machines.

**systemd-networkd** (`/etc/systemd/network/20-wlan.network` or similar):

```
[Match]
Name=wlan0

[Network]
DHCP=yes
DNS=192.168.1.100

[DHCPv4]
UseDNS=false
```

> `UseDNS=false` tells systemd-networkd to ignore DNS servers from DHCP. The `DNS=192.168.1.100` line sets Pi-hole as the only DNS resolver. This is useful for machines that need Pi-hole even when DHCP-provided DNS doesn't point to it (e.g., static network profiles, VPNs).

Restart systemd-networkd after changing the config:
```bash
sudo systemctl restart systemd-networkd
```

Verify:
```bash
resolvectl status
```

Look for `DNS Servers: 192.168.1.100` under your interface.

---

## Step 11: Test that everything works

### 11.1 Reboot the N9

```bash
sudo reboot
```

Wait a minute, then SSH back in:

```bash
ssh yourusername@192.168.1.100
```

(Or use Tailscale: `ssh yourusername@100.x.x.x`)

### 11.2 Check all services

```bash
# Tailscale should be running
tailscale ip -4

# Samba should be active
sudo systemctl status smbd --no-pager

# Docker containers should be up
docker ps
```

All three should be running.

### 11.3 Test each service

| Test | How |
|---|---|
| **File share** | On your main computer's file manager, go to `smb://192.168.1.100/files`. Enter your username and Samba password. Create a file. Check on the N9: `ls /srv/files/` |
| **Git repos** | Visit `http://100.x.x.x:3000` in your browser. Create a repo. Clone it on your main computer. Make a commit. Push. Test SSH too: `git remote add test ssh://git@100.x.x.x:222/youruser/yourrepo.git && git push test main` |
| **Ad blocking** | Visit `http://100.x.x.x/admin`. Check the dashboard — you should see queries coming in. Run `dig doubleclick.net` from your main computer — it should resolve to `0.0.0.0`. |
| **Remote access** | Disconnect from home WiFi. Connect via phone hotspot or another network. Make sure Tailscale is running on your device. Mount the Samba share using the Tailscale IP: `smb://100.x.x.x/files`. Visit Gitea at `http://100.x.x.x:3000`. Everything works. |

---

## Step 12: Mount the Samba share on your main Linux computer

### Option A: File manager (easy, manual mount)

Open your file manager (Nautilus/Files on GNOME, Dolphin on KDE). Go to "Other Locations" or the address bar and enter:

```
smb://192.168.1.100/files
```

Enter your username and Samba password.

For remote access (different network), use the Tailscale IP instead:

```
smb://100.x.x.x/files
```

### Option B: Auto-mount on boot (convenient, permanent)

Install the CIFS utilities:

**Ubuntu/Debian:**
```bash
sudo apt install cifs-utils -y
```

**Arch Linux:**
```bash
sudo pacman -S cifs-utils
```

Create the mount point:
```bash
sudo mkdir /mnt/n9
```

Create a credentials file (so you don't put your password in `/etc/fstab`):

```bash
nano ~/.smb-credentials
```

Put this inside:

```
username=yourusername
password=your-samba-password
domain=
```

Save and exit.

Lock it down so only you can read it:

```bash
chmod 600 ~/.smb-credentials
```

Add to fstab for auto-mounting:

```bash
sudo nano /etc/fstab
```

Add this line at the very end:

```
//192.168.1.100/files /mnt/n9 cifs credentials=/home/yourmainusername/.smb-credentials,uid=1000,gid=1000,noexec,nosuid,nodev 0 0
```

> Replace:
> - `192.168.1.100` with the N9's local IP (or Tailscale IP for remote)
> - `yourmainusername` with **your main computer's** username
> - `uid=1000` and `gid=1000` with your user's actual IDs (run `id` to check)

Save and exit.

Mount it now:

```bash
sudo mount /mnt/n9
```

Your N9 files are now at `/mnt/n9` on your main computer, and will auto-mount on every reboot.

---

## Step 13: Configure Tailscale DNS (so Pi-hole works remotely too)

This makes Pi-hole follow you even when you're not on your home network.

1. Go to https://login.tailscale.com/admin/dns
2. Under **Nameservers**, click **Add nameserver**
3. Select **Custom** and enter your N9's **Tailscale IP** (`100.77.72.6` for this setup)
4. **Do NOT add any other nameservers** (no public DNS like `1.1.1.1`) — Pi-hole handles upstream forwarding internally
5. **Enable "Override local DNS"** — this toggle forces all Tailscale devices to use the custom nameserver, even on other networks (cellular, hotel WiFi, etc.). Without this, remote devices use their local network's DNS and Pi-hole is ignored.
6. Save

> **Why no fallback?** If both Pi-hole and a public DNS (e.g., `1.1.1.1`) are configured as global nameservers, Tailscale queries both in parallel and uses the fastest response. The public DNS will always respond faster from a remote device than Pi-hole over a Tailscale tunnel, so ad-laden queries would bypass Pi-hole entirely. Instead, let Pi-hole itself handle forwarding to upstream DNS (Step 9 has `PIHOLE_DNS_: "1.1.1.1;8.8.8.8"`).

Now any device connected to your Tailnet (even remotely) will use Pi-hole for DNS. Ad-blocking follows you everywhere.

> **Note on the N9 host's own DNS**: The N9 itself resolves DNS via systemd-resolved → `127.0.0.1` → Pi-hole (configured in Step 4). This is separate from the Tailscale DNS setting. Other Tailscale devices (desktops, laptops, phones) use the custom nameserver set here (`100.77.72.6`). Both paths go through the same Pi-hole — ad-blocking works everywhere.

> **If you previously used the old approach** (disabling systemd-resolved, hardcoding `1.1.1.1`), and you switch to the new approach (Step 4), you must also run `sudo tailscale set --accept-dns=false` on the N9. This prevents Tailscale from overwriting `/etc/resolv.conf` with its MagicDNS resolver, which would bypass Pi-hole for the host itself.

> **iOS-specific**: After enabling Override, go to the iPhone Tailscale app → **Settings** → **DNS** and make sure **"Use Tailscale DNS"** is ON. Also temporarily disable **iCloud Private Relay** (Settings → your name → iCloud → Private Relay) for testing, as it bypasses Tailscale DNS entirely for Safari traffic.

### 13.1 Add a device to the Tailnet

Install Tailscale on the device and sign in with the same Tailscale account used by the N9. The device must appear in the admin console under **Machines** before it can use Pi-hole.

On Linux:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
sudo tailscale set --accept-dns=true
```

On Windows or macOS:

1. Install the Tailscale app from https://tailscale.com/download
2. Sign in with the same account as the N9.
3. Keep **Use Tailscale DNS** enabled in the app's DNS settings.

On iOS or Android:

1. Install the Tailscale app and sign in with the same account as the N9.
2. Connect the device to Tailscale.
3. Enable **Use Tailscale DNS** in the app settings.

The N9 itself is different from the client devices. Keep Tailscale DNS disabled on N9 so its local resolver continues to use Pi-hole:

```bash
sudo tailscale set --accept-dns=false
```

### 13.2 Verify Pi-hole from a Tailscale device

On a Linux client, confirm the device is connected and query a blocked domain:

```bash
tailscale status
resolvectl query doubleclick.net
```

The result should be `0.0.0.0` or another Pi-hole blocked response. If `resolvectl` is unavailable, use `nslookup doubleclick.net` or `dig doubleclick.net`.

If the query is not blocked, check these in order:

1. The device is signed in to the same Tailnet and is online.
2. **Override local DNS** is enabled in the Tailscale admin console.
3. **Use Tailscale DNS** is enabled on the device.
4. `100.77.72.6` is the only global nameserver; do not add a public fallback.
5. Pi-hole is healthy: `docker inspect --format '{{.State.Health.Status}}' pihole`.

---

## Step 14: Convenience shortcuts

### 14.1 Add the N9 to your hosts file

Add the N9's Tailscale IP to `/etc/hosts` on your main computer so you can use `n9` instead of typing the IP every time:

```bash
echo '100.x.x.x      n9' | sudo tee -a /etc/hosts
```

Replace `100.x.x.x` with your N9's actual Tailscale IP. Now you can use `n9` anywhere you'd normally use the IP:

| Before | After |
|--------|-------|
| `ssh tlaloch@100.x.x.x` | `ssh tlaloch@n9` |
| `http://100.x.x.x:3000` | `http://n9:3000` |
| `smb://100.x.x.x/files` | `smb://n9/files` |
| `http://100.x.x.x/admin` | `http://n9/admin` |

### 14.2 Add SSH config shortcut + Tramp keepalive

For even shorter SSH commands, add this to `~/.ssh/config`:

```
Host n9
    HostName 100.x.x.x
    User yourusername
```

Recommended additions for Emacs Tramp performance over Tailscale:

```
Host 100.*.*.* *.tailscale.com
    ServerAliveInterval 60
    ServerAliveCountMax 3
    ControlMaster auto
    ControlPath ~/.ssh/controlmasters/%r@%h:%p
    ControlPersist 10m
```

Create the ControlMaster directory:

```bash
mkdir -p ~/.ssh/controlmasters
```

> **What this does**: `ServerAliveInterval` prevents long-running Tramp sessions from timing out. `ControlMaster` reuses a single SSH connection for multiple sessions — subsequent Tramp connections are instant (no re-authentication). `ControlPersist` keeps the connection alive for 10 minutes after the last session closes.

Now you can just type `ssh n9` — no username, no IP, no `-i` key flag needed. SCP and rsync work too (`scp file n9:~/`).

---

## Step 15: Install Syncthing on the N9 (optional)

Syncthing keeps files in sync between your devices with local copies — no network mounts, no latency for LSP, full offline editing. Install it if you want code and org files available locally on both home and workstation without Samba mounts.

### 15.1 Install Syncthing

```bash
sudo apt install syncthing
systemctl --user enable --now syncthing
sudo loginctl enable-linger tlaloch
```

> **`enable-linger` is critical** — it keeps Syncthing running after you log out. Without this, Syncthing stops when the SSH session ends.

### 15.2 Configure Syncthing Web UI

Syncthing's admin interface is at `http://127.0.0.1:8384`. Since the N9 has no browser, use an SSH tunnel from your desktop:

```bash
ssh -L 8385:127.0.0.1:8384 n9
```

Then open `http://127.0.0.1:8385` in your desktop browser.

1. **Actions → Settings** → set device name to `n9-server`
2. **Show ID** — copy this device ID (needed when adding homebase from other devices)

### 15.3 Firewall note

If you have UFW enabled, allow Syncthing's ports:

```bash
sudo ufw allow 22000/tcp
sudo ufw allow 21027/udp
```

---

## Step 16: Syncthing device topology

The three machines form a mesh:

```
n9-server (100.77.72.6) ── Syncthing: org files ──┐
                                                     ├── home (100.75.18.27)
homebase also has: workspace snapshot (rsync) ─────────┤
                                                     └── workstation (100.69.124.19)
```

### Folder-to-device mapping

| Folder | homebase | home | workstation |
|--------|----------|------|-------------|
| **workspace** (`~/workspace/github.com/ManoloEsS/`) | rsync snapshot only | source + sync | source + sync |
| **org** (varies per machine) | source of truth | local copy | local copy |

> homebase does **not** run a Syncthing folder for workspace — its backup is the rsync snapshot at `/srv/files/workspace`. New machines seed from it (see **Step 21.4**).

### Device IDs and addresses

Use the **remote device's** Tailscale IP when adding a device:

| From Web UI | Adding | Address |
|-------------|--------|---------|
| homebase | home | `tcp://100.75.18.27:22000, dynamic` |
| homebase | workstation | `tcp://100.69.124.19:22000, dynamic` |
| home | homebase | `tcp://100.77.72.6:22000, dynamic` |
| home | workstation | `tcp://100.69.124.19:22000, dynamic` |
| workstation | homebase | `tcp://100.77.72.6:22000, dynamic` |
| workstation | home | `tcp://100.75.18.27:22000, dynamic` |

> The `, dynamic` fallback lets Syncthing use global discovery if the Tailscale tunnel isn't available. For maximum security (traffic stays inside Tailscale), omit `dynamic`.

### Sharing a folder

1. On the source machine's Web UI, click **Add Folder**
2. Set **Folder Label** and **Folder Path**
3. **Sharing** tab → check the target devices
4. **File Versioning** → `Trash Can` (30 days recommended) — protects against accidental changes
5. **Ignore Patterns** tab → add patterns (see below)
6. On the target machine's Web UI, **Accept** the incoming folder and set the local path

---

## Step 17: Syncthing ignore patterns

These prevent build artifacts, secrets, and OS junk from being synced. The patterns below are designed for Go/JS/Python/Lua projects.

Paste into **Ignore Patterns** tab for the `workspace` folder:

```
# Build artifacts
node_modules/
dist/
build/
.next/
__pycache__/
*.pyc
*.pyo
vendor/
target/

# IDE / editor
.opencode/
.idea/
.vscode/
*.swp
*.swo
*~

# Environment / secrets
.env
.env.local
*.env
*.pem
*.key

# OS files
.DS_Store
Thumbs.db

# Git (already cloned locally on each machine)
.git/

# Binary / large artifacts
*.bin
*.exe
*.dll
*.so
*.dylib
*.iso
```

For the `org` folder, ignore patterns can be minimal — just OS files and backups:

```
*~
*.swp
.DS_Store
```

---

## Step 18: Emacs/Doom configuration

This documents the Doom Emacs configuration shared between home and workstation. Both machines have identical configs in `~/.config/doom/`.

### 18.1 init.el — modules

The following language modules are enabled with LSP:

```elisp
(go +lsp)          ; Go — uses gopls
(javascript +lsp)  ; JS/TS — uses typescript-language-server
(python +lsp)      ; Python — uses pyright
(lua +lsp)         ; Lua — uses lua-language-server
```

Key non-language modules:

```elisp
(lsp +eglot)       ; Use Eglot as the LSP client
(format +onsave)   ; Auto-format on save via apheleia
tramp              ; Remote file editing over SSH
tree-sitter        ; Syntax highlighting engine
(corfu +orderless) ; Completion popup
vertico            ; Search/picker framework
```

### 18.2 config.el — LSP servers and formatters

**Go formatting** — uses `goimports` (formats + organizes imports) instead of gopls formatting:

```elisp
(add-hook 'go-mode-hook (lambda () (setq-local +format-with 'goimports)))
(add-hook 'go-ts-mode-hook (lambda () (setq-local +format-with 'goimports)))
```

**Python formatting** — uses `ruff` (fast Rust-based formatter):

```elisp
(after! apheleia
  (setf (alist-get 'python-mode apheleia-mode-alist) 'ruff)
  (setf (alist-get 'python-ts-mode apheleia-mode-alist) 'ruff))
```

**Org directory** — now points to the local Syncthing-synced copy:

```elisp
(setq org-directory "~/org")
```

### 18.3 LSP servers to install

On **home** and **workstation**, install the language servers:

```bash
# TypeScript/JavaScript
npm install -g typescript typescript-language-server

# Python
npm install -g pyright

# Lua
curl -sL "https://github.com/LuaLS/lua-language-server/releases/download/3.18.2/lua-language-server-3.18.2-linux-x64.tar.gz" | tar xz
cp lua-language-server-3.18.2-linux-x64/bin/lua-language-server ~/.local/bin/
rm -rf lua-language-server-3.18.2-linux-x64

# Formatters
mise use -g ruff@latest       # Python formatter
sudo pacman -S shfmt shellcheck  # Optional: shell formatting
```

> LSP servers must be on `$PATH` for Eglot to find them. Check with: `which typescript-language-server`, `which pyright`, `which lua-language-server`.

### 18.4 After modifying init.el

Run on both machines:

```bash
doom sync
```

Then restart Emacs.

---

## Step 19: Syncthing first-time setup checklist

| Machine | What to do |
|---------|------------|
| **homebase** | Install Syncthing (Step 15), configure via SSH tunnel, share `org` folder |
| **home** | Install Syncthing (`sudo pacman -S syncthing`), `systemctl --user enable --now syncthing`, accept `org` folder, set path to `~/org` |
| **workstation** | Same as home, accept both `workspace` and `org` folders |
| **all** | Update Doom `org-directory` to `~/org`, install LSP servers |

### Typical gotchas

- **`rem_workspace` or similar wrong-named folder appears**: Edit the folder in Syncthing Web UI → **Folder Path** → change to the correct existing path
- **"Temporary failure resolving" on homebase**: Verify `/etc/resolv.conf` points to `/run/systemd/resolve/resolv.conf`, `resolvectl status` shows `resolv.conf mode: uplink`, and `enp2s0` uses `127.0.0.1` as its DNS server. Restore with `sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf && sudo resolvectl dns enp2s0 127.0.0.1 && sudo resolvectl domain enp2s0 "~." && sudo systemctl restart systemd-resolved`
- **Pi-hole stops blocking**: Check `docker exec pihole grep listeningMode /etc/pihole/pihole.toml` — must be `"ALL"`, not `"LOCAL"`
- **Syncthing stops after SSH logout**: You missed `sudo loginctl enable-linger tlaloch` — run it now

---

## Step 20: DNS update mode script

The `dns-update-mode` script helps manage homebase's DNS when you need to run system updates without going through Pi-hole. It also includes diagnostics for fixing Pi-hole remote access.

### 20.1 Installation

The script lives at `~/.local/bin/dns-update-mode` on homebase.

After installing, allow passwordless `sudo` for `resolvectl` (needed for toggling DNS):

```bash
echo "%sudo ALL=(ALL) NOPASSWD: /usr/bin/resolvectl" | sudo tee /etc/sudoers.d/resolvectl
sudo chmod 440 /etc/sudoers.d/resolvectl
```

### 20.2 Usage

```bash
# Run a command with public DNS (e.g. apt update), auto-restores Pi-hole after
dns-update-mode update "sudo apt update && sudo apt upgrade -y"

# Manually switch to public DNS
dns-update-mode on

# Manually restore Pi-hole DNS
dns-update-mode off

# Check current DNS config
dns-update-mode status

# Diagnose Pi-hole remote access
dns-update-mode diagnose

# Fix Pi-hole listeningMode (required for remote Tailscale DNS queries)
dns-update-mode fix-pihole
```

### 20.3 What the diagnose command checks

| Check | What it looks for |
|-------|-------------------|
| Listening mode | `ALL` (accepts remote queries) vs `LOCAL` (local subnet only) |
| Port 53 | Pi-hole should be the sole listener |
| DNS resolution | `google.com` resolves; `doubleclick.net` returns `0.0.0.0` (blocked) |
| Tailscale DNS | Whether `--accept-dns` is enabled |

### 20.4 When remote Pi-hole stops working

Run on homebase:

```bash
dns-update-mode diagnose
```

If it shows `listeningMode = LOCAL`, run:

```bash
dns-update-mode fix-pihole
```

This is the most common reason Pi-hole stops working from remote Tailscale devices — the setting occasionally resets after Pi-hole container updates or restarts.

---

## Step 21: Set up a new client machine (fresh Omarchy install)

Use this to bring a brand-new Omarchy install to the same state as this desktop — SSH to N9, Syncthing, dotfiles, and Neovim. Everything the new machine needs lives in GitHub/Gitea or in the N9 workspace snapshot at `/srv/files/workspace`.

### 21.1 Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Log in with your Tailscale account, then verify the N9 is visible:

```bash
tailscale status
```

N9 should appear as `homebase` with IP `100.77.72.6`.

### 21.2 Hosts file and SSH config

```bash
echo '100.77.72.6      n9' | sudo tee -a /etc/hosts
mkdir -p ~/.ssh/controlmasters
```

Copy the SSH config block from **Step 14.2** into `~/.ssh/config`, then test:

```bash
ssh n9 echo connected
```

### 21.3 SSH keys (GitHub and Gitea)

```bash
ssh-keygen -t ed25519 -C "you@example.com"
```

- **GitHub**: add the pubkey (`~/.ssh/id_ed25519.pub`) at `https://github.com/settings/keys`.
- **Gitea**: add the same pubkey at `http://n9:3000/user/settings/keys`.

Test both:

```bash
ssh -T git@github.com
ssh -p 222 git@n9
```

### 21.4 Syncthing

```bash
sudo pacman -S syncthing        # or your distro's package
systemctl --user enable --now syncthing
```

- Open the web UI: `http://127.0.0.1:8384`.
- **Add Remote Device** — the N9 (`homebase`), device ID `5RQ672R-7VYA66O-N77ZAUS-SUFIQ65-BUF7YLV-BQWTWT5-FMHIO5G-5WZCJQV`, address `tcp://100.77.72.6:22000, dynamic`.
- On the N9, accept the new device in its Syncthing GUI — tunnel in with `ssh -L 8385:127.0.0.1:8384 n9` then open `http://127.0.0.1:8385`.

#### org — sync from homebase

homebase hosts the `org` folder as its source of truth. Add it on this machine:

- **Add Folder** → folder ID `2l9rt-tecrk`, path `~/org` → **Sharing** tab → check `homebase`.

#### workspace — seed from the N9 snapshot, then sync

homebase does **not** host the workspace in Syncthing — its backup is the rsync snapshot at `/srv/files/workspace`. Seed it once from there, then register it as a normal Syncthing folder:

```bash
rsync -aP --delete n9:/srv/files/workspace/ ~/workspace/
```

- **Add Folder** → folder ID `gyra3-p2nen`, path `~/workspace/github.com/ManoloEsS` → **Sharing** tab → check the other devices that should keep it in sync.

> **Replacing the old machine**: the new machine gets a brand-new Syncthing device ID. Once it's online, remove the retired device from each folder's **Sharing** tab on homebase and workstation (and from the device list) so it stops syncing.

### 21.5 Dotfiles and Neovim

Clone the dotfiles repo, then install everything it manages:

```bash
git clone git@github.com:ManoloEsS/dotfiles_omarchy_desktop.git ~/dotfiles_omarchy_desktop
cd ~/dotfiles_omarchy_desktop
```

Install base tools, Oh My Zsh, and set zsh as the default shell (needs an AUR helper such as `yay`):

```bash
./omarchy_scripts/install-dev-tools.sh
```

Install the Oh My Zsh plugins and powerlevel10k theme:

```bash
./omarchy_scripts/install-omz-plugins.sh
```

Symlink the managed configs with GNU stow (hypr, ncspot, tmux, wezterm, yazi, zsh, p10k):

```bash
sudo pacman -S stow
./omarchy_scripts/stow-configs.sh
```

Wire up the configs the scripts don't cover:

```bash
# mise
mkdir -p ~/.config/mise
ln -sf ~/dotfiles_omarchy_desktop/mise/.config/mise/config.toml ~/.config/mise/config.toml

# ghostty
mkdir -p ~/.config/ghostty
cp ~/dotfiles_omarchy_desktop/ghostty/.config/config ~/.config/ghostty/config

# opencode
mkdir -p ~/.config/opencode
cp ~/dotfiles_omarchy_desktop/opencode/.config/opencode.json ~/.config/opencode/opencode.json
cp -r ~/dotfiles_omarchy_desktop/opencode/.config/skills ~/.config/opencode/skills

# claude skills
mkdir -p ~/.claude/skills
ln -sf ~/dotfiles_omarchy_desktop/.claude/skills/fullstack-static-scaffold ~/.claude/skills/fullstack-static-scaffold
```

Install tmux plugins (TPM) by starting tmux and pressing `prefix + I`.

Clone Neovim (the kickstart fork):

```bash
git clone git@github.com:ManoloEsS/kickstart.nvim.git ~/.config/nvim
```

### 21.6 Samba mount (optional)

Follow **Step 12 (Option B)** to auto-mount `/mnt/n9` so the N9 files live at `/mnt/n9`.

---

## Conflict-free workflow (Syncthing)

Syncthing replaces the old Samba-centric workflow. Each machine has its own local copies of workspace and org files, updated automatically in the background.

```
homebase (server)
  +-- Samba share (iPhone access, media, random file sharing)
  +-- Syncthing: org files (source of truth)
  +-- Workspace snapshot (rsync at /srv/files/workspace — not Syncthing)

home (Arch desktop)
  +-- Syncthing: workspace local copy — full LSP, instant builds
  +-- Syncthing: org local copy — fast org-mode
  +-- Git push/pull to Gitea/GitHub (intentional commit workflow)

workstation (Arch desktop)
  +-- Syncthing: workspace local copy — same as home
  +-- Syncthing: org local copy — same as home
  +-- Git push/pull to Gitea/GitHub (same repos)
```

- **Code**: work locally, commit + push. Syncthing is NOT a substitute for Git — use Git for version control.
- **Files (org, documents)**: Syncthing handles bidirectional sync automatically. Versioning (Trash Can) protects against accidents.
- **Samba still exists** for non-Syncthing devices — iPhone, media center, sending a single file to a friend.
- **Avoid editing the same file on two machines simultaneously** — you'll get a `.syncthing.ConflictingCopy` file. Pick one, delete the other.

---

## Maintenance

| Task | Command |
|---|---|
| Update system packages | `sudo apt update && sudo apt upgrade -y` |
| Update Gitea image | `docker compose -f /srv/docker/gitea/docker-compose.yml pull && docker compose -f /srv/docker/gitea/docker-compose.yml up -d` |
| Update Pi-hole image | `docker compose -f /srv/docker/pihole/docker-compose.yml pull && docker compose -f /srv/docker/pihole/docker-compose.yml up -d` |
| Back up data | `rsync -av /srv/ /path/to/external/drive/` |
| Check disk space | `df -h` |
| See running containers | `docker ps` |
| Check Tailscale status | `tailscale status` |
| Reboot the N9 | `sudo reboot` |
| View Pi-hole logs | `docker logs pihole` |
| View Gitea logs | `docker logs gitea` |
| Check Syncthing status | `systemctl --user status syncthing` |
| Syncthing Web UI tunnel | `ssh -L 8385:127.0.0.1:8384 n9` then `http://127.0.0.1:8385` |
| View DNS resolver config | `resolvectl status` |
| Test Pi-hole DNS | `resolvectl query doubleclick.net` (should return `0.0.0.0`) |
| Diagnose Pi-hole remote access | `dns-update-mode diagnose` |
| Fix Pi-hole listening mode | `dns-update-mode fix-pihole` |
| System update (bypass Pi-hole) | `dns-update-mode update "sudo apt update && sudo apt upgrade -y"` |
| Allow remote DNS through UFW | `sudo ufw allow in on tailscale0 to any port 53 proto udp && sudo ufw allow in on tailscale0 to any port 53 proto tcp` |
| Check Tailscale DNS config | `https://login.tailscale.com/admin/dns` |

---

## Data layout on the N9

```
~/.config/syncthing/        Syncthing config (created on first run)
/srv/
 |-- files/                  Your Samba file share
 |-- docker/
 |    |-- gitea/
 |    |    |-- docker-compose.yml
 |    |    +-- data/         Gitea databases and repos
 |    +-- pihole/
 |         |-- docker-compose.yml
 |         |-- etc-pihole/   Pi-hole config
 |         +-- etc-dnsmasq.d/
```

---

## Common problems and fixes

| Problem | Fix |
|---|---|
| **Pi-hole won't start** | Port 53 conflict. Check `sudo ss -lntup | grep ':53'`. `pihole-FTL` should be the only DNS listener; verify `DNSStubListener=no` in Step 4 and restart systemd-resolved before restarting Pi-hole. |
| **Pi-hole container is unhealthy** | Check `docker inspect --format '{{range .State.Health.Log}}{{.Output}}{{end}}' pihole` and `docker logs --tail=200 pihole`. A timeout to `127.0.0.1:53` usually means systemd-resolved still owns the stub listener; apply Step 4 and restart the Compose service. |
| **Can't mount Samba share remotely** | Make sure both devices have Tailscale running. Use the `100.x.x.x` IP, not `192.168.1.100`. |
| **Gitea page won't load** | Wait 30 seconds after starting. Check with `docker ps` that the container is healthy. |
| **Forgot Samba password** | `sudo smbpasswd -a yourusername` to reset it. |
| **Tailscale not connecting** | Run `tailscale status`. Check https://login.tailscale.com/admin/machines — both devices should show as connected. |
| **Can't SSH after reboot** | Check the N9 is powered on. The IP might have changed if DHCP reservation failed — check the Netgear admin for the N9's current IP. |
| **Ad-blocking not working on a device** | Make sure that device is connected to the Netgear (not the XB7's WiFi). Check the device's DNS — it should be `192.168.1.100`. |
| **Pi-hole not blocking ads on remote Tailscale devices (iPhone, laptop away from home)** | Three things to check: (1) Tailscale admin console → **"Override local DNS"** must be ON, (2) devices must have **"Use Tailscale DNS"** enabled in their Tailscale app, (3) **do not** add a public DNS fallback (like `1.1.1.1`) as a second nameserver — it races with Pi-hole and lets ads through. See Step 13. |
| **Pi-hole logs show "ignoring query from non-local network"** | Pi-hole's `listeningMode` is still set to `LOCAL`. Change it to `ALL` (see Step 9: "Critical: Fix listening mode for Tailscale"). Restart Pi-hole after. |
| **DNS queries from Tailscale not resolving** | Same fix as above — `listeningMode` must be `ALL`. Also verify that Tailscale DNS is configured in the Tailscale admin console (Step 13). Run `dns-update-mode diagnose` on homebase to check. |
| **DNSMASQ_LISTENING env var doesn't take effect** | Pi-hole v6 doesn't always honor `DNSMASQ_LISTENING: "all"`. Edit `pihole.toml` directly inside the container: `dns-update-mode fix-pihole` (does this automatically) |
| **`dns-update-mode` complains about permissions** | Missing sudoers rule. Run `echo "%sudo ALL=(ALL) NOPASSWD: /usr/bin/resolvectl" | sudo tee /etc/sudoers.d/resolvectl && sudo chmod 440 /etc/sudoers.d/resolvectl` on homebase (one-time setup). |
| **apt update fails with "Temporary failure resolving"** | DNS is misconfigured. Run `dns-update-mode update "sudo apt update"` to temporarily use public DNS, or run `dns-update-mode diagnose` to troubleshoot. |
| **N9 IP changed after reboot** | DHCP reservation on the Netgear didn't take. Re-check Step 10.3. |
| **Xfinity cameras stopped working** | You accidentally enabled bridge mode on the XB7. Disable bridge mode — XB7 must stay in router mode. |

---

## Quick reference

| What | Address |
|---|---|---|
| N9 local IP | `192.168.1.100` |
| N9 Tailscale IP | `100.x.x.x` (from `tailscale ip -4`) |
| Samba share (local) | `smb://192.168.1.100/files` |
| Samba share (remote) | `smb://100.x.x.x/files` |
| Gitea (HTTP) | `http://100.x.x.x:3000` |
| Gitea (SSH) | `ssh://git@100.x.x.x:222` |
| Pi-hole admin | `http://100.x.x.x/admin` |
| SSH (local, host) | `ssh yourusername@192.168.1.100` |
| SSH (remote, host) | `ssh yourusername@100.x.x.x` |
| Netgear admin | `http://192.168.1.1` |
| XB7 admin | `http://10.0.0.1` |
| Tailscale admin | `https://login.tailscale.com/admin` |
| Tailscale DNS config | `https://login.tailscale.com/admin/dns` |
| Syncthing Web UI (local) | `http://127.0.0.1:8384` |
| Syncthing Web UI (tunnel) | `ssh -L 8385:127.0.0.1:8384 n9` → `http://127.0.0.1:8385` |
| Syncthing default port | `22000/tcp` (peer sync), `21027/udp` (discovery) |
