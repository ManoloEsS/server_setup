# Linux Baseline, Access, And Firewall

## Purpose

This module establishes the Linux administration baseline after Ubuntu Server
installation.

It covers:

- Updating the operating system.
- Verifying administrative access.
- Reviewing users, groups, permissions, and storage.
- Confirming SSH availability.
- Applying a host-based firewall policy with UFW.
- Establishing a repeatable baseline for later service deployment.

Application-specific configuration is covered in later modules.

## Learning Objectives

After completing this module, an administrator should be able to:

- Update Ubuntu packages safely.
- Inspect users, groups, permissions, storage, and system services.
- Apply administrative changes through `sudo`.
- Verify SSH before changing firewall rules.
- Define firewall rules based on service and network scope.
- Confirm that required ports are listening and reachable.
- Diagnose common access and firewall failures.

## Baseline Assumptions

| Item                  | Expected value                            |
| --------------------- | ----------------------------------------- |
| Operating system      | Ubuntu Server 24.04 LTS                   |
| Administrative access | Named user with `sudo` access             |
| Primary interface     | `<SERVER_INTERFACE>`                      |
| Local network         | `<LAN_CIDR>`                              |
| Server LAN address    | `<SERVER_LAN_IP>`                         |
| Router address        | `<ROUTER_LAN_IP>`                         |
| Remote interface      | `tailscale0`, when Tailscale is installed |

## Step 1: Update the Operating System

Update package metadata and install available upgrades:

```bash
sudo apt update
sudo apt upgrade -y
```

Optionally remove packages that are no longer required:

```bash
sudo apt autoremove -y
```

Confirm the operating system and kernel versions:

```bash
lsb_release -a
uname -r
```

A server should be updated before additional services are installed. This
reduces the chance of building services on top of outdated packages or known
vulnerabilities.

## Step 2: Review the Administrative Account

Confirm the current account:

```bash
whoami
id
```

The account used for administration should:

- Have a unique username.
- Have `sudo` access.
- Use a strong password or SSH key.
- Avoid running routine commands as the root user.
- Be used instead of direct root login.

Confirm that `sudo` works:

```bash
sudo -v
```

The command should complete without an authentication or permission error.

Review local accounts when needed:

```bash
getent passwd
```

Avoid deleting system accounts unless their purpose is understood. Many service
accounts are created for system processes and should not be treated as
interactive users.

## Step 3: Review Hostname, Time, Storage, and Resources

Confirm the hostname:

```bash
hostnamectl
```

Confirm the system clock and time synchronization:

```bash
timedatectl
```

Check storage capacity and mounted filesystems:

```bash
df -h
lsblk
```

Check memory and system load:

```bash
free -h
uptime
```

These checks establish a baseline before application data is stored on the
server.

Record any capacity or hardware constraints that may affect service reliability.
A server hosting file shares and containers should have enough free storage for
application data, logs, updates, and backups.

## Step 4: Verify SSH Access

Confirm the SSH service status:

```bash
sudo systemctl status ssh --no-pager
```

Confirm that SSH is listening:

```bash
sudo ss -lntp | grep ':22'
```

The service should be listening on the expected interface and port.

Before changing SSH or firewall configuration:

1. Keep the current SSH session open.
2. Open a second terminal.
3. Test a new SSH connection.
4. Make changes only after confirming that a second connection works.

This prevents a configuration or firewall change from locking out the active
administrator.

## SSH Security Baseline

The initial SSH baseline should include:

- Direct root login disabled.
- Password authentication retained until SSH key authentication is verified.
- SSH keys used for routine administration when possible.
- Access restricted to the local network and authorized Tailscale clients.
- Unnecessary SSH forwarding features disabled when they are not required.
- The SSH service kept updated through normal system updates.

Review the effective SSH configuration:

```bash
sudo sshd -T | less
```

If SSH configuration is changed, validate it before restarting the service:

```bash
sudo sshd -t
```

Only restart SSH after the configuration test succeeds:

```bash
sudo systemctl restart ssh
```

Do not disable password authentication until a separate SSH key login has been
tested successfully.

## Step 5: Install and Enable UFW

Install UFW if it is not already present:

```bash
sudo apt install ufw -y
```

Set conservative default policies:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

Do not enable UFW until an SSH allow rule has been added and a second SSH
session has been tested.

## Step 6: Define Firewall Rules

Firewall rules should be based on:

- The service being protected.
- The protocol and port required.
- The network or interface that should reach it.
- Whether remote Tailscale access is required.

Allow SSH from the local LAN:

```bash
sudo ufw allow in from <LAN_CIDR> to any port 22 proto tcp
```

If Tailscale is installed, allow SSH through the Tailscale interface:

```bash
sudo ufw allow in on tailscale0 to any port 22 proto tcp
```

The following service rules are added as the services are deployed:

```bash
# Samba
sudo ufw allow in from <LAN_CIDR> to any port 445 proto tcp
sudo ufw allow in from <LAN_CIDR> to any port 139 proto tcp

# Gitea web interface
sudo ufw allow in from <LAN_CIDR> to any port 3000 proto tcp

# Gitea SSH
sudo ufw allow in from <LAN_CIDR> to any port 222 proto tcp

# Pi-hole web interface
sudo ufw allow in from <LAN_CIDR> to any port 80 proto tcp

# Pi-hole DNS from the local LAN
sudo ufw allow in on <SERVER_INTERFACE> from <LAN_CIDR> to any port 53 proto udp
sudo ufw allow in on <SERVER_INTERFACE> from <LAN_CIDR> to any port 53 proto tcp

# Pi-hole DNS from Tailscale clients
sudo ufw allow in on tailscale0 to any port 53 proto udp
sudo ufw allow in on tailscale0 to any port 53 proto tcp
```

Only add rules for services that are installed and required. Avoid opening a
port globally when access can be limited to a known network or interface.

Enable UFW after the required rules are present:

```bash
sudo ufw enable
```

Confirm the active policy:

```bash
sudo ufw status verbose
sudo ufw status numbered
```

## Firewall Policy

The intended policy is:

| Traffic                                  | Policy                               |
| ---------------------------------------- | ------------------------------------ |
| Incoming connections by default          | Deny                                 |
| Outgoing connections by default          | Allow                                |
| SSH from the local LAN                   | Allow                                |
| SSH from authorized Tailscale clients    | Allow                                |
| Service ports from unauthorized networks | Deny                                 |
| DNS from the local LAN                   | Allow after Pi-hole is installed     |
| DNS from Tailscale clients               | Allow after remote DNS is configured |

UFW rules should be reviewed whenever a service is added, removed, or moved to a
different port.

## Step 7: Verify Listening Services

List listening TCP and UDP sockets:

```bash
sudo ss -lntup
```

At this stage, expected listeners may include:

- SSH on TCP port 22.
- System services required by Ubuntu.
- No application ports that have not yet been deployed.

After later modules install services, verify that each listener matches the
documented service port plan. A listening port alone does not prove that a
service is correctly configured; test it from an appropriate client as well.

## Step 8: Verify Local Connectivity

Check the server's address and route:

```bash
ip -br address
ip route
```

Test the default gateway:

```bash
ping -c 3 <ROUTER_LAN_IP>
```

Test name resolution:

```bash
getent hosts ubuntu.com
```

If name resolution fails, distinguish between:

- A network route problem.
- A DNS configuration problem.
- A firewall problem.
- An upstream connectivity problem.

Do not assume that a successful ping proves that DNS or application services are
working.

## Verification Checklist

| Check                    | Expected result                                           |
| ------------------------ | --------------------------------------------------------- |
| Package metadata updates | `apt update` completes without repository errors          |
| System packages updated  | `apt upgrade` completes successfully                      |
| Administrative account   | Named user has working `sudo` access                      |
| Hostname                 | Expected hostname is configured                           |
| Time synchronization     | System clock is synchronized or correctly configured      |
| Storage                  | Required filesystems are mounted with adequate free space |
| SSH service              | Service is active and listening on the intended port      |
| Second SSH session       | New administrative connection succeeds                    |
| UFW defaults             | Incoming denied and outgoing allowed                      |
| UFW rules                | Only required service and network rules are present       |
| Local gateway            | Server can reach the primary router                       |
| DNS baseline             | Server can resolve a known domain                         |

## Common Problems

### SSH access is lost after enabling UFW

Use the existing console or an already-open administrative session to inspect
the rules:

```bash
sudo ufw status numbered
```

Add a correctly scoped SSH rule before removing or changing other rules:

```bash
sudo ufw allow in from <LAN_CIDR> to any port 22 proto tcp
```

If the server is remote and no console access is available, use the hosting or
hardware management method available for the environment.

### UFW rule exists but traffic is still blocked

Check:

- The source network is correct.
- The client is using the expected interface.
- The service is listening on the expected address and port.
- The rule uses the correct protocol.
- Another firewall layer is not blocking the traffic.

Inspect listening ports:

```bash
sudo ss -lntup
```

### A service works locally but not from another device

Compare local and remote tests. Check:

- The service bind address.
- The server firewall.
- The router or client network.
- The destination address and port.
- Whether the client is using the LAN or Tailscale address.

### Package updates fail because DNS is unavailable

Check the route and resolver configuration:

```bash
ip route
resolvectl status
resolvectl query archive.ubuntu.com
```

DNS service configuration is handled in the networking and Pi-hole modules.

### SSH password authentication is disabled too early

Use a local console or an existing administrative session to restore access. SSH
key authentication must be verified from a separate terminal before disabling
password authentication.

## Evidence To Capture

Capture sanitized evidence showing:

- Successful package updates.
- The administrative user's group and `sudo` access.
- Hostname, time, storage, and resource baseline.
- SSH service status and a successful second connection.
- The UFW default policy and numbered rules.
- Listening ports.
- Successful gateway and DNS tests.

## Completion Criteria

This module is complete when:

- The system is updated.
- The administrative account and `sudo` access are verified.
- Hostname, time, storage, and resources have been reviewed.
- SSH access is confirmed from a second session.
- UFW is enabled with a deny-incoming policy.
- Required access rules are scoped to the appropriate network or interface.
- Listening ports and basic connectivity are verified.
- Sanitized baseline evidence has been recorded.

Next:
[Networking, DNS, and Router Configuration](04-networking-dns-and-router.md)
