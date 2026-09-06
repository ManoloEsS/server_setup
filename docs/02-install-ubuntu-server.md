# Ubuntu Server Installation

## Purpose

This module documents how to install Ubuntu Server on a small-form-factor computer and establish the first administrative connection.

The goal is to move from bare hardware to a reachable Linux server with:

- Ubuntu Server 24.04 LTS installed.
- A named server and administrative user.
- A working network connection.
- OpenSSH enabled.
- A verified first login over the local network.

System hardening, firewall configuration, Docker, and application services are covered in later modules.

## Learning Objectives

After completing this module, an administrator should be able to:

- Install a server operating system using standard installation media.
- Select the correct installation disk and preserve required data.
- Configure a server hostname and administrative user.
- Identify the server's network interface and assigned address.
- Connect to the server with SSH.
- Verify basic operating system and network information.
- Recognize common installation and first-connection failures.

## Prerequisites

| Requirement | Purpose |
|---|---|
| Small-form-factor PC or mini PC | Target server hardware |
| Ubuntu Server 24.04 LTS image | Operating system installation source |
| Second computer | Obtain the image and connect to the server |
| Wired Ethernet connection | Reliable initial network access |
| Display and keyboard | Initial installation and console login |
| Network with DHCP enabled | Temporary address assignment during installation |

The installation media and the selected server disk must be treated as destructive operations. Confirm that any important data has been backed up and that the correct disk is selected before installation.

## Installation Decisions

| Decision | Reason |
|---|---|
| Ubuntu Server LTS | Provides a stable and well-documented server platform |
| Wired Ethernet | Reduces connection problems during installation and service setup |
| DHCP during installation | Allows the server to obtain temporary network settings automatically |
| DHCP reservation later | Keeps address management centralized at the router |
| OpenSSH server | Enables remote administration after the first boot |
| No featured snaps initially | Keeps the base installation focused on the required server services |
| Guided whole-disk storage | Simplifies the lab installation on a dedicated server disk |

## Install Ubuntu Server

Install Ubuntu Server 24.04 LTS on the dedicated server disk using the standard installation media and guided storage layout. The image is available from <https://ubuntu.com/download/server>.

Use the following baseline configuration:

- Wired Ethernet with DHCP enabled.
- A descriptive hostname such as `<SERVER_HOSTNAME>`.
- A named administrative user.
- OpenSSH server enabled.
- No additional featured services or snaps unless required.
- The entire disk may be used if it contains no data that must be preserved.

Record the following values securely:

- Administrative username.
- Hostname.
- Temporary LAN address.
- Default gateway.

Do not place passwords, private keys, or other credentials in documentation or screenshots.

After installation, reboot from the internal disk and log in at the local console. The remaining sections in this module begin after the operating system is installed.

## Identify the Server's Network Settings

From the server console, inspect the network configuration:

```bash
ip -br address
ip route
hostnamectl
```

Look for:

- The active Ethernet interface.
- An address in the local LAN range.
- A default route through the primary router.
- The configured hostname.

Use the following placeholders when recording the result:

```text
Server hostname: <SERVER_HOSTNAME>
Server user: <SERVER_USER>
LAN interface: <SERVER_INTERFACE>
Temporary LAN address: <SERVER_LAN_IP>
Default gateway: <ROUTER_LAN_IP>
```

The address may change until a DHCP reservation is configured. The reservation is handled in the networking and router modules.

## Connect Over SSH

From the administrative computer, connect using the server's current LAN address:

```bash
ssh <SERVER_USER>@<SERVER_LAN_IP>
```

On the first connection, verify the host fingerprint before accepting it. Enter the server user's password when prompted.

After connecting, verify the session:

```bash
hostname
whoami
uptime
```

The output should identify the expected server, the expected user, and a running system.

## Initial Verification Checklist

| Check | Expected result |
|---|---|
| Server boots without the installation media | Ubuntu starts from the internal disk |
| Ethernet interface is active | The expected interface has a LAN address |
| Default route exists | Traffic is routed through the primary router |
| Hostname is correct | The configured server name is displayed |
| SSH connection succeeds | The administrative user can connect from another computer |
| Current user is correct | `whoami` returns the intended administrative username |
| SSH service is available | The OpenSSH service is running |

The next module handles system updates, administrative baseline configuration, firewall policy, and service management.

## Common Problems

### The server boots into the existing operating system

Check that:

- The installation media was created successfully.
- The media is connected before powering on.
- The correct boot-menu key is being used.
- The installation device is selected as the boot target.

### No network address appears

Check that:

- The Ethernet cable is connected at both ends.
- The cable and router port are working.
- The interface is enabled.
- The router's DHCP service is active.

The server can be inspected locally even when network access is unavailable.

### SSH connection is refused

Check that:

- The client is using the server's current address.
- The OpenSSH server option was selected during installation.
- The server is powered on and connected to the same LAN.
- The SSH service is running at the server console.

### SSH connects to the wrong system

This can happen when an address was previously assigned to another device. Confirm the server's current address and verify the host fingerprint before continuing.

### The password is rejected

Confirm the username and keyboard layout used during installation. If the account password cannot be recovered, use the documented account recovery process rather than repeatedly guessing credentials.

## Evidence To Capture

Capture sanitized evidence showing:

- The Ubuntu release and installation date.
- The server hostname.
- The active network interface.
- The assigned address and default route.
- A successful SSH connection.
- The verification output from `hostname`, `whoami`, and `uptime`.

Do not include passwords, private keys, host fingerprints, personal usernames, or unnecessary live network details in published evidence.

## Completion Criteria

This module is complete when:

- Ubuntu Server boots from the internal disk.
- The administrative account can log in locally.
- The server has working Ethernet connectivity.
- The hostname is configured.
- OpenSSH is installed and running.
- A remote SSH login succeeds.
- The initial network and identity checks have been recorded.

Next: [Linux Baseline, Access, and Firewall](03-linux-baseline-access-and-firewall.md)
