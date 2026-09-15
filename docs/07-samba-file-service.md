# Samba File Service

## Purpose

This module deploys a private Samba file share on the Ubuntu server.

It covers:

- Installing and enabling Samba.
- Preparing `/srv/files` with controlled permissions.
- Creating Samba authentication for authorized Linux users.
- Configuring a non-guest SMB share.
- Restricting SMB access to the LAN and Tailscale interfaces.
- Validating the Samba configuration.
- Testing local and remote file access.
- Mounting the share from a Linux client.
- Diagnosing authentication, permissions, firewall, and connectivity failures.

The share is intended for private LAN and authorized Tailscale access. SMB must
not be exposed directly to the public internet.

## Learning Objectives

After completing this module, an administrator should be able to:

- Explain the difference between Linux filesystem permissions and Samba
  permissions.
- Configure a private SMB share without guest access.
- Create and manage Samba credentials.
- Use groups to control access to shared files.
- Validate Samba configuration before restarting the service.
- Restrict SMB traffic with UFW.
- Test file access from local and remote clients.
- Diagnose common Samba failures.
- Document backup and recovery requirements for shared data.

## Prerequisites

| Requirement | Purpose |
|---|---|
| Ubuntu Server installed | Samba host |
| Working local SSH access | Administration and recovery |
| Working Tailscale access | Optional remote file access |
| Administrative user with `sudo` | Package and configuration changes |
| Docker host completed | Provides the planned `/srv` layout |
| LAN address or DHCP reservation | Local client access |
| Tailscale address | Remote client access |
| Available storage | Shared files and backups |

Complete these modules first:

- [Ubuntu Server Installation](02-install-ubuntu-server.md)
- [Linux Baseline, Access, and Firewall](03-linux-baseline-access-and-firewall.md)
- [Networking, DNS, and Router Configuration](04-networking-dns-and-router.md)
- [SSH and Tailscale](05-ssh-and-tailscale.md)
- [Docker Host](06-docker-host.md)

## Service Design

The Samba service provides one private share:

```text
Local LAN client
    |
    | SMB over TCP 445
    v
Ubuntu server
    |
    +-- smbd
          |
          +-- /srv/files

Remote Tailscale client
    |
    | Encrypted Tailscale path
    v
Ubuntu server
    |
    +-- smbd
          |
          +-- /srv/files
```

The intended access paths are:

| Client type | Server address | Protocol |
|---|---|---|
| Local client | `<SERVER_LAN_IP>` | SMB over TCP 445 |
| Remote client | `<SERVER_TAILSCALE_IP>` | SMB over TCP 445 through Tailscale |

No router port forwarding is required.

## Design Decisions

| Decision | Reason |
|---|---|
| Private authenticated share | Prevents anonymous access |
| Guest access disabled | Requires an identified user |
| SMB2 or newer | Avoids obsolete SMB1 behavior |
| TCP 445 as the default port | Modern direct-host SMB access |
| TCP 139 disabled unless required | Avoids legacy NetBIOS exposure |
| Dedicated `fileshare` group | Separates file access from unrelated permissions |
| No `force user` setting | Preserves user identity and file ownership |
| LAN and Tailscale access only | Prevents public SMB exposure |
| Root-owned Samba configuration | Protects service settings |
| Separate backup procedure | A file share is not a backup |

The `force user` option is intentionally not used. It can simplify a
single-user setup, but it causes files to appear as if they were created by one
forced account and reduces identity separation.

## Step 1: Install Samba

Update package metadata:

```bash
sudo apt update
```

Install Samba and the command-line client utilities:

```bash
sudo apt install samba smbclient
```

Verify the installed version:

```bash
smbd --version
```

Check the Samba service:

```bash
sudo systemctl status smbd --no-pager
```

Enable and start the service:

```bash
sudo systemctl enable --now smbd
```

Verify that it starts automatically:

```bash
systemctl is-active smbd
systemctl is-enabled smbd
```

Expected results:

```text
active
enabled
```

## Step 2: Review the Existing Filesystem

Confirm that the planned share directory exists:

```bash
sudo ls -ld /srv/files
```

Review the filesystem capacity:

```bash
df -h /srv/files
```

Review the underlying block devices:

```bash
lsblk
```

The share requires sufficient capacity for:

- User files.
- Temporary files.
- Samba logs.
- Files retained for recovery.
- Backup staging, if used.

Do not treat the server's only copy of `/srv/files` as a backup.

## Step 3: Create the Share Group

Create a dedicated system group for users allowed to access the share:

```bash
getent group fileshare || sudo groupadd --system fileshare
```

Add the intended Linux user to the group:

```bash
sudo usermod -aG fileshare <SERVER_USER>
```

A new login session is normally required before the group membership appears.
Verify it after reconnecting:

```bash
id <SERVER_USER>
```

Only add users who require access to the share.

Group membership controls filesystem access. Samba credentials are configured
separately in the next step.

## Step 4: Set Filesystem Ownership and Permissions

Assign the share directory to the root account and the dedicated share group:

```bash
sudo chown root:fileshare /srv/files
```

Set the directory permissions:

```bash
sudo chmod 2770 /srv/files
```

The `2` in `2770` enables the setgid bit. New directories created inside the
share inherit the `fileshare` group.

Review the result:

```bash
sudo ls -ld /srv/files
```

Expected characteristics:

- Owner: `root`.
- Group: `fileshare`.
- Owner and group have access.
- Other users have no access.
- New content inherits the shared group.

Existing files may require a separate permission review:

```bash
sudo find /srv/files -maxdepth 2 -printf '%M %u %g %p\n'
```

Do not recursively change all file permissions without reviewing the impact on
existing users and data.

## Step 5: Create Samba Authentication

A Samba account must correspond to an existing Linux account.

Add the Linux user to Samba's local password database:

```bash
sudo smbpasswd -a <SERVER_USER>
```

Enable the Samba account:

```bash
sudo smbpasswd -e <SERVER_USER>
```

Samba credentials are separate from the Linux login password. The passwords
may be the same, but they do not have to be.

Review Samba account entries when needed:

```bash
sudo pdbedit -L
```

Do not publish the output if it contains real usernames or other identifying
information.

To change the Samba password later:

```bash
sudo smbpasswd <SERVER_USER>
```

To disable a Samba account without deleting the Linux account:

```bash
sudo smbpasswd -d <SERVER_USER>
```

## Step 6: Back Up the Samba Configuration

Before changing the configuration, create a protected backup:

```bash
sudo cp --preserve=mode,ownership /etc/samba/smb.conf \
  /etc/samba/smb.conf.bak
```

Review the existing configuration:

```bash
sudo testparm -s
```

Do not replace the complete configuration file unless the existing settings
have been reviewed. Preserve unrelated global settings required by the
environment.

## Step 7: Configure the Private Share

Edit the configuration using `sudoedit`:

```bash
sudoedit /etc/samba/smb.conf
```

Add or update the global settings so that guest access is not used and obsolete
SMB1 connections are not accepted:

```ini
[global]
   map to guest = Never
   server min protocol = SMB2
```

Add the share definition:

```ini
[files]
   path = /srv/files
   browseable = yes
   read only = no
   guest ok = no
   valid users = @fileshare
   force group = fileshare
   create mask = 0660
   directory mask = 2770
   inherit permissions = yes
```

Configuration notes:

- `path` identifies the Linux directory being shared.
- `browseable` allows authenticated clients to see the share.
- `read only = no` permits authorized writes.
- `guest ok = no` prevents anonymous access.
- `valid users` limits access to members of `fileshare`.
- `force group` applies the shared group without changing the file owner.
- `create mask` controls permissions for new files.
- `directory mask` controls permissions for new directories.
- `inherit permissions` keeps new content aligned with the share directory.

Do not add `force user`. It would make files appear to belong to one forced
user and would reduce identity separation.

## Step 8: Validate the Configuration

Validate the complete Samba configuration:

```bash
sudo testparm
```

Review the effective share configuration:

```bash
sudo testparm -s
```

The validation should complete without syntax errors.

If validation fails:

1. Read the reported file and line number.
2. Check for duplicate or misspelled parameters.
3. Check section names and indentation.
4. Confirm that the share path exists.
5. Correct the configuration.
6. Run `testparm` again.

Do not restart Samba while the configuration is invalid.

## Step 9: Restart and Verify Samba

Restart Samba after the configuration passes validation:

```bash
sudo systemctl restart smbd
```

Check the service:

```bash
sudo systemctl status smbd --no-pager
```

Check the listening port:

```bash
sudo ss -lntp | grep ':445'
```

The service should listen on TCP port `445`.

Check whether the legacy NetBIOS service is active:

```bash
sudo systemctl status nmbd --no-pager
```

Direct SMB access by IP address or hostname does not normally require `nmbd`.
If legacy NetBIOS discovery is not required, it should not be enabled as an
additional service.

## Step 10: Configure the Firewall

Allow SMB from the local LAN:

```bash
sudo ufw allow in on <SERVER_INTERFACE> \
  from <LAN_CIDR> to any port 445 proto tcp
```

For example:

```bash
sudo ufw allow in on enp2s0 \
  from 192.168.1.0/24 to any port 445 proto tcp
```

Allow SMB from authorized Tailscale clients:

```bash
sudo ufw allow in on tailscale0 \
  to any port 445 proto tcp
```

Review the active rules:

```bash
sudo ufw status verbose
sudo ufw status numbered
```

TCP port `139` should not be opened by default. If a specific legacy client
requires it, add a narrowly scoped rule only after documenting the requirement:

```bash
sudo ufw allow in on <SERVER_INTERFACE> \
  from <LAN_CIDR> to any port 139 proto tcp
```

Do not allow SMB from the public internet and do not create router port
forwarding for ports `445` or `139`.

## Step 11: Test the Share From the Server

List available shares locally:

```bash
smbclient -L localhost -U <SERVER_USER>
```

Connect to the share locally:

```bash
smbclient //localhost/files -U <SERVER_USER>
```

Inside the `smbclient` prompt, test basic operations:

```text
dir
mkdir verification
cd verification
put <LOCAL_TEST_FILE>
dir
del <LOCAL_TEST_FILE>
cd ..
rmdir verification
quit
```

Use a non-sensitive test file. Confirm that the test file is created under
`/srv/files` with the expected ownership and group:

```bash
sudo find /srv/files -maxdepth 2 -printf '%M %u %g %p\n'
```

A successful local test confirms that Samba, the Samba account, and the
filesystem permissions work together. It does not prove that remote firewall
access works.

## Step 12: Test Local LAN Access

From a client connected to the local LAN, test the share:

```bash
smbclient //<SERVER_LAN_IP>/files -U <SERVER_USER>
```

A graphical file manager can use:

```text
smb://<SERVER_LAN_IP>/files
```

Test the following:

- Authentication succeeds with the Samba credentials.
- The share is visible.
- A test file can be created.
- The test file can be read.
- The test file can be deleted.
- Unauthorized users are denied.
- Guest access is denied.

Test TCP reachability if needed:

```bash
nc -vz <SERVER_LAN_IP> 445
```

A successful port test confirms reachability, not authorization.

## Step 13: Test Remote Tailscale Access

From an authorized client outside the home LAN, ensure Tailscale is connected:

```bash
tailscale status
```

Test the server's Tailscale path:

```bash
tailscale ping <SERVER_TAILSCALE_IP>
```

Connect to the share using the Tailscale address:

```bash
smbclient //<SERVER_TAILSCALE_IP>/files -U <SERVER_USER>
```

A file manager can use:

```text
smb://<SERVER_TAILSCALE_IP>/files
```

Test creating, reading, and deleting a non-sensitive file.

Remote Samba access should work without:

- A public server address.
- Router port forwarding.
- Dynamic DNS.
- SMB exposed to the internet.

Both the client and server must be connected to the same authorized tailnet.

## Step 14: Mount the Share on a Linux Client

Install the CIFS utilities on the client.

Ubuntu or Debian:

```bash
sudo apt install cifs-utils
```

Arch Linux:

```bash
sudo pacman -S cifs-utils
```

Create a mount point:

```bash
sudo install -d -o "$USER" -g "$USER" -m 0750 /mnt/n9
```

Create a credentials file:

```bash
install -m 0600 /dev/null "$HOME/.smb-credentials"
```

Edit it:

```bash
${EDITOR:-nano} "$HOME/.smb-credentials"
```

Use this format:

```text
username=<SERVER_USER>
password=<SAMBA_PASSWORD>
domain=
```

Never place the Samba password directly in `/etc/fstab`.

Test a manual local mount:

```bash
sudo mount -t cifs //<SERVER_LAN_IP>/files /mnt/n9 \
  -o credentials="$HOME/.smb-credentials",uid="$(id -u)",gid="$(id -g)",\
vers=3.1.1,nosuid,nodev,noexec
```

Verify the mount:

```bash
findmnt /mnt/n9
ls -la /mnt/n9
```

Unmount it after testing:

```bash
sudo umount /mnt/n9
```

For a persistent local mount, add an entry to `/etc/fstab`:

```text
//<SERVER_LAN_IP>/files /mnt/n9 cifs credentials=/home/<CLIENT_USER>/.smb-credentials,uid=<CLIENT_UID>,gid=<CLIENT_GID>,vers=3.1.1,_netdev,x-systemd.automount,nofail,nosuid,nodev,noexec 0 0
```

Validate the entry:

```bash
sudo mount /mnt/n9
findmnt /mnt/n9
```

Use the Tailscale address instead of the LAN address only for a remote client.
Remote automatic mounts may require the client Tailscale service to be active
before the mount is attempted.

## Access Control Model

Access requires all of the following:

```text
Client reaches server
    |
    +-- Correct LAN or Tailscale path
    |
    +-- UFW allows TCP 445
    |
    +-- Samba account is enabled
    |
    +-- User is in the fileshare group
    |
    +-- Linux filesystem permissions allow access
    |
    +-- Samba share permissions allow access
```

A failure at any layer can produce an access error.

Samba authentication and Linux filesystem permissions are separate checks. A
valid Samba password does not override Linux filesystem permissions.

## Security Considerations

- Keep guest access disabled.
- Use SMB2 or newer and do not enable SMB1.
- Use strong, unique Samba passwords.
- Limit membership in the `fileshare` group.
- Do not use `force user` for convenience.
- Do not expose TCP ports `445` or `139` to the public internet.
- Do not create router port forwarding for SMB.
- Restrict UFW rules to the LAN and Tailscale interfaces.
- Keep `/etc/samba/smb.conf` root-owned.
- Protect client credentials files with mode `0600`.
- Do not place passwords in `/etc/fstab`.
- Review Samba logs when investigating access.
- Back up shared data to a separate storage target.
- Test restoring at least one file from backup.
- Remove or disable Samba accounts that no longer require access.

## Verification Checklist

| Check | Expected result |
|---|---|
| Samba installed | `smbd --version` succeeds |
| Samba service | `smbd` is active |
| Samba startup | Service is enabled at boot |
| Share directory | `/srv/files` exists |
| Share ownership | Directory uses the intended owner and group |
| Share permissions | Only authorized users have filesystem access |
| Samba account | Intended user is enabled |
| Guest access | Anonymous access is denied |
| SMB protocol | SMB1 is not accepted |
| Configuration | `testparm` reports no errors |
| SMB listener | TCP port 445 is listening |
| LAN firewall | TCP 445 is allowed from the LAN |
| Tailscale firewall | TCP 445 is allowed on `tailscale0` |
| Local test | Share works through `<SERVER_LAN_IP>` |
| Remote test | Share works through `<SERVER_TAILSCALE_IP>` |
| Write test | Authorized user can create and delete a test file |
| Unauthorized test | Unapproved user is denied |
| Client mount | CIFS mount succeeds |
| Credentials | Client credentials file is mode `0600` |
| Backup | Shared data backup procedure is documented |

## Common Problems

### Samba will not start

Check the service:

```bash
sudo systemctl status smbd --no-pager
```

Validate the configuration:

```bash
sudo testparm
```

Review service logs:

```bash
sudo journalctl -u smbd --no-pager -n 100
```

Common causes include invalid configuration syntax, duplicate or misspelled
options, a missing share path, a port conflict, and filesystem permission
errors.

### `testparm` reports an error

Read the reported line and parameter. Check:

```bash
sudo testparm -s
```

Confirm that:

- Section headers use square brackets.
- Paths exist.
- Options are spelled correctly.
- The `fileshare` group exists.
- The configuration does not contain conflicting duplicate settings.

Do not restart Samba until validation succeeds.

### The share is reachable but access is denied

Check the Samba account:

```bash
sudo pdbedit -L
```

Check Linux group membership:

```bash
id <SERVER_USER>
getent group fileshare
```

Check filesystem permissions:

```bash
sudo ls -ld /srv/files
sudo find /srv/files -maxdepth 2 -printf '%M %u %g %p\n'
```

Confirm that the user is included in the share:

```ini
valid users = @fileshare
```

Restart or reconnect the client after changing group membership.

### The Samba password is rejected

Reset the Samba password:

```bash
sudo smbpasswd <SERVER_USER>
```

Confirm that the account is enabled:

```bash
sudo smbpasswd -e <SERVER_USER>
```

The Samba password is separate from the Linux password.

### The share works locally but not from the LAN

Check the listener:

```bash
sudo ss -lntp | grep ':445'
```

Check the server address and interface:

```bash
ip -br address
ip route
```

Check UFW:

```bash
sudo ufw status verbose
sudo ufw status numbered
```

Confirm that the client is using the server's LAN address and is connected to
the intended router.

### The share works on the LAN but not through Tailscale

Check both devices:

```bash
tailscale status
```

Test the overlay:

```bash
tailscale ping <SERVER_TAILSCALE_IP>
```

Check the Tailscale firewall rule:

```bash
sudo ufw status numbered
```

Confirm that TCP port `445` is allowed on `tailscale0`.

Also confirm that the client is using the Tailscale address, not the LAN
address.

### The client attempts guest access

Ensure the share contains:

```ini
guest ok = no
map to guest = Never
```

Remove saved anonymous or incorrect credentials from the client and reconnect
using the intended Samba account.

### SMB1 or legacy discovery is required

Do not enable SMB1 automatically. First identify the affected client and confirm
that it cannot use direct SMB2 or SMB3 access.

Prefer direct access using:

```text
smb://<SERVER_LAN_IP>/files
```

or:

```text
smb://<SERVER_TAILSCALE_IP>/files
```

If TCP port `139` is genuinely required, document the client requirement and
scope the firewall rule to the smallest necessary network.

### The CIFS mount fails

Check the mount point:

```bash
findmnt /mnt/n9
```

Check the credentials file:

```bash
ls -l "$HOME/.smb-credentials"
```

It should be readable only by the client user.

Test the share manually:

```bash
smbclient //<SERVER_LAN_IP>/files -U <SERVER_USER>
```

Check the client kernel messages:

```bash
sudo journalctl -k --no-pager -n 100
```

Verify the CIFS version option. Modern servers should use SMB2 or SMB3, for
example:

```text
vers=3.1.1
```

### The automatic mount fails after reboot

Check the `/etc/fstab` entry:

```bash
grep ' /mnt/n9 ' /etc/fstab
```

Test it manually:

```bash
sudo mount /mnt/n9
```

Confirm that:

- The server is reachable.
- The credentials file exists.
- The credentials file has mode `0600`.
- The `_netdev` option is present.
- The client has Tailscale connected when using the Tailscale address.
- `nofail` is present if boot must continue when the server is unavailable.

### Files have unexpected ownership or permissions

Inspect the share:

```bash
sudo find /srv/files -maxdepth 2 -printf '%M %u %g %p\n'
```

Review the share settings:

```bash
sudo testparm -s
```

Confirm that `force user` is not configured. Review the `create mask`,
`directory mask`, `force group`, and `inherit permissions` settings.

Correct individual files or directories only after confirming the intended
ownership model.

### Samba logs show repeated authentication failures

Review recent service logs:

```bash
sudo journalctl -u smbd --no-pager -n 100
```

Check for:

- Incorrect saved client credentials.
- Disabled Samba account.
- Wrong username.
- Expired or changed password.
- Client attempts to use guest access.
- Access from an unexpected network.

Do not publish unsanitized logs because they may contain usernames, hostnames,
or client addresses.

## Evidence To Capture

Capture sanitized evidence showing:

- The installed Samba version.
- The `smbd` service status.
- The share directory ownership and permissions.
- The `fileshare` group structure without personal identifiers.
- A redacted `testparm -s` result.
- TCP port `445` listening.
- LAN-scoped UFW rules.
- Tailscale-scoped UFW rules.
- A successful local share test.
- A successful remote share test.
- A successful CIFS mount.
- The protected client credentials file permissions.
- A documented failed authentication or permission test.
- The backup and restore procedure for shared data.

Remove or replace:

- Samba passwords.
- Linux passwords.
- Usernames.
- Hostnames.
- Live LAN addresses.
- Live Tailscale addresses.
- Client addresses.
- Unsanitized Samba logs.
- Private credentials files.
- Personal filenames or document contents.

## Completion Criteria

This module is complete when:

- Samba is installed and enabled.
- `smbd` is active and listening.
- `/srv/files` has documented ownership and permissions.
- The dedicated `fileshare` group exists.
- Authorized Linux users are members of the group.
- Samba credentials are configured and enabled.
- Guest access is disabled.
- SMB1 is not required or enabled.
- The share configuration passes `testparm`.
- TCP port `445` is allowed from the LAN.
- TCP port `445` is allowed through `tailscale0`.
- Local clients can authenticate and access the share.
- Remote Tailscale clients can authenticate and access the share.
- Unauthorized access is denied.
- A Linux client can mount the share.
- Credentials are protected from other local users.
- Backup and restore requirements are documented.
- Sanitized evidence has been captured.

Next: [Gitea Git Service](08-gitea-git-service.md)
