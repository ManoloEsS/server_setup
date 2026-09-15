# Pi-hole DNS Service

## Purpose

This module deploys Pi-hole as the network's DNS filtering service.

It covers:

- Preparing persistent Pi-hole storage.
- Releasing host port 53 safely.
- Deploying Pi-hole with Docker Compose.
- Using current Pi-hole v6 configuration variables.
- Protecting the web interface password.
- Allowing DNS from the LAN and Tailscale interfaces.
- Configuring the server's own resolver.
- Configuring router DHCP to distribute Pi-hole.
- Configuring Tailscale DNS for remote clients.
- Testing permitted and blocked DNS queries.
- Diagnosing DNS, container, resolver, and firewall failures.
- Backing up and restoring Pi-hole configuration.

Pi-hole is a critical network dependency. If it becomes unavailable, clients
may lose DNS resolution even when IP connectivity continues to work.

## Learning Objectives

After completing this module, an administrator should be able to:

- Explain the role of a DNS filtering service.
- Deploy Pi-hole with Docker Compose.
- Explain why port 53 must have one clear owner.
- Configure Pi-hole v6 through environment variables.
- Protect administrative credentials.
- Configure clients to use Pi-hole through DHCP and Tailscale DNS.
- Test DNS directly and through normal client resolver paths.
- Distinguish routing, resolver, firewall, and Pi-hole failures.
- Back up and restore Pi-hole configuration.
- Document the impact of a DNS service outage.

## Prerequisites

| Requirement | Purpose |
|---|---|
| Ubuntu Server installed | Pi-hole host |
| Working Docker Engine | Container runtime |
| Docker Compose plugin | Service deployment |
| Explicit `sudo` Docker access | Administrative operations |
| Stable server LAN address | Local client DNS |
| Working Tailscale access | Remote DNS clients |
| Working upstream internet access | Forwarded DNS queries |
| Completed resolver module | `systemd-resolved` planning |
| Available persistent storage | Pi-hole configuration and databases |

Complete these modules first:

- [Ubuntu Server Installation](02-install-ubuntu-server.md)
- [Linux Baseline, Access, and Firewall](03-linux-baseline-access-and-firewall.md)
- [Networking, DNS, and Router Configuration](04-networking-dns-and-router.md)
- [SSH and Tailscale](05-ssh-and-tailscale.md)
- [Docker Host](06-docker-host.md)
- [Samba File Service](07-samba-file-service.md)
- [Gitea Git Service](08-gitea-git-service.md)

## Service Design

The Pi-hole container uses host networking so it can bind directly to the
server's DNS and web ports.

```text
Local client
    |
    | DNS request to <SERVER_LAN_IP>:53
    v
Pi-hole on the Ubuntu server
    |
    | Forward permitted queries
    v
Configured upstream DNS providers

Remote Tailscale client
    |
    | DNS request to <SERVER_TAILSCALE_IP>:53
    v
Pi-hole on the Ubuntu server
    |
    | Forward permitted queries
    v
Configured upstream DNS providers
```

The intended service ports are:

| Function | Protocol and port | Intended access |
|---|---|---|
| DNS | UDP and TCP 53 | LAN and authorized Tailscale clients |
| Pi-hole web interface | TCP 80 | Administrative LAN and Tailscale clients |

Host networking means the Compose file does not use ordinary `ports:` mappings.
Pi-hole binds directly to the host interfaces.

## DNS Dependency Sequence

The configuration must be applied in this order:

```text
Check current port 53 owner
    |
    v
Disable systemd-resolved stub listener
    |
    v
Keep temporary uplink DNS available
    |
    v
Start Pi-hole
    |
    v
Verify Pi-hole owns port 53
    |
    v
Point the server resolver to 127.0.0.1
    |
    v
Configure router DHCP to distribute Pi-hole
    |
    v
Configure Tailscale DNS
    |
    v
Test local and remote clients
```

Do not point the server's resolver to `127.0.0.1` before Pi-hole is running and
answering queries. Doing so can create a DNS bootstrap failure.

## Design Decisions

| Decision | Reason |
|---|---|
| Pi-hole in Docker | Separates DNS filtering from the Ubuntu base system |
| Host networking | Allows direct access to host ports 53 and 80 |
| Pinned image version | Makes updates deliberate and reproducible |
| Pi-hole v6 environment variables | Uses the current supported configuration model |
| Root-owned Compose file | Protects service configuration |
| Compose secret for web password | Avoids storing the password in YAML |
| Pi-hole as client DNS | Centralizes filtering policy |
| No public client DNS fallback | Prevents clients from bypassing filtering |
| Tailscale DNS for remote clients | Extends filtering outside the home LAN |
| Router DHCP for LAN DNS | Avoids manually configuring every local client |
| No Pi-hole DHCP service | Keeps DHCP responsibility with the primary router |

The router remains the DHCP authority. Pi-hole provides DNS filtering and
upstream query forwarding.

## Step 1: Review Current DNS and Port 53 Ownership

Check the current resolver state:

```bash
resolvectl status
```

Check the resolver file:

```bash
ls -l /etc/resolv.conf
```

Check all listeners on port 53:

```bash
sudo ss -lntup | grep ':53'
```

Before Pi-hole is deployed, the listener may be `systemd-resolved` or another
resolver supplied by the base installation.

Record the current state privately:

```text
Current DNS listener: <CURRENT_DNS_LISTENER>
Current uplink resolver: <UPSTREAM_DNS>
Server interface: <SERVER_INTERFACE>
Server LAN address: <SERVER_LAN_IP>
Server Tailscale address: <SERVER_TAILSCALE_IP>
```

There must be one clear owner for host port 53 after Pi-hole is deployed.

## Step 2: Preserve Temporary DNS During the Change

Pi-hole must be able to start before the server points its own resolver at
Pi-hole.

Confirm that the server can currently resolve the Docker registry:

```bash
resolvectl query registry-1.docker.io
```

If DNS is currently provided by DHCP, preserve the existing uplink DNS while
releasing only the local systemd-resolved stub listener.

Create the resolver drop-in directory:

```bash
sudo install -d -m 0755 /etc/systemd/resolved.conf.d
```

Create the drop-in:

```bash
sudoedit /etc/systemd/resolved.conf.d/pihole.conf
```

Use:

```ini
[Resolve]
DNSStubListener=no
```

Do not add `DNS=127.0.0.1` yet. Pi-hole is not running at this stage.

Point `/etc/resolv.conf` to the non-stub resolver file:

```bash
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
```

Restart `systemd-resolved`:

```bash
sudo systemctl restart systemd-resolved
```

Confirm that the stub listener no longer owns port 53:

```bash
sudo ss -lntup | grep ':53'
```

Confirm that temporary DNS still works:

```bash
resolvectl query registry-1.docker.io
```

If DNS fails at this point, restore the previous resolver state or use the
documented DNS recovery procedure before continuing.

## Step 3: Prepare Pi-hole Storage

Create the Pi-hole directory:

```bash
sudo install -d -o root -g root -m 0750 /srv/docker/pihole
```

Create the persistent configuration directory:

```bash
sudo install -d -o root -g root -m 0750 \
  /srv/docker/pihole/etc-pihole
```

For a fresh Pi-hole v6 deployment, the `/etc/pihole` directory is the primary
persistent directory.

If migrating from an older Pi-hole installation that uses custom dnsmasq
configuration, also create:

```bash
sudo install -d -o root -g root -m 0750 \
  /srv/docker/pihole/etc-dnsmasq.d
```

Review the directories:

```bash
sudo ls -ld /srv/docker/pihole \
  /srv/docker/pihole/etc-pihole \
  /srv/docker/pihole/etc-dnsmasq.d
```

Do not recursively change ownership of existing Pi-hole data without first
checking the version and expected container permissions.

## Step 4: Create the Pi-hole Web Password Secret

Create a root-owned password file:

```bash
sudo install -o root -g root -m 0600 \
  /dev/null /srv/docker/pihole/pihole_webpassword
```

Edit it with `sudoedit`:

```bash
sudoedit /srv/docker/pihole/pihole_webpassword
```

Store only the Pi-hole web interface password in the file.

Do not publish:

- The password.
- The password file.
- Screenshots of the password.
- The password in the Compose file.
- The password in shell history.

Verify the file permissions:

```bash
sudo ls -l /srv/docker/pihole/pihole_webpassword
```

The file should be readable only by root.

## Step 5: Select and Record the Pi-hole Image Version

Do not use the `latest` tag for this service. Select a supported Pi-hole
container version and record it privately:

```text
Pi-hole image: pihole/pihole:<PIHOLE_VERSION>
Timezone: <TIMEZONE>
Primary upstream DNS: <UPSTREAM_DNS_1>
Secondary upstream DNS: <UPSTREAM_DNS_2>
```

Review the Pi-hole release notes before selecting an upgrade.

This module uses Pi-hole v6-style variables, including:

- `FTLCONF_webserver_api_password`
- `FTLCONF_dns_upstreams`
- `FTLCONF_dns_listeningMode`

The password is supplied through `WEBPASSWORD_FILE` and a Compose secret.

## Step 6: Create the Compose File

Create the root-owned Compose file:

```bash
sudoedit /srv/docker/pihole/docker-compose.yml
```

Use this structure and replace the placeholders:

```yaml
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:<PIHOLE_VERSION>
    network_mode: host
    environment:
      TZ: "<TIMEZONE>"
      WEBPASSWORD_FILE: /run/secrets/pihole_webpassword
      FTLCONF_dns_upstreams: "<UPSTREAM_DNS_1>;<UPSTREAM_DNS_2>"
      FTLCONF_dns_listeningMode: "ALL"
    volumes:
      - /srv/docker/pihole/etc-pihole:/etc/pihole
    secrets:
      - pihole_webpassword
    restart: unless-stopped

secrets:
  pihole_webpassword:
    file: ./pihole_webpassword
```

Example upstream values may be:

```yaml
      FTLCONF_dns_upstreams: "1.1.1.1;8.8.8.8"
```

Replace them with the upstream DNS providers selected for the environment.

The `FTLCONF_dns_listeningMode` setting is intentionally `ALL` because
Tailscale clients are outside the local LAN subnet. This setting must be
verified after startup.

Do not add `ports:` when using `network_mode: host`. The two networking modes
must not be combined.

Do not add `NET_ADMIN` unless Pi-hole is later configured to provide DHCP or
requires another documented network capability.

Review the Compose file permissions:

```bash
sudo ls -l /srv/docker/pihole/docker-compose.yml
```

The Compose file and password secret should be root-owned.

## Step 7: Validate the Compose Configuration

Validate the Compose file:

```bash
sudo docker compose \
  -f /srv/docker/pihole/docker-compose.yml config
```

The command should complete without YAML or Compose errors.

Check the image reference:

```bash
sudo docker compose \
  -f /srv/docker/pihole/docker-compose.yml config --images
```

Confirm that the output contains a specific Pi-hole version and not `latest`.

Confirm that the secret file exists:

```bash
sudo ls -l /srv/docker/pihole/pihole_webpassword
```

Do not publish the rendered configuration if it reveals environment values or
secret paths that should remain private.

## Step 8: Pull and Start Pi-hole

Pull the selected image:

```bash
sudo docker compose \
  -f /srv/docker/pihole/docker-compose.yml pull
```

Start the container:

```bash
sudo docker compose \
  -f /srv/docker/pihole/docker-compose.yml up -d
```

Check the container:

```bash
sudo docker ps --filter name=pihole
```

View recent logs:

```bash
sudo docker logs --tail=200 pihole
```

Check the container health state:

```bash
sudo docker inspect \
  --format '{{.State.Health.Status}}' pihole
```

The container should be running and should not report an unresolved health
failure.

## Step 9: Verify Pi-hole Owns Port 53

Check TCP and UDP port 53:

```bash
sudo ss -lntup | grep ':53'
```

The expected DNS listener should be Pi-hole's DNS process, not the
`systemd-resolved` stub.

Check the web listener:

```bash
sudo ss -lntp | grep ':80'
```

Host networking means that `docker ps` may not display ordinary port mappings.
The host listener output is the authoritative check.

If `systemd-resolved` still owns port 53:

```bash
sudo systemctl status systemd-resolved --no-pager
sudo journalctl -u systemd-resolved --no-pager -n 100
```

Confirm that the drop-in contains:

```ini
DNSStubListener=no
```

Restart the resolver before restarting Pi-hole:

```bash
sudo systemctl restart systemd-resolved

sudo docker compose \
  -f /srv/docker/pihole/docker-compose.yml restart
```

Do not delete the Pi-hole data directory to solve a port conflict.

## Step 10: Verify Pi-hole Listening Mode

Inspect the effective Pi-hole configuration:

```bash
sudo docker exec pihole \
  grep -E 'listeningMode|dnsmasq' /etc/pihole/pihole.toml
```

The DNS listening mode should be `ALL`.

The environment variable is the intended source of truth:

```yaml
FTLCONF_dns_listeningMode: "ALL"
```

If the mode remains `LOCAL`:

1. Confirm the Compose file uses the current v6 variable.
2. Confirm the container was recreated after the change.
3. Check the container logs.
4. Check the persisted `pihole.toml`.
5. Recreate the container only after preserving the data directory.

Recreate the service after a configuration change:

```bash
sudo docker compose \
  -f /srv/docker/pihole/docker-compose.yml up -d
```

Do not use a broad `sed` replacement on the persistent configuration without
first confirming the Pi-hole version and configuration format.

## Step 11: Configure the Server Resolver

After Pi-hole is running and answering on port 53, update the resolver
configuration.

Edit the drop-in:

```bash
sudoedit /etc/systemd/resolved.conf.d/pihole.conf
```

Use:

```ini
[Resolve]
DNS=127.0.0.1
DNSStubListener=no
```

Restart `systemd-resolved`:

```bash
sudo systemctl restart systemd-resolved
```

Configure the active server interface to use local Pi-hole:

```bash
sudo resolvectl dns <SERVER_INTERFACE> 127.0.0.1
sudo resolvectl domain <SERVER_INTERFACE> "~."
```

Verify the resolver:

```bash
resolvectl status
resolvectl query ubuntu.com
```

Verify that the resolver file still uses the non-stub path:

```bash
ls -l /etc/resolv.conf
```

Expected target:

```text
/run/systemd/resolve/resolv.conf
```

Verify port 53 ownership again:

```bash
sudo ss -lntup | grep ':53'
```

The intended host DNS path is:

```text
Local application
    |
    v
systemd-resolved
    |
    | 127.0.0.1
    v
Pi-hole
    |
    v
Configured upstream DNS
```

## Step 12: Configure Tailscale DNS Behavior on the Server

The N9 host must continue using its local Pi-hole resolver rather than having
Tailscale replace it with a tailnet resolver.

Run:

```bash
sudo tailscale set --accept-dns=false
```

Verify the setting through the Tailscale client status or admin console.

This setting applies to the N9 itself. It does not disable Tailscale DNS for
other client devices.

## Step 13: Configure the Firewall

Allow DNS from the local LAN:

```bash
sudo ufw allow in on <SERVER_INTERFACE> \
  from <LAN_CIDR> to any port 53 proto udp

sudo ufw allow in on <SERVER_INTERFACE> \
  from <LAN_CIDR> to any port 53 proto tcp
```

Allow DNS from Tailscale clients:

```bash
sudo ufw allow in on tailscale0 \
  to any port 53 proto udp

sudo ufw allow in on tailscale0 \
  to any port 53 proto tcp
```

Allow the Pi-hole web interface from the local LAN:

```bash
sudo ufw allow in on <SERVER_INTERFACE> \
  from <LAN_CIDR> to any port 80 proto tcp
```

Allow the Pi-hole web interface through Tailscale:

```bash
sudo ufw allow in on tailscale0 \
  to any port 80 proto tcp
```

Review the policy:

```bash
sudo ufw status verbose
sudo ufw status numbered
```

Do not open DNS or the Pi-hole web interface globally when access can be
restricted to known interfaces and networks.

## Step 14: Test Pi-hole Directly

Query Pi-hole directly from the server:

```bash
dig @127.0.0.1 ubuntu.com
```

If `dig` is not installed:

```bash
sudo apt install dnsutils
```

Check a domain expected to be blocked:

```bash
dig @127.0.0.1 doubleclick.net
```

The result should contain the Pi-hole blocked response, commonly `0.0.0.0`
or another locally configured response.

Check a permitted domain:

```bash
dig @127.0.0.1 ubuntu.com
```

A permitted domain should return a valid address.

Check Pi-hole query activity:

```bash
sudo docker logs --tail=200 pihole
```

The Pi-hole web interface is available at:

```text
http://<SERVER_LAN_IP>/admin
```

Use the Tailscale address for authorized remote administration:

```text
http://<SERVER_TAILSCALE_IP>/admin
```

## Step 15: Configure Router DHCP DNS

Configure the primary router's LAN DHCP settings to distribute:

```text
Primary DNS: <SERVER_LAN_IP>
```

Do not configure a public secondary DNS server if every client query must pass
through Pi-hole. A client may select the secondary resolver and bypass filtering.

The router may use a separate upstream DNS configuration for its own operation.
The important client setting is the DNS server distributed through LAN DHCP.

After applying the router setting:

1. Renew a client DHCP lease.
2. Confirm the client received `<SERVER_LAN_IP>` as DNS.
3. Query a permitted domain.
4. Query a blocked test domain.
5. Confirm the request appears in Pi-hole.

On a Linux client:

```bash
resolvectl status
resolvectl query ubuntu.com
resolvectl query doubleclick.net
```

If available, query directly:

```bash
dig @<SERVER_LAN_IP> ubuntu.com
dig @<SERVER_LAN_IP> doubleclick.net
```

The client should use the server's LAN address for DNS.

## Step 16: Configure Tailscale DNS for Remote Clients

After local Pi-hole DNS has been verified, configure the tailnet DNS policy.

In the Tailscale admin console:

1. Open **DNS**.
2. Add a custom nameserver.
3. Enter `<SERVER_TAILSCALE_IP>`.
4. Enable **Override local DNS**.
5. Do not add a public fallback nameserver.
6. Save the policy.

The reason for avoiding a public fallback is that remote clients may send queries
to the public resolver instead of Pi-hole, bypassing filtering.

On each client:

- Connect to the intended tailnet.
- Enable **Use Tailscale DNS**.
- Disable competing VPN or private relay DNS features during testing.

On Linux:

```bash
sudo tailscale set --accept-dns=true
```

The N9 server itself is the exception and should retain:

```bash
sudo tailscale set --accept-dns=false
```

## Step 17: Test Remote DNS

From an authorized Tailscale client, query Pi-hole directly:

```bash
dig @<SERVER_TAILSCALE_IP> ubuntu.com
dig @<SERVER_TAILSCALE_IP> doubleclick.net
```

Then test the normal client resolver:

```bash
resolvectl query ubuntu.com
resolvectl query doubleclick.net
```

The blocked query should return Pi-hole's blocked response.

Confirm that Pi-hole received the request in the web dashboard or logs.

If the direct query works but the normal resolver query does not, investigate
the client's Tailscale DNS settings rather than the Pi-hole container first.

## Step 18: Test Reboot Recovery

Before rebooting, record the current state:

```bash
sudo docker ps
sudo ufw status verbose
resolvectl status
sudo ss -lntup | grep ':53'
```

Reboot the server:

```bash
sudo reboot
```

After reconnecting, verify:

```bash
sudo systemctl status docker --no-pager
sudo systemctl status systemd-resolved --no-pager
sudo docker ps --filter name=pihole
sudo docker inspect \
  --format '{{.State.Health.Status}}' pihole
```

Check port 53:

```bash
sudo ss -lntup | grep ':53'
```

Test local DNS:

```bash
resolvectl query ubuntu.com
resolvectl query doubleclick.net
```

Test the Pi-hole web interface and a remote Tailscale DNS client again.

The server must not depend on manually starting Pi-hole after every reboot.

## Step 19: Back Up Pi-hole

Back up:

- `/srv/docker/pihole/etc-pihole`
- `/srv/docker/pihole/docker-compose.yml`
- `/srv/docker/pihole/pihole_webpassword`
- Resolver drop-in configuration.
- UFW rules.
- Router DHCP DNS settings.
- Tailscale DNS policy.
- The selected Pi-hole image version.

Protect the password backup as a secret.

For a consistent filesystem backup, stop Pi-hole briefly:

```bash
sudo docker compose \
  -f /srv/docker/pihole/docker-compose.yml stop
```

Create an archive on a separate backup target:

```bash
sudo tar -C /srv/docker/pihole \
  -czf /path/to/backup/pihole-<DATE>.tar.gz \
  etc-pihole docker-compose.yml pihole_webpassword
```

Start Pi-hole:

```bash
sudo docker compose \
  -f /srv/docker/pihole/docker-compose.yml start
```

Verify:

```bash
sudo docker ps --filter name=pihole
sudo docker inspect \
  --format '{{.State.Health.Status}}' pihole
```

The backup destination must not be only the same server or disk.

## Step 20: Restore Pi-hole

Before restoring:

1. Confirm that the backup is readable.
2. Preserve the current Pi-hole data.
3. Confirm the expected image version.
4. Verify available storage.
5. Keep an alternate DNS resolver available.
6. Plan for temporary client DNS disruption.

Stop the service:

```bash
sudo docker compose \
  -f /srv/docker/pihole/docker-compose.yml stop
```

Preserve the current data:

```bash
sudo mv /srv/docker/pihole/etc-pihole \
  /srv/docker/pihole/etc-pihole-before-restore
```

Recreate the destination:

```bash
sudo install -d -o root -g root -m 0750 \
  /srv/docker/pihole/etc-pihole
```

Extract the backup:

```bash
sudo tar -C /srv/docker/pihole \
  -xzf /path/to/backup/pihole-<DATE>.tar.gz etc-pihole
```

Restore the secret file if required:

```bash
sudo chmod 0600 /srv/docker/pihole/pihole_webpassword
sudo chown root:root /srv/docker/pihole/pihole_webpassword
```

Start Pi-hole:

```bash
sudo docker compose \
  -f /srv/docker/pihole/docker-compose.yml start
```

Verify:

```bash
sudo docker ps --filter name=pihole
sudo docker logs --tail=200 pihole
sudo docker inspect \
  --format '{{.State.Health.Status}}' pihole
```

Test DNS resolution, blocked responses, the web interface, and remote Tailscale
DNS before deleting `etc-pihole-before-restore`.

## Step 21: Update Pi-hole

Before updating:

1. Review Pi-hole release notes.
2. Confirm a tested backup exists.
3. Record the current image version.
4. Review v5-to-v6 or other migration notes if applicable.
5. Select a specific new image version.
6. Schedule a maintenance window.

Record the current image:

```bash
sudo docker inspect pihole \
  --format '{{.Config.Image}}'
```

Edit the root-owned Compose file:

```bash
sudoedit /srv/docker/pihole/docker-compose.yml
```

Update only the image version:

```yaml
image: pihole/pihole:<NEW_PIHOLE_VERSION>
```

Validate:

```bash
sudo docker compose \
  -f /srv/docker/pihole/docker-compose.yml config
```

Pull and recreate:

```bash
sudo docker compose \
  -f /srv/docker/pihole/docker-compose.yml pull

sudo docker compose \
  -f /srv/docker/pihole/docker-compose.yml up -d
```

Verify:

```bash
sudo docker ps --filter name=pihole
sudo docker logs --tail=200 pihole
sudo docker inspect \
  --format '{{.State.Health.Status}}' pihole
sudo ss -lntup | grep ':53'
```

Test:

- Server DNS.
- Local client DNS.
- Blocked domains.
- Pi-hole web access.
- Tailscale DNS.
- Router client behavior.

Do not remove the previous image until the update is verified.

## DNS Troubleshooting Sequence

When a client cannot resolve a domain, test in this order:

1. Confirm the client has a valid IP address.
2. Confirm the client has a default route.
3. Test the default gateway.
4. Test the server's LAN or Tailscale address.
5. Confirm the client received the intended DNS server.
6. Confirm the Pi-hole container is running.
7. Confirm port 53 ownership.
8. Query Pi-hole directly.
9. Check UFW UDP and TCP rules.
10. Check Pi-hole logs.
11. Check upstream DNS resolution.
12. Check for VPN, encrypted DNS, or private relay bypass.

Useful commands include:

```bash
ip -br address
ip route
resolvectl status
resolvectl query ubuntu.com
sudo docker ps --filter name=pihole
sudo docker logs --tail=200 pihole
sudo ss -lntup | grep ':53'
sudo ufw status verbose
```

Test both UDP and TCP DNS when investigating unusual failures.

## Security Considerations

- Use a specific Pi-hole image version instead of `latest`.
- Keep the Compose file root-owned.
- Protect the web password with a root-owned mode `0600` secret file.
- Do not publish the Pi-hole password or secret file.
- Do not expose DNS or the web interface to the public internet.
- Do not create router port forwarding for ports 53 or 80.
- Restrict DNS to the LAN and Tailscale interfaces.
- Keep guest or unauthenticated administration disabled.
- Do not add unnecessary container capabilities.
- Do not enable Pi-hole DHCP when the router remains the DHCP authority.
- Use `FTLCONF_dns_listeningMode: "ALL"` for Tailscale clients.
- Do not add public client DNS fallbacks when filtering must be enforced.
- Review VPN, encrypted DNS, and private relay bypass behavior.
- Keep Pi-hole and Ubuntu packages updated.
- Back up Pi-hole configuration separately from the server.
- Test restoring Pi-hole data.
- Maintain an alternate DNS recovery procedure.
- Treat Pi-hole as a single point of failure for client name resolution.

## Verification Checklist

| Check | Expected result |
|---|---|
| Pi-hole directory | Persistent directory exists |
| Password secret | Root-owned and mode `0600` |
| Image version | Compose file uses a specific version |
| Compose validation | `docker compose config` succeeds |
| Container startup | Pi-hole container is running |
| Container health | Health status is healthy |
| Port 53 ownership | Pi-hole owns TCP and UDP DNS listeners |
| Port 80 ownership | Pi-hole web service is listening |
| Resolver stub | `systemd-resolved` stub is not using port 53 |
| Server DNS | Server resolves through local Pi-hole |
| Listening mode | Pi-hole accepts LAN and Tailscale queries |
| LAN firewall | DNS is allowed from the LAN |
| Tailscale firewall | DNS is allowed on `tailscale0` |
| LAN client DNS | DHCP distributes `<SERVER_LAN_IP>` |
| LAN permitted query | Normal domains resolve |
| LAN blocked query | Test domains are blocked |
| Remote DNS | Tailscale client reaches Pi-hole |
| Remote blocked query | Remote test domains are blocked |
| Web administration | Admin UI is reachable only on intended paths |
| Reboot recovery | Pi-hole returns after reboot |
| Backup | Configuration and secret are backed up securely |
| Restore | Restore test has been completed |
| Public exposure | No router port forwarding exists |

## Evidence To Capture

Capture sanitized evidence showing:

- The selected Pi-hole version.
- The Compose configuration without secrets.
- Root ownership of the Compose file.
- Protected password file permissions without its contents.
- Container status and health.
- Pi-hole logs with identifying information removed.
- TCP and UDP port 53 ownership.
- The `systemd-resolved` stub configuration.
- The server resolver path.
- LAN and Tailscale UFW rules.
- A LAN client receiving the Pi-hole DNS address.
- A successful permitted DNS query.
- A blocked DNS query.
- A successful remote Tailscale DNS query.
- Router DHCP DNS configuration without credentials.
- Successful reboot recovery.
- Backup and restore verification.
- The absence of public DNS and web port forwarding.

Remove or replace:

- Pi-hole passwords.
- API tokens.
- Live IP addresses.
- Tailscale device identifiers.
- Router credentials.
- Hostnames.
- Client names.
- Unsanitized logs.
- Personal DNS query history.
- Private backup paths.

## Completion Criteria

This module is complete when:

- Pi-hole is deployed with a pinned image version.
- The web password is stored outside the Compose YAML.
- The Compose file validates successfully.
- Pi-hole is healthy and persistent.
- Pi-hole owns host port 53.
- `systemd-resolved` does not conflict with Pi-hole.
- The server resolves through local Pi-hole.
- Pi-hole accepts authorized LAN and Tailscale queries.
- UFW restricts DNS and web access appropriately.
- The router distributes Pi-hole through DHCP.
- Local clients resolve permitted domains.
- Local clients receive blocked responses for filtered domains.
- Remote Tailscale clients use Pi-hole DNS.
- Remote blocked queries are verified.
- Pi-hole recovers after reboot.
- Configuration and credentials are backed up securely.
- A restore test has been completed.
- No public router port forwarding is configured.
- Sanitized evidence has been captured.

Next: [Validation and Acceptance Tests](10-validation-and-acceptance-tests.md)
