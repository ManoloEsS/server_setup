# Troubleshooting Playbooks

## Purpose

This module provides repeatable troubleshooting procedures for the self-hosted
server.

Each playbook follows the same support workflow:

```text
Identify the symptom
    |
    v
Define the affected scope
    |
    v
Check the lowest dependency layer
    |
    v
Test one hypothesis
    |
    v
Apply the smallest safe change
    |
    v
Verify the result
    |
    v
Document the incident
```

The goal is to show a disciplined support process rather than applying random
commands until a service happens to work.

## Learning Objectives

After completing this module, an administrator should be able to:

- Classify failures by network, host, service, authentication, or data layer.
- Use evidence to narrow the problem before changing configuration.
- Compare local LAN and remote Tailscale behavior.
- Diagnose DNS failures without confusing them with routing failures.
- Diagnose Docker, Samba, Gitea, and Pi-hole failures.
- Preserve service data during recovery.
- Apply safe rollback and restore procedures.
- Record symptoms, hypotheses, tests, fixes, and verification results.
- Identify when a problem exceeds the scope of the homelab.

## Prerequisites

Complete these modules first:

- [Ubuntu Server Installation](02-install-ubuntu-server.md)
- [Linux Baseline, Access, and Firewall](03-linux-baseline-access-and-firewall.md)
- [Networking, DNS, and Router Configuration](04-networking-dns-and-router.md)
- [SSH and Tailscale](05-ssh-and-tailscale.md)
- [Docker Host](06-docker-host.md)
- [Samba File Service](07-samba-file-service.md)
- [Gitea Git Service](08-gitea-git-service.md)
- [Pi-hole DNS Service](09-pihole-dns-service.md)
- [Validation and Acceptance Tests](10-validation-and-acceptance-tests.md)
- [Operations, Backups, and Recovery](11-operations-backups-and-recovery.md)

## Safety Rules

Before changing anything:

- Preserve an active administrative session.
- Open a second recovery session when possible.
- Record the current service and network state.
- Confirm the affected service and scope.
- Check whether a backup is required.
- Avoid changing multiple layers at the same time.
- Validate configuration before restarting a service.
- Avoid destructive commands until data impact is understood.

Do not run these commands as a first troubleshooting step:

```bash
sudo ufw reset
sudo docker system prune
sudo rm -rf /srv/docker
sudo rm -rf /srv/files
sudo rm -rf /etc/samba
```

These commands can destroy configuration, data, or access controls.

## Initial Triage

When a problem is reported, collect:

```text
Incident ID: <INCIDENT_ID>
Reported time: <TIME>
Affected user or device: <SANITIZED_IDENTIFIER>
Affected service: <SERVICE>
Access path: LAN / Tailscale / server-local
Observed symptom: <SYMPTOM>
Recent change: <CHANGE_OR_NONE>
Business or lab impact: <IMPACT>
```

Run the safe baseline checks on the server:

```bash
date -Is
uptime
ip -br address
ip route
resolvectl status
df -h
sudo systemctl --failed
sudo ufw status verbose
sudo ss -lntup
sudo docker ps
```

Determine whether the issue affects:

- One client or multiple clients.
- LAN clients or Tailscale clients.
- One service or all services.
- Name resolution or direct IP connectivity.
- Authentication or basic reachability.
- Current data or only new operations.

## Diagnostic Layers

Use the following order when the failure is unclear:

| Layer          | Questions                                  | Useful checks                |
| -------------- | ------------------------------------------ | ---------------------------- |
| Physical       | Is the server powered and connected?       | Console, link lights, cables |
| Network        | Does the server have an address and route? | `ip`, `ping`, `ip route`     |
| Resolver       | Can names be resolved?                     | `resolvectl`, `dig`          |
| Firewall       | Is the traffic permitted?                  | `ufw`, `ss`                  |
| Host service   | Is the service active and listening?       | `systemctl`, `ss`            |
| Container      | Is the container running and healthy?      | `docker ps`, logs            |
| Authentication | Are credentials and permissions valid?     | Service-specific checks      |
| Data           | Is persistent data present and readable?   | `ls`, backups, mounts        |

Do not begin at the application layer when the server has no route or DNS.

## Playbook 1: Server Is Unreachable

### Symptom

The server does not respond to SSH, service requests, or Tailscale checks.

### Scope

Determine whether the server is powered off, unreachable on the LAN, or
unreachable only through Tailscale.

### Diagnostics

Check the server locally if console access is available:

```bash
ip -br address
ip route
sudo systemctl status ssh --no-pager
sudo systemctl status tailscaled --no-pager
sudo ufw status verbose
```

From a local client:

```bash
ping -c 3 <SERVER_LAN_IP>
nc -vz <SERVER_LAN_IP> 22
```

From a Tailscale client:

```bash
tailscale status
tailscale ping <SERVER_TAILSCALE_IP>
nc -vz <SERVER_TAILSCALE_IP> 22
```

### Likely causes

- Server power or hardware failure.
- Ethernet cable or router port failure.
- DHCP reservation failure.
- Changed server address.
- SSH service stopped.
- Tailscale service stopped.
- UFW rule blocking the path.
- Client connected to the wrong network.

### Recovery

Use the lowest available recovery path:

1. Check power and physical connections.
2. Use the local console if available.
3. Inspect the current LAN address.
4. Restore the DHCP reservation if required.
5. Start or restart only the affected service.
6. Confirm firewall rules before testing again.

If the LAN address changed, use Tailscale if it remains available rather than
changing every service configuration immediately.

### Verification

Repeat:

```bash
ssh <SERVER_USER>@<SERVER_LAN_IP>
ssh <SERVER_USER>@<SERVER_TAILSCALE_IP>
```

Record the original address, root cause, recovery action, and final address.

## Playbook 2: Local LAN Connectivity Failure

### Symptom

The server or a local client cannot reach another device on the LAN.

### Diagnostics

On the affected device:

```bash
ip -br address
ip route
```

Test the gateway:

```bash
ping -c 3 <ROUTER_LAN_IP>
ip route get <ROUTER_LAN_IP>
```

Test the server:

```bash
ping -c 3 <SERVER_LAN_IP>
ip route get <SERVER_LAN_IP>
```

Check the server interface:

```bash
sudo ethtool <SERVER_INTERFACE>
```

### Likely causes

- Disconnected cable.
- Failed switch or router port.
- Client connected to a different network.
- Incorrect subnet or gateway.
- DHCP failure.
- Server interface down.
- Firewall blocking the service rather than the network.

### Recovery

Correct the physical or network configuration first. Do not change service
configuration while the client cannot reach the server's LAN address.

If the interface is down, use the appropriate network-management method for the
installation. Confirm the change survives a reconnect or reboot.

### Verification

Confirm:

```bash
ip -br address
ip route
ping -c 3 <ROUTER_LAN_IP>
ping -c 3 <SERVER_LAN_IP>
```

Then test the affected application port.

## Playbook 3: DHCP Reservation or Server Address Changed

### Symptom

The server received a different LAN address after reboot or reconnect.

### Diagnostics

On the server:

```bash
ip -br address
ip route
ip link show <SERVER_INTERFACE>
```

On the router, review the DHCP lease and reservation using the server's MAC
address.

### Likely causes

- Reservation was not saved.
- Reservation uses the wrong MAC address.
- Reservation is assigned to a different interface.
- Server is connected to the wrong router.
- Router DHCP service restarted with different settings.

### Recovery

Correct the reservation at the router. Prefer a DHCP reservation over manually
hardcoding a static address on the server for this project.

Renew the lease or reboot during a maintenance window. Do not update Pi-hole,
Samba, or Gitea configuration until the address is stable.

### Verification

```bash
ip -br address
ip route
```

Confirm the router shows the same reserved address and that clients can reach
the server.

## Playbook 4: SSH Access Fails

### Symptom

SSH is refused, times out, or authentication fails.

### Diagnostics

Identify the access path:

```text
LAN SSH: port 22 at <SERVER_LAN_IP>
Tailscale SSH: port 22 at <SERVER_TAILSCALE_IP>
Gitea Git SSH: port 222
```

On the server:

```bash
sudo systemctl status ssh --no-pager
sudo ss -lntp | grep ':22'
sudo ufw status numbered
```

From a client:

```bash
ssh -vvv <SERVER_USER>@<SERVER_LAN_IP>
```

For Tailscale:

```bash
tailscale status
tailscale ping <SERVER_TAILSCALE_IP>
ssh -vvv <SERVER_USER>@<SERVER_TAILSCALE_IP>
```

### Likely causes

- Wrong address or username.
- SSH service stopped.
- UFW rule missing.
- Tailscale policy failure.
- Tailscale SSH not enabled.
- Password or key authentication failure.
- Host fingerprint mismatch.
- Client using the wrong private key.

### Recovery

Keep an active recovery session. Add or correct only the required firewall or
SSH setting:

```bash
sudo ufw allow in from <LAN_CIDR> to any port 22 proto tcp
sudo ufw allow in on tailscale0 to any port 22 proto tcp
```

Validate SSH configuration before restarting it:

```bash
sudo sshd -t
sudo systemctl restart ssh
```

Do not disable password authentication until key login has been tested from a
separate client.

### Verification

Test the original path and the alternate recovery path. Record whether the
failure was network, firewall, service, policy, or authentication related.

## Playbook 5: Tailscale Remote Access Fails

### Symptom

The server is reachable locally but not through Tailscale.

### Diagnostics

On the server:

```bash
sudo systemctl status tailscaled --no-pager
tailscale status
tailscale ip -4
ip link show tailscale0
sudo ufw status numbered
```

On the client:

```bash
tailscale status
tailscale netcheck
tailscale ping <SERVER_TAILSCALE_IP>
```

### Likely causes

- Tailscale service stopped.
- Device is not authorized.
- Client and server use different tailnets.
- Tailscale access policy denies the connection.
- Tailscale SSH is not enabled on the server.
- UFW blocks `tailscale0`.
- The client is using the LAN address remotely.
- Relay or upstream network restrictions.

### Recovery

Restart the service only from a local or alternate administrative path:

```bash
sudo systemctl restart tailscaled
```

Confirm Tailscale SSH is enabled:

```bash
sudo tailscale set --ssh
```

Review the tailnet policy before changing it. Add only the required UFW rule:

```bash
sudo ufw allow in on tailscale0 to any port 22 proto tcp
```

### Verification

```bash
tailscale status
tailscale ping <SERVER_TAILSCALE_IP>
ssh <SERVER_USER>@<SERVER_TAILSCALE_IP>
```

## Playbook 6: DNS Resolution Fails

### Symptom

The server or clients can reach IP addresses but cannot resolve domain names.

### Diagnostics

Start with the route:

```bash
ip route
ping -c 3 <ROUTER_LAN_IP>
```

Check resolver state:

```bash
resolvectl status
ls -l /etc/resolv.conf
```

Query the server resolver:

```bash
resolvectl query ubuntu.com
```

Query Pi-hole directly:

```bash
dig @127.0.0.1 ubuntu.com
```

Check Pi-hole and port 53:

```bash
sudo docker ps --filter name=pihole
sudo docker inspect \
  --format '{{.State.Health.Status}}' pihole
sudo ss -lntup | grep ':53'
sudo docker logs --tail=200 pihole
```

### Likely causes

- No default route.
- Resolver file points to the wrong target.
- `systemd-resolved` stub conflict.
- Pi-hole container stopped or unhealthy.
- Pi-hole cannot reach upstream DNS.
- UFW blocks TCP or UDP port 53.
- Router distributes the wrong DNS server.
- Client has stale DHCP or resolver state.
- VPN, encrypted DNS, or private relay bypasses Pi-hole.

### Recovery

Restore the lowest failed layer first:

1. Correct the route if necessary.
2. Confirm one owner for port 53.
3. Restore Pi-hole health.
4. Verify direct Pi-hole queries.
5. Restore the server resolver configuration.
6. Renew client DHCP leases.
7. Restore Tailscale DNS settings.

Use a public resolver only as a temporary diagnostic or emergency workaround:

```bash
dig @1.1.1.1 ubuntu.com
```

Do not make a public fallback the permanent client DNS configuration when
filtering is required.

### Verification

```bash
dig @127.0.0.1 ubuntu.com
resolvectl query ubuntu.com
resolvectl query doubleclick.net
```

Repeat the tests from a LAN client and a Tailscale client.

## Playbook 7: Pi-hole Will Not Start

### Symptom

The Pi-hole container exits, remains unhealthy, or cannot bind to DNS port 53.

### Diagnostics

```bash
sudo docker ps -a --filter name=pihole
sudo docker logs --tail=200 pihole
sudo docker inspect \
  --format '{{.State.Status}} {{.State.Health.Status}}' pihole
sudo ss -lntup | grep ':53'
sudo systemctl status systemd-resolved --no-pager
```

Validate Compose:

```bash
sudo docker compose \
  -f /srv/docker/pihole/docker-compose.yml config
```

### Likely causes

- `systemd-resolved` still owns port 53.
- Invalid Pi-hole v6 environment variable.
- Missing password secret.
- Incorrect Compose file permissions.
- Invalid image version.
- Corrupt persisted configuration.
- Upstream DNS or network failure.

### Recovery

Confirm the resolver drop-in contains:

```ini
[Resolve]
DNSStubListener=no
```

Restart the resolver:

```bash
sudo systemctl restart systemd-resolved
```

Confirm that port 53 is available, then recreate Pi-hole:

```bash
sudo docker compose \
  -f /srv/docker/pihole/docker-compose.yml up -d
```

Do not delete `/srv/docker/pihole/etc-pihole` before checking backups and logs.

### Verification

```bash
sudo docker inspect \
  --format '{{.State.Health.Status}}' pihole
sudo ss -lntup | grep ':53'
dig @127.0.0.1 ubuntu.com
```

## Playbook 8: Pi-hole Is Not Blocking Queries

### Symptom

Normal domains resolve, but known blocked domains return real addresses.

### Diagnostics

Check the client's DNS server:

```bash
resolvectl status
```

Query directly through Pi-hole:

```bash
dig @<SERVER_LAN_IP> doubleclick.net
dig @<SERVER_TAILSCALE_IP> doubleclick.net
```

Check the listening mode:

```bash
sudo docker exec pihole \
  grep -E 'listeningMode' /etc/pihole/pihole.toml
```

### Likely causes

- Client is using a public DNS server.
- Router DHCP setting was not renewed.
- Tailscale **Override local DNS** is disabled.
- Client **Use Tailscale DNS** is disabled.
- Pi-hole listening mode is `LOCAL` instead of `ALL`.
- VPN or encrypted DNS bypasses the configured resolver.
- Client has cached DNS information.

### Recovery

Confirm the Pi-hole Compose setting:

```yaml
FTLCONF_dns_listeningMode: "ALL"
```

Recreate the container if the setting changed:

```bash
sudo docker compose \
  -f /srv/docker/pihole/docker-compose.yml up -d
```

Renew the client's network lease and confirm its resolver state.

### Verification

Test direct and normal resolution. Confirm the query appears in Pi-hole and that
the client is not using a competing DNS path.

## Playbook 9: Docker Service or Container Fails

### Symptom

Docker commands fail, containers stop, or a container repeatedly restarts.

### Diagnostics

```bash
sudo systemctl status docker --no-pager
sudo journalctl -u docker --no-pager -n 100
sudo docker ps -a
sudo docker system df
df -h
```

For a specific service:

```bash
sudo docker logs --tail=200 <CONTAINER_NAME>
sudo docker inspect <CONTAINER_NAME>
```

### Likely causes

- Docker daemon stopped.
- Full filesystem.
- Invalid Compose configuration.
- Port conflict.
- Data-directory permission problem.
- Image architecture mismatch.
- Missing environment variable or secret.
- Corrupt container configuration.

### Recovery

Check the cause before restarting Docker. If the daemon is stopped and the cause
is understood:

```bash
sudo systemctl restart docker
```

Validate a service before recreating it:

```bash
sudo docker compose \
  -f /srv/docker/<SERVICE>/docker-compose.yml config
```

Do not run `docker system prune` as a general repair command. It can remove
unused images, networks, and volumes.

### Verification

```bash
sudo docker ps
sudo docker logs --tail=100 <CONTAINER_NAME>
```

Repeat the service-specific acceptance tests.

## Playbook 10: Samba Share Is Inaccessible

### Symptom

The share cannot be found, authentication fails, or access is denied.

### Diagnostics

On the server:

```bash
sudo systemctl status smbd --no-pager
sudo testparm -s
sudo ss -lntp | grep ':445'
sudo ufw status numbered
sudo ls -ld /srv/files
```

Check the account and group:

```bash
sudo pdbedit -L
id <SERVER_USER>
getent group fileshare
```

From a client:

```bash
nc -vz <SERVER_LAN_IP> 445
smbclient //<SERVER_LAN_IP>/files -U <SERVER_USER>
```

### Likely causes

- `smbd` stopped.
- Invalid `smb.conf` syntax.
- TCP port 445 blocked.
- User is not in `fileshare`.
- Samba account is disabled.
- Wrong Samba password.
- Linux filesystem permissions deny access.
- Client uses stale credentials.
- Client is using the wrong server address.

### Recovery

Validate before restarting:

```bash
sudo testparm
```

Correct the specific failed layer. Reset a Samba password only when the account
and authorization are confirmed:

```bash
sudo smbpasswd <SERVER_USER>
```

Do not use `force user` or world-writable permissions as a quick fix.

### Verification

Test listing, creating, reading, and deleting a non-sensitive file. Repeat
through both LAN and Tailscale if remote access is required.

## Playbook 11: Gitea Web Interface Is Unavailable

### Symptom

The Gitea page does not load or returns a connection error.

### Diagnostics

```bash
sudo docker ps -a --filter name=gitea
sudo docker logs --tail=200 gitea
sudo docker inspect gitea
sudo ss -lntp | grep ':3000'
sudo ufw status numbered
curl -I http://127.0.0.1:3000
```

From a LAN client:

```bash
nc -vz <SERVER_LAN_IP> 3000
```

From a Tailscale client:

```bash
nc -vz <SERVER_TAILSCALE_IP> 3000
```

### Likely causes

- Gitea container stopped.
- Invalid image version.
- Data-directory permission problem.
- Port 3000 conflict.
- UFW or Docker forwarding issue.
- Database or migration error.
- Client uses an unreachable hostname.

### Recovery

Validate the Compose file:

```bash
sudo docker compose \
  -f /srv/docker/gitea/docker-compose.yml config
```

Inspect data permissions before changing them. Recreate the service only after
the cause is understood:

```bash
sudo docker compose \
  -f /srv/docker/gitea/docker-compose.yml up -d
```

Do not remove the data directory to repair a web-interface failure.

### Verification

```bash
curl -I http://127.0.0.1:3000
```

Test web access, administrator login, and repository visibility from the
intended client paths.

## Playbook 12: Gitea Git SSH Fails

### Symptom

The Gitea web interface works, but Git clone or push over SSH fails.

### Diagnostics

Check the port:

```bash
sudo ss -lntp | grep ':222'
nc -vz <SERVER_TAILSCALE_IP> 222
```

Test with verbose SSH output:

```bash
ssh -vvv -p 222 git@<SERVER_TAILSCALE_IP>
```

Check the Git remote:

```bash
git remote -v
git ls-remote origin
```

### Likely causes

- Remote uses port 22 instead of 222.
- Public key was added to the wrong Gitea account.
- Client uses the wrong private key.
- Repository permissions deny the operation.
- Port 222 is blocked.
- Gitea container is not running.
- Gitea SSH configuration contains the wrong external port.

### Recovery

Correct the remote when necessary:

```bash
git remote set-url origin \
  ssh://git@<GITEA_HOSTNAME>:222/<GITEA_USER>/<REPOSITORY>.git
```

Confirm the public key in Gitea. Do not copy it to the Ubuntu administrator's
`authorized_keys` file for this purpose.

If Gitea's generated SSH keys are stale, use the documented Gitea key
regeneration procedure only after preserving the data directory.

### Verification

```bash
ssh -p 222 git@<SERVER_TAILSCALE_IP>
git ls-remote origin
git push origin main
```

## Playbook 13: Firewall or Port Access Failure

### Symptom

A service works locally but is refused or times out from a client.

### Diagnostics

On the server:

```bash
sudo ss -lntup
sudo ufw status verbose
sudo ufw status numbered
```

From the client:

```bash
nc -vz <SERVER_ADDRESS> <PORT>
```

Compare the local and remote paths:

```text
Server-local test: 127.0.0.1:<PORT>
LAN test: <SERVER_LAN_IP>:<PORT>
Tailscale test: <SERVER_TAILSCALE_IP>:<PORT>
```

### Likely causes

- Service is bound only to localhost.
- Service is not listening.
- UFW source or interface is incorrect.
- Client uses the wrong address.
- Docker-published port bypasses ordinary UFW assumptions.
- Router or client network isolation.
- Tailscale policy denies the connection.

### Recovery

Add only the required, correctly scoped rule:

```bash
sudo ufw allow in on <SERVER_INTERFACE> \
  from <LAN_CIDR> to any port <PORT> proto tcp
```

For a Tailscale client:

```bash
sudo ufw allow in on tailscale0 \
  to any port <PORT> proto tcp
```

Review Docker's published ports and `DOCKER-USER` rules separately. Do not open
the port globally as a first response.

### Verification

Repeat local, LAN, Tailscale, and unauthorized-network tests where safe.

## Playbook 14: Disk Space or Storage Failure

### Symptom

Updates fail, containers stop, files cannot be created, or services report
database and filesystem errors.

### Diagnostics

```bash
df -h
df -i
lsblk
sudo docker system df
sudo du -sh /var/lib/docker /srv/docker /srv/files
sudo journalctl -p warning..alert --since "24 hours ago" --no-pager
```

### Likely causes

- Shared files consumed available capacity.
- Docker images or layers accumulated.
- Logs grew unexpectedly.
- Backup archive was written to the wrong filesystem.
- Inodes are exhausted.
- Filesystem or disk failure.

### Recovery

First protect data and stop writes if the filesystem is critically full.

Review large files before deleting anything:

```bash
sudo du -xhd1 /srv
sudo du -xhd1 /var/lib/docker
```

Remove only identified temporary data. Do not delete application volumes or
shared files without a verified backup.

Do not run `docker system prune` until its impact is understood.

Check disk health using the hardware-appropriate tools if a device failure is
suspected. Escalate before continuing if filesystem corruption is indicated.

### Verification

Confirm free space, service health, container stability, and the ability to
create a controlled test file.

## Playbook 15: Service Fails After Reboot

### Symptom

A service worked before reboot but is unavailable afterward.

### Diagnostics

```bash
sudo systemctl --failed
sudo systemctl status docker --no-pager
sudo systemctl status smbd --no-pager
sudo systemctl status tailscaled --no-pager
sudo systemctl status systemd-resolved --no-pager
sudo docker ps -a
sudo ss -lntup
```

Check dependencies:

```bash
ip -br address
ip route
resolvectl status
df -h
```

### Likely causes

- Service is not enabled.
- Docker started before a required filesystem was available.
- Tailscale interface was not ready.
- Port 53 resolver conflict returned.
- Container restart policy is missing.
- DHCP address changed.
- Data directory is unavailable.

### Recovery

Restore the lowest failed dependency. Start only the affected service after
checking its configuration and logs.

For a Compose service:

```bash
sudo docker compose \
  -f /srv/docker/<SERVICE>/docker-compose.yml up -d
```

If the issue repeats, investigate startup ordering and enablement rather than
manually starting the service after every reboot.

### Verification

Repeat the reboot acceptance tests from
[Validation and Acceptance Tests](10-validation-and-acceptance-tests.md).

## Playbook 16: Backup or Restore Failure

### Symptom

A backup cannot be created, an archive is unreadable, or restored data does not
work.

### Diagnostics

Confirm the backup target:

```bash
findmnt <BACKUP_TARGET>
df -h <BACKUP_TARGET>
sudo ls -lh <BACKUP_TARGET>/server-setup
```

Check an archive:

```bash
sudo tar -tzf <BACKUP_ARCHIVE>
```

Check source and destination sizes:

```bash
sudo du -sh <SOURCE>
sudo du -sh <BACKUP_DESTINATION>
```

### Likely causes

- Backup target is not mounted.
- Backup target is full.
- Permissions deny access.
- `--delete` was used with the wrong destination.
- Archive was interrupted.
- Required configuration or secret was omitted.
- Restored files have incorrect ownership.
- Restored application version is incompatible.

### Recovery

Preserve the current data before attempting another restore. Use a temporary
restore directory when possible.

Do not overwrite a working service until the backup contents and target paths
have been confirmed.

Restore service data using the procedures in
[Operations, Backups, and Recovery](11-operations-backups-and-recovery.md).

### Verification

Confirm the restored service, data, permissions, credentials, and client access.
Record what was restored and whether the original data was preserved.

## Playbook 17: Controlled Incident Drill

Use this drill to practice the support workflow without risking network-wide DNS
service.

Stop Gitea during a maintenance window:

```bash
sudo docker compose \
  -f /srv/docker/gitea/docker-compose.yml stop
```

Record the failure:

```bash
sudo docker ps --filter name=gitea
curl -I http://127.0.0.1:3000
```

Diagnose the symptom using the Gitea playbook. Restore the service:

```bash
sudo docker compose \
  -f /srv/docker/gitea/docker-compose.yml start
```

Verify:

```bash
sudo docker ps --filter name=gitea
curl -I http://127.0.0.1:3000
git ls-remote \
  ssh://git@<GITEA_HOSTNAME>:222/<GITEA_USER>/<REPOSITORY>.git
```

Record the symptom, hypothesis, diagnostic path, recovery action, and result.

Do not use Pi-hole as the first failure-drill target because it can interrupt
DNS for every client.

## Escalation Boundaries

Escalate or stop when:

- Hardware failure is suspected.
- Filesystem corruption is reported.
- Backups are missing or unreadable.
- Data may have been deleted or overwritten.
- Credentials or private keys may have been exposed.
- Router or ISP configuration is outside the available access.
- A service requires production-grade availability.
- Recovery steps could destroy the only remaining copy of data.
- The cause cannot be isolated without risky changes.

Preserve logs, current state, and backups before escalation.

## Incident Record Template

```text
Incident ID: <INCIDENT_ID>
Start time: <TIME>
End time: <TIME>
Reporter: <ROLE_OR_ALIAS>
Affecting: <SERVICE_OR_SCOPE>
Access path: LAN / Tailscale / server-local
Observed symptom: <SYMPTOM>
User impact: <IMPACT>
Recent change: <CHANGE_OR_NONE>

Initial hypothesis:
<HYPOTHESIS>

Diagnostic commands and results:
<TESTS_AND_RESULTS>

Root cause:
<CAUSE>

Recovery action:
<ACTION>

Verification:
<RESULT>

Follow-up action:
<ACTION_OR_NONE>

Evidence reference:
<SANITIZED_REFERENCE>
```

## Evidence To Capture

Capture sanitized evidence showing:

- The reported symptom.
- The affected scope.
- The diagnostic sequence.
- The hypothesis tested.
- The relevant service and network state.
- The recovery action.
- The verification result.
- The final incident record.

Remove or replace:

- Passwords.
- Private keys.
- Access tokens.
- Personal usernames.
- Email addresses.
- Live IP addresses.
- Tailscale device IDs.
- Host fingerprints.
- Private repository names.
- Personal filenames.
- Full DNS query history.
- Unsanitized logs.

## Completion Criteria

This module is complete when:

- A standard troubleshooting workflow is documented.
- Initial triage can identify affected scope.
- Network, resolver, firewall, host, container, authentication, and data layers
  are distinguished.
- SSH and Tailscale failures have playbooks.
- DNS and Pi-hole failures have playbooks.
- Docker failures have a playbook.
- Samba failures have a playbook.
- Gitea web and Git SSH failures have playbooks.
- Storage and backup failures have playbooks.
- Reboot failures have a playbook.
- A controlled incident drill has been documented.
- Escalation boundaries are defined.
- Incident records can be completed without exposing secrets.
- Sanitized troubleshooting evidence has been captured.

Next: [Optional Home-Lab Extensions](13-optional-home-lab-extensions.md)
