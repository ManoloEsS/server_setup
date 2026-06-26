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
sudo ufw allow 53           # Pi-hole DNS
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

---

## Step 4: Disable systemd-resolved (needed for Pi-hole)

Ubuntu uses a built-in DNS resolver that occupies port 53. Pi-hole needs this port.

```bash
sudo systemctl disable systemd-resolved
sudo systemctl stop systemd-resolved
```

Replace the DNS config so the N9 can still resolve domains:

```bash
sudo nano /etc/resolv.conf
```

Delete everything in the file and replace with:

```
nameserver 1.1.1.1
nameserver 8.8.8.8
```

Save (`Ctrl+O`, then `Enter`) and exit (`Ctrl+X`).

> **Nano basics**: Ctrl+O saves, Ctrl+X exits. Use arrow keys to navigate. It's a simple text editor — don't overthink it.

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

> **If Pi-hole won't start**: you likely forgot to disable `systemd-resolved` in Step 4. Go back and check. The error will mention port 53 being in use.

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
| **Git repos** | Visit `http://100.x.x.x:3000` in your browser. Create a repo. Clone it on your main computer. Make a commit. Push. |
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
3. Select **Custom** and enter your N9's **Tailscale IP** (e.g., `100.x.x.x`)
4. Save

Now any device connected to your Tailnet (even remotely) will use Pi-hole for DNS. Ad-blocking follows you everywhere.

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

### 14.2 Add an SSH config shortcut

For even shorter SSH commands, add this to `~/.ssh/config`:

```
Host n9
    HostName 100.x.x.x
    User yourusername
```

Then you can just type `ssh n9` — no username, no IP, no `-i` key flag needed. SCP and rsync work too (`scp file n9:~/`).

---

## Conflict-free workflow

To avoid merge conflicts when working on multiple machines:

```
Desktop (main workstation)
  +-- Mounts Samba drive from N9 (main file access)
  +-- Clones/pushes to Gitea (dev code)
  |
  +-- Laptop SSH's into Desktop (via Tailscale)
      (Laptop never mounts Samba directly)
```

- **Code**: clone repos from Gitea, work locally, commit and push. Intentional commits handle conflicts.
- **General files**: mount the Samba drive on your desktop only. Access from laptop via SSH/RDP/rsync into the desktop.
- **Laptop**: VS Code Remote SSH into the desktop over Tailscale for a full dev environment.

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

---

## Data layout on the N9

```
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
| **Pi-hole won't start** | You forgot to disable `systemd-resolved` (Step 4). Run those commands again. Check with: `sudo lsof -i :53` |
| **Can't mount Samba share remotely** | Make sure both devices have Tailscale running. Use the `100.x.x.x` IP, not `192.168.1.100`. |
| **Gitea page won't load** | Wait 30 seconds after starting. Check with `docker ps` that the container is healthy. |
| **Forgot Samba password** | `sudo smbpasswd -a yourusername` to reset it. |
| **Tailscale not connecting** | Run `tailscale status`. Check https://login.tailscale.com/admin/machines — both devices should show as connected. |
| **Can't SSH after reboot** | Check the N9 is powered on. The IP might have changed if DHCP reservation failed — check the Netgear admin for the N9's current IP. |
| **Ad-blocking not working on a device** | Make sure that device is connected to the Netgear (not the XB7's WiFi). Check the device's DNS — it should be `192.168.1.100`. |
| **Pi-hole logs show "ignoring query from non-local network"** | Pi-hole's `listeningMode` is still set to `LOCAL`. Change it to `ALL` (see Step 9: "Critical: Fix listening mode for Tailscale"). Restart Pi-hole after. |
| **DNS queries from Tailscale not resolving** | Same fix as above — `listeningMode` must be `ALL`. Also verify that Tailscale DNS is configured in the Tailscale admin console (Step 13). |
| **DNSMASQ_LISTENING env var doesn't take effect** | Pi-hole v6 doesn't always honor `DNSMASQ_LISTENING: "all"`. Edit `pihole.toml` directly inside the container: `docker exec pihole sed -i 's/listeningMode = "LOCAL"/listeningMode = "ALL"/' /etc/pihole/pihole.toml && docker restart pihole` |
| **N9 IP changed after reboot** | DHCP reservation on the Netgear didn't take. Re-check Step 10.3. |
| **Xfinity cameras stopped working** | You accidentally enabled bridge mode on the XB7. Disable bridge mode — XB7 must stay in router mode. |

---

## Quick reference

| What | Address |
|---|---|
| N9 local IP | `192.168.1.100` |
| N9 Tailscale IP | `100.x.x.x` (from `tailscale ip -4`) |
| Samba share (local) | `smb://192.168.1.100/files` |
| Samba share (remote) | `smb://100.x.x.x/files` |
| Gitea | `http://100.x.x.x:3000` |
| Pi-hole admin | `http://100.x.x.x/admin` |
| SSH (local) | `ssh yourusername@192.168.1.100` |
| SSH (remote) | `ssh yourusername@100.x.x.x` |
| Netgear admin | `http://192.168.1.1` |
| XB7 admin | `http://10.0.0.1` |
| Tailscale admin | `https://login.tailscale.com/admin` |
