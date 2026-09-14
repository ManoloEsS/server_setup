# SSH And Tailscale

## Purpose

This module establishes local and remote administration for the Ubuntu server.

It covers:

- Verifying standard OpenSSH access over the local LAN.
- Installing and authenticating Tailscale.
- Enabling Tailscale SSH for remote administration.
- Applying separate firewall rules for LAN and Tailscale access.
- Testing both administration paths.
- Diagnosing common SSH and Tailscale failures.

The administration model uses:

- Standard OpenSSH for local LAN access and recovery.
- Tailscale SSH for remote access through the encrypted tailnet.

## Learning Objectives

After completing this module, an administrator should be able to:

- Explain the difference between LAN SSH and Tailscale SSH.
- Install and authenticate a Tailscale client.
- Identify the server's LAN and Tailscale addresses.
- Configure Tailscale SSH for identity-based remote access.
- Apply firewall rules scoped to the correct interface.
- Test local and remote administration separately.
- Diagnose network, firewall, authentication, and service failures.

## Prerequisites

| Requirement | Purpose |
|---|---|
| Ubuntu Server installed | Target server |
| Named administrative user | SSH administration |
| Working local SSH access | Initial configuration and recovery |
| Working LAN connectivity | Tailscale installation |
| Sudo access | Service and firewall changes |
| Tailscale account | Authorize server and clients |
| Second computer or device | Test local and remote access |

The server should already have the baseline firewall policy from
[Linux Baseline, Access, and Firewall](03-linux-baseline-access-and-firewall.md).

## Administration Model

The server has two separate SSH access paths:

```text
Local client
    |
    | Local LAN
    v
Server LAN address
    |
    +-- Standard OpenSSH
        Local administration and recovery

Remote client
    |
    | Tailscale encrypted overlay
    v
Server Tailscale address
    |
    +-- Tailscale SSH
        Remote administration
```

The two paths use the same Linux user account but different authentication
layers:

| Access path | Network | SSH service | Authentication |
|---|---|---|---|
| Local administration | LAN | System `sshd` | Password or SSH key |
| Remote administration | Tailscale | Tailscale SSH | Tailscale identity and policy |
| Console recovery | Physical console | None required | Local account |

Tailscale SSH does not modify `/etc/ssh/sshd_config` or
`~/.ssh/authorized_keys`. Local LAN connections continue to use the normal
OpenSSH service.

## Design Decisions

| Decision | Reason |
|---|---|
| Standard OpenSSH on the LAN | Provides a conventional administration and recovery path |
| Tailscale SSH remotely | Uses centralized identity-based access |
| No public SSH port forwarding | Reduces internet exposure |
| UFW rules scoped by interface | Separates LAN and Tailscale access |
| SSH keys for local administration | Demonstrates conventional Linux access management |
| Tailscale policy for remote administration | Allows centralized access and revocation |

Tailscale membership does not replace Linux permissions. The requested Linux
user must already exist on the server.

## Step 1: Verify Local OpenSSH Access

Keep the current SSH session open while configuring remote access.

Check the SSH service:

```bash
sudo systemctl status ssh --no-pager
```

Check the listening socket:

```bash
sudo ss -lntp | grep ':22'
```

From another computer on the same LAN, test the server's LAN address:

```bash
ssh <SERVER_USER>@<SERVER_LAN_IP>
```

After connecting, verify the session:

```bash
hostname
whoami
ip -br address
```

The output should identify the expected server, administrative user, and
network interfaces.

Before changing firewall or SSH settings:

1. Keep the current session open.
2. Open a second terminal.
3. Confirm that a new LAN SSH connection succeeds.
4. Use the second session for configuration changes.

The local SSH path is the primary recovery path if Tailscale or its policy is
misconfigured.

## Step 2: Confirm the Local Firewall Rule

Review the current UFW policy:

```bash
sudo ufw status verbose
sudo ufw status numbered
```

A LAN-scoped SSH rule should exist:

```bash
sudo ufw allow in from <LAN_CIDR> to any port 22 proto tcp
```

For example:

```bash
sudo ufw allow in from 192.168.1.0/24 to any port 22 proto tcp
```

Use the actual LAN range from the network plan. Do not open SSH globally when
access can be limited to the local network.

Test local SSH again after confirming the rule:

```bash
ssh <SERVER_USER>@<SERVER_LAN_IP>
```

## Step 3: Install Tailscale

Install Tailscale using the official installation method:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Confirm that the client is installed:

```bash
tailscale version
```

Check the Tailscale service:

```bash
sudo systemctl status tailscaled --no-pager
```

The `tailscaled` service should be active.

## Step 4: Authenticate the Server

Start Tailscale:

```bash
sudo tailscale up
```

A URL should be displayed. Open it from an administrative computer and
authenticate with the approved Tailscale account.

Verify the connection:

```bash
tailscale status
```

The server should appear as an authorized device in the tailnet.

Do not publish the authentication URL, account details, device identifiers, or
live infrastructure information in project evidence.

## Step 5: Identify the Tailscale Interface and Address

Inspect the network interfaces:

```bash
ip -br address
```

The Tailscale interface should appear as `tailscale0`.

Display the server's Tailscale IPv4 address:

```bash
tailscale ip -4
```

Record the values privately:

```text
Server LAN address: <SERVER_LAN_IP>
Server Tailscale address: <SERVER_TAILSCALE_IP>
Tailscale interface: tailscale0
```

Do not publish live addresses unless they have been intentionally sanitized.

Inspect the Tailscale connection:

```bash
tailscale status
tailscale netcheck
```

## Step 6: Configure Tailscale SSH

Enable Tailscale SSH on the server:

```bash
sudo tailscale set --ssh
```

Run this command from the local LAN SSH session or the server console rather
than from an existing SSH session using the server's Tailscale address.
Enabling Tailscale SSH can interrupt existing Tailscale SSH connections.

Tailscale SSH takes over port `22` only for traffic arriving through the
Tailscale interface. The standard OpenSSH service continues to handle local
LAN connections.

Tailscale SSH uses Tailscale node identity and tailnet policy for
authentication. It does not require copying a client public key into the
server user's `authorized_keys` file.

Confirm the Tailscale state:

```bash
tailscale status
tailscale ip -4
```

## Step 7: Review Tailscale Access Policy

Tailscale SSH requires two types of permission:

- Network access from the client to the server.
- SSH access for the intended Tailscale identity and Linux user.

Review the tailnet policy in the Tailscale admin console under **Access
controls**.

If the tailnet uses a custom policy, ensure it contains an equivalent network
grant and SSH rule. Use the actual identity, device, tag, and Linux username
for the environment.

Example policy structure:

```json
{
  "grants": [
    {
      "src": ["<ADMIN_IDENTITY>"],
      "dst": ["<SERVER_DEVICE_OR_TAG>"],
      "ip": ["22"]
    }
  ],
  "ssh": [
    {
      "action": "check",
      "src": ["<ADMIN_IDENTITY>"],
      "dst": ["<SERVER_DEVICE_OR_TAG>"],
      "users": ["<SERVER_USER>"]
    }
  ]
}
```

The `check` action requires periodic reauthentication. An `accept` action can
be used when repeated identity checks are not required.

Do not replace an existing tailnet policy with this example without reviewing
the complete policy first. Preserve unrelated network and device rules.

## Step 8: Allow Tailscale SSH Through UFW

Allow SSH through the Tailscale interface:

```bash
sudo ufw allow in on tailscale0 to any port 22 proto tcp
```

Review the active policy:

```bash
sudo ufw status verbose
sudo ufw status numbered
```

The intended policy is:

| Traffic | Expected policy |
|---|---|
| Incoming traffic by default | Deny |
| Outgoing traffic by default | Allow |
| SSH from local LAN | Allow |
| SSH through `tailscale0` | Allow |
| SSH from other interfaces | Deny unless specifically required |

Verify the system SSH listener:

```bash
sudo ss -lntp | grep ':22'
```

The presence of the standard SSH listener does not by itself verify Tailscale
SSH. Tailscale access must be tested from a connected client.

## Step 9: Test Local Administration

From a client connected to the home LAN:

```bash
ssh <SERVER_USER>@<SERVER_LAN_IP>
```

Verify:

```bash
hostname
whoami
uptime
```

The connection should use standard OpenSSH and should continue to work
regardless of whether Tailscale is connected.

Test the LAN SSH port if needed:

```bash
nc -vz <SERVER_LAN_IP> 22
```

A successful port test confirms reachability, but not successful
authentication.

## Step 10: Test Remote Administration

From an authorized client with Tailscale enabled, confirm that the server is
visible:

```bash
tailscale status
```

Test the overlay path:

```bash
tailscale ping <SERVER_TAILSCALE_IP>
```

Then connect using Tailscale SSH:

```bash
ssh <SERVER_USER>@<SERVER_TAILSCALE_IP>
```

Verify the session:

```bash
hostname
whoami
uptime
```

A successful connection confirms:

- The client and server are in the same authorized tailnet.
- The Tailscale path is working.
- The tailnet policy allows SSH.
- UFW permits SSH through `tailscale0`.
- The requested Linux user exists on the server.

## Step 11: Test From Outside the Home LAN

Use an authorized client on another network, such as a phone hotspot or
another trusted connection.

Confirm:

```bash
tailscale status
tailscale ping <SERVER_TAILSCALE_IP>
ssh <SERVER_USER>@<SERVER_TAILSCALE_IP>
```

Remote administration should work without:

- A public IP address for the server.
- Router port forwarding.
- Dynamic DNS.
- Direct internet exposure of TCP port 22.

The Tailscale connection provides the encrypted network path and identity
verification. The Linux account still controls what the administrator can do
after login.

## Optional: Local SSH Key Authentication

Tailscale SSH does not require normal SSH key distribution for remote
connections. SSH keys can still be configured for the standard LAN OpenSSH
path.

Generate a key on the local administration client if necessary:

```bash
ssh-keygen -t ed25519
```

Copy the public key to the server over the LAN:

```bash
ssh-copy-id <SERVER_USER>@<SERVER_LAN_IP>
```

Test local key authentication:

```bash
ssh <SERVER_USER>@<SERVER_LAN_IP>
```

This key is used by standard OpenSSH connections. It is not the authentication
mechanism used by Tailscale SSH.

Do not disable password authentication until a separate key-based login has
been tested successfully and a recovery path is available.

## Security Considerations

- Protect the Tailscale account with multi-factor authentication.
- Authorize only trusted devices in the tailnet.
- Review Tailscale access policies periodically.
- Use the narrowest practical source, destination, and Linux user rules.
- Keep local LAN SSH available as a documented recovery path.
- Do not expose SSH through public router port forwarding.
- Do not publish authentication URLs, device IDs, private keys, or live logs.
- Continue applying Ubuntu and Tailscale updates.
- Treat Tailscale membership as an access boundary, not as a replacement for
  Linux account permissions.

## Verification Checklist

| Check | Expected result |
|---|---|
| Local SSH service | System `sshd` is active and listening |
| LAN firewall rule | SSH is allowed from `<LAN_CIDR>` |
| Local SSH login | Login succeeds through `<SERVER_LAN_IP>` |
| Tailscale installed | `tailscale version` returns successfully |
| Tailscale service | `tailscaled` is active |
| Tailscale authorization | Server appears in `tailscale status` |
| Tailscale interface | `tailscale0` is present |
| Tailscale SSH | `sudo tailscale set --ssh` succeeds |
| Tailnet policy | Network and SSH access are permitted |
| Tailscale firewall rule | SSH is allowed on `tailscale0` |
| Remote SSH login | Login succeeds through `<SERVER_TAILSCALE_IP>` |
| External test | Remote login works outside the home LAN |
| Recovery path | Local LAN or console access remains available |
| Public exposure | No router SSH port forward exists |

## Common Problems

### The Tailscale command is not found

Check the installation and service:

```bash
tailscale version
sudo systemctl status tailscaled --no-pager
```

Inspect service logs if needed:

```bash
sudo journalctl -u tailscaled --no-pager -n 100
```

### The server does not appear in the tailnet

Check:

```bash
sudo tailscale status
sudo systemctl status tailscaled --no-pager
```

Run authentication again if required:

```bash
sudo tailscale up
```

Confirm that authorization was completed with the intended Tailscale account.

### `tailscale0` is missing

Check:

```bash
sudo systemctl status tailscaled --no-pager
tailscale status
ip link show tailscale0
```

If the service is stopped, restart it from the local LAN session or console:

```bash
sudo systemctl restart tailscaled
```

Then verify the interface again.

### Tailscale SSH is denied

Check:

- The client and server use the same tailnet.
- The server has Tailscale SSH enabled.
- The tailnet policy permits network access.
- The tailnet policy permits SSH access.
- The requested Linux user exists on the server.
- The client is using the correct Tailscale address.

Check the server state:

```bash
tailscale status
tailscale ip -4
```

Review the policy in the Tailscale admin console.

### Tailscale ping works but SSH fails

Check the firewall:

```bash
sudo ufw status numbered
```

Add the interface-scoped rule if necessary:

```bash
sudo ufw allow in on tailscale0 to any port 22 proto tcp
```

Confirm the standard SSH service is active:

```bash
sudo systemctl status ssh --no-pager
sudo ss -lntp | grep ':22'
```

### Local SSH works but remote SSH fails

Test the two paths separately:

```bash
ssh <SERVER_USER>@<SERVER_LAN_IP>
ssh <SERVER_USER>@<SERVER_TAILSCALE_IP>
```

If LAN SSH works but Tailscale SSH fails, inspect:

- Tailscale status.
- Tailscale SSH enablement.
- Tailnet policy.
- The `tailscale0` UFW rule.
- The server's Tailscale address.

### Remote SSH works but local SSH fails

Check the LAN rule and local network:

```bash
sudo ufw status verbose
ip -br address
ip route
```

Confirm that the client is connected to the intended LAN and that the server
has the expected LAN address.

### Enabling Tailscale SSH interrupts an existing connection

Connect through the LAN address or local console and verify:

```bash
sudo tailscale set --ssh
tailscale status
```

Do not repeatedly enable or disable the feature through the only available
remote session.

### Remote access stops after a reboot

Check:

```bash
sudo systemctl status tailscaled --no-pager
tailscale status
tailscale ip -4
sudo systemctl status ssh --no-pager
sudo ufw status verbose
```

Both Tailscale and SSH should start automatically.

### DNS fails after Tailscale is installed

Remote SSH connectivity and DNS resolution are separate concerns. Follow the
resolver procedures in
[Networking, DNS, and Router Configuration](04-networking-dns-and-router.md).

Remote Pi-hole DNS configuration is handled later after Pi-hole is deployed.

## Evidence To Capture

Capture sanitized evidence showing:

- The Tailscale client version.
- The `tailscaled` service status.
- The presence of `tailscale0`.
- A redacted `tailscale status` result.
- The local LAN UFW rule.
- The Tailscale UFW rule.
- A successful LAN SSH login.
- A successful Tailscale SSH login.
- A successful test from outside the home LAN.
- The absence of public router port forwarding.

Remove or replace:

- Live Tailscale IP addresses.
- Device IDs.
- Account names and email addresses.
- Host fingerprints.
- Usernames.
- Private keys.
- Authentication URLs.
- Unsanitized logs.

## Completion Criteria

This module is complete when:

- Standard OpenSSH works through the local LAN address.
- The local LAN firewall rule is verified.
- Tailscale is installed and authenticated.
- The server has a working `tailscale0` interface.
- Tailscale SSH is enabled.
- Tailnet network and SSH policies permit the intended access.
- UFW allows SSH through `tailscale0`.
- Remote SSH works through the Tailscale address.
- Remote access works from outside the home LAN.
- Local LAN or console access remains available for recovery.
- No public router port forwarding is required.
- Sanitized evidence has been captured.

Next: [Docker Host](06-docker-host.md)
