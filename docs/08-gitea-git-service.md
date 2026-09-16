# Gitea Git Service

## Purpose

This module deploys Gitea as a private Git hosting service on the Ubuntu server.

It covers:

- Preparing persistent Gitea storage.
- Deploying Gitea with Docker Compose.
- Pinning a known Gitea image version.
- Configuring the web and Git SSH interfaces.
- Creating the initial administrator account.
- Adding SSH keys to Gitea.
- Testing repository creation, clone, commit, and push operations.
- Restricting access to the LAN and Tailscale interfaces.
- Backing up and updating Gitea.
- Diagnosing application, container, permission, and authentication failures.

Gitea is intended for private LAN and authorized Tailscale access. It is not
configured as a public internet service.

## Learning Objectives

After completing this module, an administrator should be able to:

- Explain how a containerized Git service uses persistent storage.
- Deploy an application with Docker Compose.
- Configure Gitea's web and SSH access.
- Distinguish host administration SSH from Gitea Git SSH.
- Use Gitea's web interface and Git command-line workflow.
- Restrict application access by network and interface.
- Validate container health and application logs.
- Back up and restore Gitea repositories and configuration.
- Troubleshoot Git authentication and service connectivity.

## Prerequisites

| Requirement                     | Purpose                    |
| ------------------------------- | -------------------------- |
| Ubuntu Server installed         | Gitea host                 |
| Working Docker Engine           | Container runtime          |
| Docker Compose plugin           | Service deployment         |
| Explicit `sudo` Docker access   | Administrative operations  |
| Working local SSH access        | Configuration and recovery |
| Working Tailscale access        | Optional remote Git access |
| Samba and Docker storage layout | `/srv` organization        |
| Available persistent storage    | Repositories and database  |
| Gitea image version selected    | Reproducible deployment    |

Complete these modules first:

- [Ubuntu Server Installation](02-install-ubuntu-server.md)
- [Linux Baseline, Access, and Firewall](03-linux-baseline-access-and-firewall.md)
- [Networking, DNS, and Router Configuration](04-networking-dns-and-router.md)
- [SSH and Tailscale](05-ssh-and-tailscale.md)
- [Docker Host](06-docker-host.md)
- [Samba File Service](07-samba-file-service.md)

## Service Design

The Gitea service uses Docker Compose with bind-mounted persistent data:

```text
Git client
    |
    +-- HTTP TCP 3000
    |
    +-- SSH TCP 222
            |
            v
      Ubuntu server
            |
            v
      Gitea container
            |
            +-- /srv/docker/gitea/data
                  |
                  +-- Repositories
                  +-- SQLite database
                  +-- Application configuration
                  +-- Gitea-managed SSH keys
```

The administration and Git SSH ports are intentionally different:

| Function                     | Port | Service                          |
| ---------------------------- | ---: | -------------------------------- |
| Ubuntu server administration |   22 | System OpenSSH and Tailscale SSH |
| Gitea web interface          | 3000 | Gitea HTTP service               |
| Gitea Git operations         |  222 | Gitea SSH service                |

This distinction prevents Git traffic from being confused with host
administration traffic.

## Design Decisions

| Decision                           | Reason                                                |
| ---------------------------------- | ----------------------------------------------------- |
| Gitea in Docker                    | Separates the Git service from the Ubuntu base system |
| Pinned image version               | Makes upgrades deliberate and reproducible            |
| SQLite database                    | Suitable for this single-server homelab               |
| Bind-mounted data                  | Keeps repositories and configuration persistent       |
| Root-owned Compose file            | Protects service configuration                        |
| No plaintext secrets in Compose    | Avoids publishing credentials                         |
| Web port 3000                      | Matches the planned service port                      |
| Git SSH port 222                   | Avoids conflict with host administration SSH          |
| LAN and Tailscale access only      | Prevents public Git service exposure                  |
| Registration disabled after setup  | Prevents uncontrolled account creation                |
| Regular backup and restore testing | Protects repositories and service configuration       |

SQLite is appropriate for this small lab. A larger deployment may require an
external database and additional application infrastructure.

## Step 1: Review the Host and Storage

Confirm that Docker is available:

```bash
sudo docker version
sudo docker compose version
```

Confirm that the Docker service is running:

```bash
sudo systemctl status docker --no-pager
```

Review available storage:

```bash
df -h /srv
lsblk
```

Confirm the Gitea parent directory exists:

```bash
sudo ls -ld /srv/docker/gitea
```

If it does not exist, create it with root ownership:

```bash
sudo install -d -o root -g root -m 0750 /srv/docker/gitea
```

Record the Linux user and group IDs that Gitea will use for its data:

```bash
id <SERVER_USER>
```

The selected UID and GID must match the values configured in the Compose file.

## Step 2: Prepare Gitea Data Directories

Create the persistent data directory using the selected Gitea UID and GID.

For a typical first administrative user with UID and GID `1000`:

```bash
sudo install -d -o 1000 -g 1000 -m 0750 \
  /srv/docker/gitea/data
```

Replace `1000` with the actual UID and GID when necessary.

Verify the result:

```bash
sudo ls -ld /srv/docker/gitea /srv/docker/gitea/data
```

The container must be able to write to the data directory. Do not solve
permission errors by making the directory world-writable.

If existing data is already present, inspect it before changing ownership:

```bash
sudo find /srv/docker/gitea/data -maxdepth 2 \
  -printf '%M %u %g %p\n'
```

Do not recursively change ownership of an existing Gitea installation without
first confirming the expected container UID and GID.

## Step 3: Select and Record the Gitea Image Version

Do not use the `latest` tag for this service. Select a specific supported Gitea
version and record it privately:

```text
Gitea image: gitea/gitea:<GITEA_VERSION>
Gitea UID: <GITEA_UID>
Gitea GID: <GITEA_GID>
```

A version tag makes upgrades intentional. An image digest can provide even
stronger reproducibility if it is recorded and maintained.

Before deployment, review the selected version's release notes and migration
requirements.

## Step 4: Create the Compose File

Create the root-owned Compose file:

```bash
sudoedit /srv/docker/gitea/docker-compose.yml
```

Use the following structure and replace the placeholders:

```yaml
services:
  gitea:
    image: gitea/gitea:<GITEA_VERSION>
    container_name: gitea
    restart: unless-stopped
    ports:
      - "3000:3000"
      - "222:22"
    environment:
      - USER_UID=<GITEA_UID>
      - USER_GID=<GITEA_GID>
    volumes:
      - /srv/docker/gitea/data:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
```

For a typical first administrative user, the environment may be:

```yaml
environment:
  - USER_UID=1000
  - USER_GID=1000
```

Use the actual values from `id <SERVER_USER>`.

The Compose file intentionally does not contain:

- An administrator password.
- A database password.
- A private key.
- An access token.
- A public registry credential.

Review the file permissions:

```bash
sudo ls -l /srv/docker/gitea/docker-compose.yml
```

The file should be owned by root and should not be world-writable.

## Step 5: Validate the Compose Configuration

Validate the Compose file before downloading or starting the service:

```bash
sudo docker compose \
  -f /srv/docker/gitea/docker-compose.yml config
```

The command should render a valid configuration without syntax errors.

Check the image reference:

```bash
sudo docker compose \
  -f /srv/docker/gitea/docker-compose.yml config --images
```

The output should show the selected version rather than `latest`.

If validation fails:

1. Read the reported line and field.
2. Check YAML indentation.
3. Check quotation marks and colons.
4. Confirm that all placeholders were replaced.
5. Run the validation command again.
6. Do not start the service until validation succeeds.

## Step 6: Pull and Start Gitea

Pull the selected image:

```bash
sudo docker compose \
  -f /srv/docker/gitea/docker-compose.yml pull
```

Record the image details:

```bash
sudo docker image inspect gitea/gitea:<GITEA_VERSION>
```

Start the service:

```bash
sudo docker compose \
  -f /srv/docker/gitea/docker-compose.yml up -d
```

Check the container:

```bash
sudo docker ps --filter name=gitea
```

The container should be running.

View recent logs:

```bash
sudo docker logs --tail=200 gitea
```

Gitea may take a short time to initialize its data directory on the first start.

## Step 7: Verify the Gitea Web Service

Check the published ports:

```bash
sudo docker port gitea
```

Expected mappings include:

```text
3000/tcp -> 0.0.0.0:3000
22/tcp -> 0.0.0.0:222
```

Check the server listener:

```bash
sudo ss -lntp | grep -E ':(3000|222)'
```

Test the web service locally:

```bash
curl -I http://127.0.0.1:3000
```

A response from Gitea confirms that the HTTP service is responding. It does not
confirm that remote clients can reach it.

From a local LAN client, open:

```text
http://<SERVER_LAN_IP>:3000
```

From an authorized Tailscale client, open:

```text
http://<SERVER_TAILSCALE_IP>:3000
```

Use a stable hostname instead of an IP address if the environment provides one.

## Step 8: Complete the Initial Gitea Setup

Open the Gitea setup page in a browser.

Use the following general settings:

| Setting         | Recommended value                              |
| --------------- | ---------------------------------------------- |
| Database type   | SQLite3                                        |
| Database path   | `/data/gitea/gitea.db`                         |
| Server domain   | `<GITEA_HOSTNAME>`                             |
| SSH server port | `222`                                          |
| Gitea base URL  | `http://<GITEA_HOSTNAME>:3000/`                |
| Registration    | Disable after creating the first administrator |
| Email service   | Leave disabled unless specifically required    |

The server domain and base URL should use a name or address that the intended
clients can resolve and reach. If local and remote clients require different
paths, use a documented hostname and DNS strategy rather than changing clone
URLs manually for every repository.

Create the first administrator account.

Do not use a real password in documentation or screenshots.

After installation, confirm that the administrator can:

- Sign in.
- Open the administration settings.
- Create a repository.
- View repository settings.
- Add an SSH key.

## Step 9: Restrict Account Registration

Open the Gitea administration settings and disable open registration unless
additional users are intentionally required.

Review:

- Registration status.
- Default repository visibility.
- Default organization permissions.
- Password requirements.
- Email requirements.
- Administrator accounts.
- Existing users and access tokens.

If registration is changed through a configuration file, validate and restart
the service afterward:

```bash
sudo docker compose \
  -f /srv/docker/gitea/docker-compose.yml restart
```

Confirm the setting through the web interface after the restart.

## Step 10: Configure the Firewall

Allow the Gitea web interface from the local LAN:

```bash
sudo ufw allow in on <SERVER_INTERFACE> \
  from <LAN_CIDR> to any port 3000 proto tcp
```

Allow the Gitea web interface through Tailscale:

```bash
sudo ufw allow in on tailscale0 \
  to any port 3000 proto tcp
```

Allow Gitea Git SSH from the local LAN:

```bash
sudo ufw allow in on <SERVER_INTERFACE> \
  from <LAN_CIDR> to any port 222 proto tcp
```

Allow Gitea Git SSH through Tailscale:

```bash
sudo ufw allow in on tailscale0 \
  to any port 222 proto tcp
```

Review the policy:

```bash
sudo ufw status verbose
sudo ufw status numbered
```

Do not open ports `3000` or `222` globally when access can be scoped to the LAN
and Tailscale interfaces.

## Docker and UFW Consideration

Docker may add its own NAT and forwarding rules for published ports. A published
port may not behave exactly like a normal host service under UFW.

Inspect the published ports:

```bash
sudo docker ps --format 'table {{.Names}}\t{{.Ports}}\t{{.Status}}'
```

Test from:

- The server itself.
- A local LAN client.
- An authorized Tailscale client.
- An unauthorized network, if a safe test environment is available.

If Docker-published ports bypass the intended UFW restrictions, use the
`DOCKER-USER` chain or a more restrictive Docker binding strategy. Any such
rules must be documented and made persistent across reboots.

Do not assume that a successful UFW status output proves that Docker-published
ports are correctly restricted.

## Step 11: Configure a Gitea SSH Key

On the Git client, generate an SSH key if one does not already exist:

```bash
ssh-keygen -t ed25519
```

Display the public key:

```bash
cat ~/.ssh/id_ed25519.pub
```

Copy only the public key.

In Gitea:

1. Open the user settings.
2. Select **SSH / GPG Keys**.
3. Select **Add Key**.
4. Provide a descriptive key name.
5. Paste the public key.
6. Save the key.

Never upload or publish the private key.

## Step 12: Test Gitea SSH Authentication

Test through the local LAN:

```bash
ssh -p 222 git@<SERVER_LAN_IP>
```

Test through Tailscale:

```bash
ssh -p 222 git@<SERVER_TAILSCALE_IP>
```

Gitea normally authenticates the key and refuses an interactive shell. A message
indicating that shell access is not supported can be an expected result.

This is different from host administration:

```text
Host administration:
ssh <SERVER_USER>@<SERVER_TAILSCALE_IP>

Gitea Git SSH:
ssh -p 222 git@<SERVER_TAILSCALE_IP>
```

The `git` account is managed by the Gitea service. It is not the Ubuntu
administrative user.

Test the port independently if needed:

```bash
nc -vz <SERVER_TAILSCALE_IP> 222
```

A successful port test confirms reachability but not key authentication.

## Step 13: Create and Test a Repository

Create a test repository in Gitea.

Clone it over HTTP:

```bash
git clone \
  http://<GITEA_HOSTNAME>:3000/<GITEA_USER>/<REPOSITORY>.git
```

Or clone it over Gitea SSH:

```bash
git clone \
  ssh://git@<GITEA_HOSTNAME>:222/<GITEA_USER>/<REPOSITORY>.git
```

Create a test change:

```bash
cd <REPOSITORY>
printf '%s\n' 'Gitea verification' > verification.txt
git commit -m "Verify Gitea repository access"
```

Push through SSH:

```bash
git push origin main
```

If the default branch is different, use the branch created by the repository.

Verify:

```bash
git status
git log --oneline -1
git remote -v
```

Confirm the commit appears in the Gitea web interface.

Test repository access through both:

- The local LAN address or hostname.
- The Tailscale address or hostname.

Remove the temporary verification repository or file after testing if it is not
part of the project evidence.

## Step 14: Configure Repository Access

For each repository, review:

- Repository visibility.
- Collaborators.
- Organization membership.
- Branch protection.
- Webhooks.
- Deploy keys.
- Personal access tokens.
- Repository actions or automation.
- Default branch.
- Release permissions.

Use the narrowest permissions required.

Personal access tokens should be created only when a Git client or integration
requires them. Store tokens in a protected credential manager and never place
them in repository URLs, Compose files, or documentation.

## Step 15: Verify Persistent Data

Inspect the Gitea data directory:

```bash
sudo find /srv/docker/gitea/data -maxdepth 3 \
  -printf '%M %u %g %p\n'
```

Confirm that the data directory is mounted into the container:

```bash
sudo docker inspect gitea \
  --format '{{json .Mounts}}'
```

Confirm that the repository and database paths exist inside the container:

```bash
sudo docker exec gitea ls -la /data
sudo docker exec gitea ls -la /data/gitea
```

Do not modify files inside the container when the same persistent files can be
managed through the bind-mounted host directory.

## Step 16: Test Restart and Reboot Recovery

Restart the Compose service:

```bash
sudo docker compose \
  -f /srv/docker/gitea/docker-compose.yml restart
```

Wait for the service to initialize, then verify:

```bash
sudo docker ps --filter name=gitea
sudo docker logs --tail=100 gitea
curl -I http://127.0.0.1:3000
```

Verify that the test repository and commit still exist.

For a full reboot test:

```bash
sudo reboot
```

After the server returns:

```bash
sudo systemctl status docker --no-pager
sudo docker ps --filter name=gitea
sudo docker logs --tail=100 gitea
```

Test the web interface and Git SSH access again.

A running container does not prove that the application is healthy. Confirm both
service availability and repository access.

## Step 17: Back Up Gitea

Back up the following items:

- `/srv/docker/gitea/data`
- `/srv/docker/gitea/docker-compose.yml`
- The selected Gitea version.
- Any external configuration used by the service.
- Required secrets stored outside the Compose file.
- Repository and administrator recovery information.

For a simple consistent filesystem backup, stop Gitea briefly:

```bash
sudo docker compose \
  -f /srv/docker/gitea/docker-compose.yml stop
```

Create an archive on a separate backup target:

```bash
sudo tar -C /srv/docker/gitea \
  -czf /path/to/backup/gitea-data-<DATE>.tar.gz data docker-compose.yml
```

Start Gitea again:

```bash
sudo docker compose \
  -f /srv/docker/gitea/docker-compose.yml start
```

Verify that it is available:

```bash
sudo docker ps --filter name=gitea
curl -I http://127.0.0.1:3000
```

The backup destination must not be located only inside the same server or disk.
A local archive does not protect against hardware failure.

## Step 18: Restore Gitea Data

Before restoring:

1. Stop the Gitea container.
2. Preserve the current data directory.
3. Confirm that the backup is readable.
4. Confirm the expected Gitea version.
5. Verify available disk space.
6. Record the current container state.

Stop the service:

```bash
sudo docker compose \
  -f /srv/docker/gitea/docker-compose.yml stop
```

Move the existing data directory aside:

```bash
sudo mv /srv/docker/gitea/data \
  /srv/docker/gitea/data-before-restore
```

Recreate the destination:

```bash
sudo install -d -o <GITEA_UID> -g <GITEA_GID> -m 0750 \
  /srv/docker/gitea/data
```

Extract the backup:

```bash
sudo tar -C /srv/docker/gitea \
  -xzf /path/to/backup/gitea-data-<DATE>.tar.gz data
```

Restore ownership if required:

```bash
sudo chown -R <GITEA_UID>:<GITEA_GID> \
  /srv/docker/gitea/data
```

Start the service:

```bash
sudo docker compose \
  -f /srv/docker/gitea/docker-compose.yml start
```

Verify:

```bash
sudo docker ps --filter name=gitea
sudo docker logs --tail=200 gitea
```

Confirm that the web interface, administrator account, repositories, and Git SSH
access work.

Do not delete `data-before-restore` until the restore has been validated.

## Step 19: Update Gitea

Before updating:

1. Review the Gitea release notes.
2. Confirm that a tested backup exists.
3. Check the current image version.
4. Review migration requirements.
5. Select the next specific image version.
6. Schedule a maintenance window.

Record the current version:

```bash
sudo docker inspect gitea \
  --format '{{.Config.Image}}'
```

Edit the root-owned Compose file:

```bash
sudoedit /srv/docker/gitea/docker-compose.yml
```

Update only the image version:

```yaml
image: gitea/gitea:<NEW_GITEA_VERSION>
```

Validate the file:

```bash
sudo docker compose \
  -f /srv/docker/gitea/docker-compose.yml config
```

Pull and recreate the service:

```bash
sudo docker compose \
  -f /srv/docker/gitea/docker-compose.yml pull

sudo docker compose \
  -f /srv/docker/gitea/docker-compose.yml up -d
```

Verify:

```bash
sudo docker ps --filter name=gitea
sudo docker logs --tail=200 gitea
curl -I http://127.0.0.1:3000
```

Test:

- Administrator login.
- Repository browsing.
- HTTP clone.
- SSH clone.
- Push and pull.
- Existing repository data.
- Tailscale access.

Do not remove the previous image until the update has been verified and a
rollback decision is no longer required.

## Security Considerations

- Use a specific Gitea image version instead of `latest`.
- Review image release notes before updates.
- Keep the Compose file root-owned.
- Do not store passwords or tokens in the Compose file.
- Disable open registration unless it is intentionally required.
- Use strong administrator and user passwords.
- Protect SSH private keys.
- Use narrow repository and organization permissions.
- Restrict ports `3000` and `222` to the LAN and Tailscale interfaces.
- Do not expose Gitea through public router port forwarding.
- Do not expose the Docker API socket.
- Do not grant users Docker group access solely to manage Gitea.
- Keep SQLite and repository data on persistent storage.
- Back up the application data to separate storage.
- Test restoring a repository and the database.
- Use HTTPS in a future reverse-proxy deployment if credentials must cross an
  untrusted network.
- Treat Tailscale access and Gitea authentication as separate security layers.

The current HTTP interface is acceptable for this controlled lab when access is
limited to trusted LAN and Tailscale paths. It should not be presented as a
public production deployment.

## Verification Checklist

| Check                 | Expected result                               |
| --------------------- | --------------------------------------------- |
| Gitea directory       | Persistent data directory exists              |
| Data ownership        | Directory uses the selected Gitea UID and GID |
| Image version         | Compose file uses a specific version          |
| Compose validation    | `docker compose config` succeeds              |
| Container startup     | Gitea container is running                    |
| Container logs        | No unresolved startup errors                  |
| Web listener          | TCP port 3000 is listening                    |
| Git SSH listener      | TCP port 222 is listening                     |
| Local web access      | Gitea opens through the LAN path              |
| Remote web access     | Gitea opens through Tailscale                 |
| LAN firewall          | Ports 3000 and 222 are LAN-scoped             |
| Tailscale firewall    | Ports 3000 and 222 are Tailscale-scoped       |
| Initial administrator | Administrator can sign in                     |
| Registration          | Open registration is disabled unless required |
| SSH key               | Gitea accepts the configured public key       |
| Git SSH               | Clone and push work through port 222          |
| HTTP Git              | Clone and push work if enabled                |
| Persistence           | Data survives container restart               |
| Reboot recovery       | Gitea returns after server reboot             |
| Backup                | Gitea data and configuration are backed up    |
| Restore               | A restore test has been completed             |
| Public exposure       | No router port forwarding exists              |

## Common Problems

### The container will not start

Check the container state:

```bash
sudo docker ps -a --filter name=gitea
```

Review logs:

```bash
sudo docker logs --tail=200 gitea
```

Validate the Compose file:

```bash
sudo docker compose \
  -f /srv/docker/gitea/docker-compose.yml config
```

Check the Docker service:

```bash
sudo systemctl status docker --no-pager
```

Common causes include invalid Compose syntax, an incorrect image version, port
conflicts, missing data directories, incorrect data ownership, insufficient disk
space, and unsupported image architecture.

### Gitea reports a permission error

Inspect the host directory:

```bash
sudo ls -ld /srv/docker/gitea/data
sudo find /srv/docker/gitea/data -maxdepth 2 \
  -printf '%M %u %g %p\n'
```

Inspect the configured UID and GID:

```bash
id <SERVER_USER>
```

Compare them with the Compose environment:

```yaml
USER_UID=<GITEA_UID> USER_GID=<GITEA_GID>
```

Correct ownership only after confirming the expected container user:

```bash
sudo chown -R <GITEA_UID>:<GITEA_GID> \
  /srv/docker/gitea/data
```

Restart and review the logs afterward.

### Port 3000 is already in use

Check the listener:

```bash
sudo ss -lntp | grep ':3000'
```

Check Docker-published ports:

```bash
sudo docker ps --format 'table {{.Names}}\t{{.Ports}}\t{{.Status}}'
```

Identify the service using the port before stopping or reconfiguring it.

Do not change Gitea's port without also updating the firewall rules, Gitea base
URL, client clone URLs, documentation, and validation tests.

### Port 222 is already in use

Check:

```bash
sudo ss -lntp | grep ':222'
```

Port `222` must not conflict with another service.

Host administration uses port `22`. Gitea Git SSH uses port `222`.

### The web interface works locally but not from the LAN

Check:

```bash
sudo ss -lntp | grep ':3000'
sudo ufw status verbose
sudo docker ps --format 'table {{.Names}}\t{{.Ports}}\t{{.Status}}'
```

Confirm that:

- The client uses the server's LAN address.
- TCP port 3000 is allowed from the LAN.
- Docker published the port.
- The client is connected to the intended network.
- No Docker forwarding rule bypasses or conflicts with the intended policy.

### The web interface works on the LAN but not through Tailscale

Check both devices:

```bash
tailscale status
tailscale ping <SERVER_TAILSCALE_IP>
```

Check the firewall:

```bash
sudo ufw status numbered
```

Confirm that TCP port 3000 is allowed on `tailscale0`.

Use the server's Tailscale address or a hostname that resolves through the
tailnet. A LAN address is not normally reachable from a remote client.

### SSH authentication to Gitea fails

Test the port:

```bash
nc -vz <SERVER_TAILSCALE_IP> 222
```

Test SSH with verbose output:

```bash
ssh -vvv -p 222 git@<SERVER_TAILSCALE_IP>
```

Check:

- The public key was added to the correct Gitea account.
- The client is using the intended private key.
- The client is using port `222`.
- The server address is correct.
- The Gitea container is running.
- The Tailscale or LAN firewall rule is present.

Do not add the key to the Ubuntu administrator's `authorized_keys` file when the
goal is Gitea Git authentication.

### Gitea accepts SSH but Git push fails

Check the Git remote:

```bash
git remote -v
```

The remote should use the Gitea SSH port:

```text
ssh://git@<GITEA_HOSTNAME>:222/<GITEA_USER>/<REPOSITORY>.git
```

Test repository access:

```bash
git ls-remote origin
```

Check the repository permissions in Gitea. The SSH key may authenticate
successfully while the Gitea account still lacks write access to the repository.

### The Git remote uses port 22 instead of 222

Inspect the remote:

```bash
git remote -v
```

Correct it:

```bash
git remote set-url origin \
  ssh://git@<GITEA_HOSTNAME>:222/<GITEA_USER>/<REPOSITORY>.git
```

Port `22` is reserved for Ubuntu host administration.

### Gitea generates unusable clone URLs

Review the Gitea server domain, SSH port, and base URL in the administrator
settings.

The configured values must match the address and port that clients use.

If local and remote clients need different addresses, provide a documented
hostname and DNS strategy rather than relying on users to edit every clone URL.

### The container is running but the application is unhealthy

Review logs:

```bash
sudo docker logs --tail=200 gitea
```

Check the HTTP response:

```bash
curl -I http://127.0.0.1:3000
```

Inspect the container:

```bash
sudo docker inspect gitea
```

Check data directory permissions, database path, available storage, image
version, configuration files, port mappings, and recent migrations.

### Data disappears after recreating the container

Confirm that the Compose file uses the persistent bind mount:

```yaml
volumes:
  - /srv/docker/gitea/data:/data
```

Inspect the mount:

```bash
sudo docker inspect gitea \
  --format '{{json .Mounts}}'
```

Do not store important data only in a container writable layer.

### Gitea loses data after an update

Stop further changes and preserve the current state:

```bash
sudo docker compose \
  -f /srv/docker/gitea/docker-compose.yml stop
```

Review the backup and restore procedure. Do not delete the current data
directory or previous image until the cause is understood.

Check the image version and migration notes before attempting another update.

### Docker-published ports are exposed beyond the intended networks

Review:

```bash
sudo docker ps --format 'table {{.Names}}\t{{.Ports}}\t{{.Status}}'
sudo ufw status verbose
```

Test from the LAN and Tailscale paths. If the published ports bypass the
intended UFW restrictions, implement and document `DOCKER-USER` filtering or
another host-level access control strategy.

Do not solve the problem by opening more ports globally.

## Evidence To Capture

Capture sanitized evidence showing:

- The selected Gitea version.
- The Compose configuration without secrets.
- Root ownership of the Compose file.
- Gitea data directory ownership and permissions.
- Successful Compose validation.
- Container status and sanitized logs.
- TCP ports 3000 and 222 listening.
- LAN-scoped UFW rules.
- Tailscale-scoped UFW rules.
- Successful web access.
- Successful Gitea SSH authentication.
- Successful repository clone, commit, and push.
- Successful restart and reboot recovery.
- Backup contents without credentials.
- A completed restore test.
- The absence of public router port forwarding.

## Completion Criteria

This module is complete when:

- Gitea storage directories exist with documented ownership.
- A specific Gitea image version is selected.
- The root-owned Compose file validates successfully.
- The Gitea container starts and remains running.
- The web service responds on TCP port 3000.
- Git SSH responds on TCP port 222.
- LAN firewall rules are scoped correctly.
- Tailscale firewall rules are scoped correctly.
- The initial administrator account is created.
- Open registration is disabled unless intentionally required.
- An SSH public key is added to Gitea.
- Git clone, commit, and push work through SSH.
- Repository data persists across container restarts.
- Gitea recovers after a server reboot.
- Gitea data and configuration are backed up.
- A restore test has been completed.
- No public router port forwarding is configured.
- Sanitized evidence has been captured.

Next: [Pi-hole DNS Service](09-pihole-dns-service.md)
