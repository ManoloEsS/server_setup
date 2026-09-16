# Operations, Backups, And Recovery

## Purpose

This module documents the recurring operational procedures for the self-hosted
server.

It covers:

- Routine maintenance.
- Package and container updates.
- Service health checks.
- Disk and log reviews.
- Backup planning and execution.
- Restore procedures.
- Recovery priorities.
- Planned maintenance and rollback.
- Operational evidence and change records.

The server is a single-server homelab. Backups reduce the impact of failure but
do not provide high availability or automatic failover.

## Learning Objectives

After completing this module, an administrator should be able to:

- Perform routine server maintenance safely.
- Review system, service, storage, and container health.
- Back up application and user data to separate storage.
- Protect backups that contain credentials or private data.
- Restore an individual file.
- Restore Gitea and Pi-hole data.
- Recover services after a disk, configuration, or update failure.
- Document maintenance, incidents, and recovery results.
- Explain the limitations of the recovery design.

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

## Operational Boundaries

This project intentionally does not provide:

- High availability.
- Automatic failover.
- Redundant server hardware.
- Redundant network connectivity.
- Enterprise monitoring.
- A production service-level agreement.
- Guaranteed recovery from every hardware failure.

The server has a single primary storage location and may have a single network
interface. A disk, power, motherboard, or router failure can affect multiple
services at once.

## Service Recovery Priorities

Use this order when multiple services are unavailable:

```text
1. Physical power and hardware
2. Network interface and LAN address
3. SSH administration
4. Docker daemon
5. Pi-hole DNS
6. Samba file service
7. Gitea Git service
8. Remote Tailscale access and DNS
```

The actual order may change depending on the incident. Restore the management
path first so that later service work can be performed safely.

## Maintenance Principles

- Keep a current backup before updates that may affect service behavior.
- Make one logical change at a time.
- Validate configuration before restarting services.
- Keep an active recovery session during remote changes.
- Record the previous image or package version before updating.
- Verify the service after every update.
- Do not delete old backups or images until recovery is confirmed.
- Use maintenance windows for reboots and service interruptions.
- Document failures as operational evidence.

## Routine Maintenance Schedule

### Per-session checks

Run during normal administrative sessions:

```bash
date -Is
uptime
df -h
sudo systemctl --failed
sudo docker ps
sudo ufw status verbose
```

Review:

- Unexpected failed services.
- Low disk space.
- Repeatedly restarting containers.
- Unexpected firewall changes.
- Unexplained system load.

### Weekly checks

Run at least weekly:

```bash
sudo apt update
apt list --upgradable
sudo docker system df
sudo journalctl -p warning..alert --since "7 days ago" --no-pager
```

Review:

- Available security updates.
- Filesystem capacity.
- Docker image and container storage.
- Warning and error messages.
- Backup completion.

### Monthly checks

Run at least monthly:

```bash
sudo systemctl status ssh --no-pager
sudo systemctl status tailscaled --no-pager
sudo systemctl status docker --no-pager
sudo systemctl status smbd --no-pager
sudo systemctl status systemd-resolved --no-pager
sudo docker ps
sudo ss -lntup
```

Also:

- Review local users and groups.
- Review Samba accounts.
- Review Gitea accounts and tokens.
- Review Tailscale devices and access policy.
- Review UFW rules.
- Test a backup restore.
- Review disk and hardware limitations.
- Confirm that sensitive evidence has not entered the repository.

## Step 1: Record the Current State

Before planned maintenance, record:

```bash
date -Is
hostnamectl
uname -r
ip -br address
ip route
resolvectl status
df -h
sudo ufw status verbose
sudo ss -lntup
sudo docker ps
sudo docker images
```

Record service versions:

```bash
docker --version
sudo docker compose version
smbd --version
```

Record the configured application images:

```bash
sudo docker inspect gitea \
  --format '{{.Name}} {{.Config.Image}}'

sudo docker inspect pihole \
  --format '{{.Name}} {{.Config.Image}}'
```

Save only sanitized output as evidence.

Do not publish passwords, private keys, access tokens, live Tailscale device
identifiers, personal usernames, full DNS query history, or unsanitized logs.

## Step 2: Update Ubuntu Packages

Confirm that a working administrative path exists:

- Current SSH session.
- Second SSH session.
- Local console or other documented recovery path.

Update package metadata:

```bash
sudo apt update
```

Review available upgrades:

```bash
apt list --upgradable
```

Apply upgrades during a planned maintenance window:

```bash
sudo apt upgrade
```

Remove unused packages only after review:

```bash
sudo apt autoremove
```

Reboot if required:

```bash
sudo reboot
```

After the update, verify:

```bash
uname -r
sudo systemctl --failed
sudo systemctl status ssh --no-pager
sudo systemctl status docker --no-pager
sudo systemctl status smbd --no-pager
sudo docker ps
resolvectl query ubuntu.com
```

Do not use an unverified DNS workaround without documenting why it was needed.
The server should normally resolve through local Pi-hole after Pi-hole is
healthy.

## Step 3: Update Docker Images

Before updating an application image:

1. Review the image release notes.
2. Confirm a current backup exists.
3. Record the current image version.
4. Review compatibility and migration requirements.
5. Select a specific new image version.
6. Schedule a maintenance window.

Review current images:

```bash
sudo docker images
sudo docker image ls
```

Update Gitea:

```bash
sudoedit /srv/docker/gitea/docker-compose.yml
```

Change only the selected image version:

```yaml
image: gitea/gitea:<NEW_GITEA_VERSION>
```

Validate and update:

```bash
sudo docker compose \
  -f /srv/docker/gitea/docker-compose.yml config

sudo docker compose \
  -f /srv/docker/gitea/docker-compose.yml pull

sudo docker compose \
  -f /srv/docker/gitea/docker-compose.yml up -d
```

Update Pi-hole:

```bash
sudoedit /srv/docker/pihole/docker-compose.yml
```

Change only the selected image version:

```yaml
image: pihole/pihole:<NEW_PIHOLE_VERSION>
```

Validate and update:

```bash
sudo docker compose \
  -f /srv/docker/pihole/docker-compose.yml config

sudo docker compose \
  -f /srv/docker/pihole/docker-compose.yml pull

sudo docker compose \
  -f /srv/docker/pihole/docker-compose.yml up -d
```

Verify both services:

```bash
sudo docker ps
sudo docker logs --tail=100 gitea
sudo docker logs --tail=100 pihole
sudo docker inspect \
  --format '{{.State.Health.Status}}' pihole
sudo ss -lntup | grep -E ':(53|80|222|3000)'
```

Repeat the service tests:

- Pi-hole permitted DNS query.
- Pi-hole blocked DNS query.
- Gitea web login.
- Gitea Git SSH.
- Repository clone and push.
- LAN access.
- Tailscale access.

Do not remove the previous image until the update has been verified:

```bash
sudo docker image ls
```

## Step 4: Review Logs

Review Docker service logs:

```bash
sudo journalctl -u docker --since "24 hours ago" --no-pager
```

Review Samba logs:

```bash
sudo journalctl -u smbd --since "24 hours ago" --no-pager
```

Review resolver logs:

```bash
sudo journalctl -u systemd-resolved \
  --since "24 hours ago" --no-pager
```

Review Tailscale logs:

```bash
sudo journalctl -u tailscaled \
  --since "24 hours ago" --no-pager
```

Review application logs:

```bash
sudo docker logs --tail=200 gitea
sudo docker logs --tail=200 pihole
```

When reviewing logs, look for repeated restarts, authentication failures,
permission errors, port conflicts, database errors, DNS forwarding failures,
disk errors, and network failures.

Do not publish logs without removing usernames, addresses, device IDs, queries,
tokens, and other identifying information.

## Step 5: Prepare the Backup Target

Backups must be stored on a separate storage target.

Possible targets include:

- External disk.
- Separate computer.
- Encrypted network storage.
- Encrypted cloud backup.
- Another physical location.

Define the backup target privately:

```text
Backup target: <BACKUP_TARGET>
Backup filesystem: <BACKUP_FILESYSTEM>
Encryption: <ENCRYPTION_METHOD>
Retention: <RETENTION_POLICY>
```

Confirm that the target is mounted before writing:

```bash
findmnt <BACKUP_TARGET>
df -h <BACKUP_TARGET>
```

Do not assume that a directory on the same server is a separate backup target.

Create a protected backup directory:

```bash
sudo install -d -o root -g root -m 0700 \
  <BACKUP_TARGET>/server-setup
```

Before using a mirror-style backup with `--delete`, perform a dry run:

```bash
sudo rsync -aHAXn --delete \
  /srv/ <BACKUP_TARGET>/server-setup/srv/
```

Review the dry-run output before removing the `n` option.

## Backup Scope

Back up the following categories:

| Category                        | Example data                                 |
| ------------------------------- | -------------------------------------------- |
| User files                      | `/srv/files`                                 |
| Gitea data                      | `/srv/docker/gitea/data`                     |
| Pi-hole data                    | `/srv/docker/pihole/etc-pihole`              |
| Compose files                   | Gitea and Pi-hole Compose files              |
| Password secret                 | Pi-hole web password file                    |
| Samba configuration             | `/etc/samba/smb.conf`                        |
| Resolver configuration          | `/etc/systemd/resolved.conf.d/`              |
| Firewall configuration          | UFW rules and defaults                       |
| Docker repository configuration | Docker APT source and key                    |
| Recovery documentation          | This project and private operational records |

Do not back up temporary container layers, unneeded Docker cache, private
Tailscale state unless specifically required, unneeded system logs, or secrets
into an unencrypted public repository.

## Step 6: Back Up Shared Files

Review the source:

```bash
sudo du -sh /srv/files
sudo find /srv/files -maxdepth 2 \
  -printf '%M %u %g %p\n'
```

Perform a dry run:

```bash
sudo rsync -aHAXn --delete \
  /srv/files/ <BACKUP_TARGET>/server-setup/srv/files/
```

After reviewing the output, perform the backup:

```bash
sudo rsync -aHAX --delete \
  /srv/files/ <BACKUP_TARGET>/server-setup/srv/files/
```

Record the result:

```bash
sudo du -sh <BACKUP_TARGET>/server-setup/srv/files
```

The backup should preserve file ownership, permissions, timestamps, and
directory structure where the backup filesystem supports them.

## Step 7: Back Up Gitea

For a consistent simple filesystem backup, stop Gitea briefly:

```bash
sudo docker compose \
  -f /srv/docker/gitea/docker-compose.yml stop
```

Create the backup archive:

```bash
sudo tar -C /srv/docker/gitea \
  -czf <BACKUP_TARGET>/server-setup/gitea-<DATE>.tar.gz \
  data docker-compose.yml
```

Start Gitea:

```bash
sudo docker compose \
  -f /srv/docker/gitea/docker-compose.yml start
```

Verify:

```bash
sudo docker ps --filter name=gitea
curl -I http://127.0.0.1:3000
```

Test repository access after the backup:

```bash
git ls-remote \
  ssh://git@<GITEA_HOSTNAME>:222/<GITEA_USER>/<REPOSITORY>.git
```

Protect the backup because it contains repositories, the database, application
configuration, and possibly generated keys.

## Step 8: Back Up Pi-hole

Because Pi-hole affects client DNS, schedule this during a maintenance window.

Stop Pi-hole briefly:

```bash
sudo docker compose \
  -f /srv/docker/pihole/docker-compose.yml stop
```

Create the backup archive:

```bash
sudo tar -C /srv/docker/pihole \
  -czf <BACKUP_TARGET>/server-setup/pihole-<DATE>.tar.gz \
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
sudo ss -lntup | grep ':53'
dig @127.0.0.1 ubuntu.com
```

The password-containing archive must be protected as a secret.

## Step 9: Back Up Host Configuration

Create a temporary staging directory outside the repository:

```bash
sudo install -d -o root -g root -m 0700 \
  /root/server-setup-config-backup
```

Copy required configuration:

```bash
sudo cp --preserve=mode,ownership \
  /etc/samba/smb.conf \
  /root/server-setup-config-backup/

sudo cp --preserve=mode,ownership \
  /etc/systemd/resolved.conf.d/pihole.conf \
  /root/server-setup-config-backup/

sudo cp --preserve=mode,ownership \
  /etc/docker/daemon.json \
  /root/server-setup-config-backup/
```

If a file does not exist, record it as not applicable rather than creating an
empty replacement.

Record UFW configuration:

```bash
sudo ufw status verbose \
  > /root/server-setup-config-backup/ufw-status.txt

sudo ufw show raw \
  > /root/server-setup-config-backup/ufw-raw.txt
```

Archive the staging directory:

```bash
sudo tar -C /root \
  -czf <BACKUP_TARGET>/server-setup/host-config-<DATE>.tar.gz \
  server-setup-config-backup
```

Remove the temporary staging directory after confirming the archive exists:

```bash
sudo rm -rf /root/server-setup-config-backup
```

Do not include the staging directory or archive in the public repository.

## Step 10: Verify Backup Completion

List recent backups:

```bash
sudo ls -lh <BACKUP_TARGET>/server-setup
```

Check archive contents without extracting:

```bash
sudo tar -tzf \
  <BACKUP_TARGET>/server-setup/gitea-<DATE>.tar.gz

sudo tar -tzf \
  <BACKUP_TARGET>/server-setup/pihole-<DATE>.tar.gz
```

Compare shared file totals:

```bash
sudo du -sh /srv/files
sudo du -sh <BACKUP_TARGET>/server-setup/srv/files
```

Record:

```text
Backup date: <DATE>
Backup target: <BACKUP_TARGET>
Backup result: PASS / FAIL
Files backup: PASS / FAIL
Gitea backup: PASS / FAIL
Pi-hole backup: PASS / FAIL
Host configuration backup: PASS / FAIL
Restore test due: <DATE>
```

A backup is not considered complete until its contents can be read and at least
one restore has been tested.

## Step 11: Restore an Individual File

Identify the required file in the backup:

```bash
sudo find <BACKUP_TARGET>/server-setup/srv/files \
  -name '<FILE_NAME>'
```

Create a temporary restore directory:

```bash
sudo install -d -o root -g root -m 0700 \
  /root/file-restore-test
```

Copy the file into the temporary directory:

```bash
sudo rsync -a \
  <BACKUP_TARGET>/server-setup/srv/files/<PATH_TO_FILE> \
  /root/file-restore-test/
```

Inspect it:

```bash
sudo ls -l /root/file-restore-test
```

Compare it with the expected file using a safe method:

```bash
sudo sha256sum /root/file-restore-test/<FILE_NAME>
```

Restore it to the share only after confirming the destination and ownership:

```bash
sudo install -o <FILE_OWNER> -g <FILE_GROUP> -m <FILE_MODE> \
  /root/file-restore-test/<FILE_NAME> \
  /srv/files/<RESTORE_PATH>
```

Remove the temporary restore directory:

```bash
sudo rm -rf /root/file-restore-test
```

Record the restore result and verify that the file is available through Samba.

## Step 12: Restore Gitea

Before restoring:

1. Confirm the backup archive is readable.
2. Confirm the expected Gitea version.
3. Stop the current Gitea container.
4. Preserve the existing data directory.
5. Verify the target filesystem has enough space.
6. Record the current container state.

Stop Gitea:

```bash
sudo docker compose \
  -f /srv/docker/gitea/docker-compose.yml stop
```

Preserve the current data:

```bash
sudo mv /srv/docker/gitea/data \
  /srv/docker/gitea/data-before-restore
```

Extract the backup:

```bash
sudo tar -C /srv/docker/gitea \
  -xzf <BACKUP_TARGET>/server-setup/gitea-<DATE>.tar.gz
```

Restore the expected ownership:

```bash
sudo chown -R <GITEA_UID>:<GITEA_GID> \
  /srv/docker/gitea/data
```

Start Gitea:

```bash
sudo docker compose \
  -f /srv/docker/gitea/docker-compose.yml start
```

Verify:

```bash
sudo docker ps --filter name=gitea
sudo docker logs --tail=200 gitea
curl -I http://127.0.0.1:3000
```

Confirm administrator login, repository visibility, repository clone, Git SSH on
port 222, test commit and push, and existing repository data.

Do not delete `data-before-restore` until the restore is validated.

## Step 13: Restore Pi-hole

Before restoring:

1. Confirm the backup archive is readable.
2. Confirm the expected Pi-hole version.
3. Prepare an alternate DNS resolver.
4. Warn affected users about possible DNS interruption.
5. Stop Pi-hole.
6. Preserve the current data directory.

Stop Pi-hole:

```bash
sudo docker compose \
  -f /srv/docker/pihole/docker-compose.yml stop
```

Preserve current data:

```bash
sudo mv /srv/docker/pihole/etc-pihole \
  /srv/docker/pihole/etc-pihole-before-restore
```

Extract the backup:

```bash
sudo tar -C /srv/docker/pihole \
  -xzf <BACKUP_TARGET>/server-setup/pihole-<DATE>.tar.gz
```

Restore the password secret permissions:

```bash
sudo chown root:root \
  /srv/docker/pihole/pihole_webpassword

sudo chmod 0600 \
  /srv/docker/pihole/pihole_webpassword
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

## Step 14: Recover From a Failed Configuration Change

If a service fails after a configuration change:

1. Keep an existing administrative session open.
2. Identify the changed file.
3. Validate the configuration.
4. Review the service logs.
5. Restore the previous configuration if necessary.
6. Restart only the affected service.
7. Repeat the relevant acceptance tests.

For Samba:

```bash
sudo testparm
sudo systemctl status smbd --no-pager
sudo journalctl -u smbd --no-pager -n 100
```

For Gitea:

```bash
sudo docker compose \
  -f /srv/docker/gitea/docker-compose.yml config

sudo docker logs --tail=200 gitea
```

For Pi-hole:

```bash
sudo docker compose \
  -f /srv/docker/pihole/docker-compose.yml config

sudo docker logs --tail=200 pihole
sudo ss -lntup | grep ':53'
```

For the resolver:

```bash
resolvectl status
sudo systemctl status systemd-resolved --no-pager
sudo journalctl -u systemd-resolved --no-pager -n 100
```

Do not delete persistent application data to solve a configuration problem.

## Step 15: Recover From a DNS Outage

If Pi-hole is unavailable:

1. Confirm that IP connectivity still works.
2. Confirm the Pi-hole container state.
3. Confirm port 53 ownership.
4. Review Pi-hole logs.
5. Check the resolver configuration.
6. Use an alternate resolver temporarily if necessary.
7. Restore Pi-hole.
8. Verify blocked and permitted queries.
9. Restore normal client DNS settings.

Check Pi-hole:

```bash
sudo docker ps --filter name=pihole
sudo docker logs --tail=200 pihole
sudo docker inspect \
  --format '{{.State.Health.Status}}' pihole
sudo ss -lntup | grep ':53'
```

Temporarily test an external resolver without changing permanent configuration:

```bash
dig @1.1.1.1 ubuntu.com
```

Do not add a public fallback to router DHCP as a permanent workaround when
network-wide filtering is required.

After recovery:

```bash
resolvectl query ubuntu.com
dig @127.0.0.1 doubleclick.net
```

Document the outage, impact, cause, workaround, recovery, and verification.

## Step 16: Recover From a Docker Outage

Check Docker:

```bash
sudo systemctl status docker --no-pager
sudo journalctl -u docker --no-pager -n 100
```

Check storage:

```bash
df -h
sudo docker system df
```

Check containers:

```bash
sudo docker ps -a
```

If the daemon is stopped and the cause is understood:

```bash
sudo systemctl restart docker
```

Verify:

```bash
sudo docker ps
sudo docker logs --tail=100 gitea
sudo docker logs --tail=100 pihole
```

Repeat DNS and Gitea tests after Docker recovery.

## Step 17: Recover From a Server Address Change

Check the current address:

```bash
ip -br address
ip route
```

Check the router DHCP lease and reservation.

Confirm:

- The server MAC address matches the reservation.
- The reservation is assigned to the correct interface.
- The server is connected to the intended router.
- Clients are using the current address.
- Pi-hole DHCP DNS settings are updated if necessary.

Use Tailscale access as an alternate management path when available:

```bash
tailscale status
ssh <SERVER_USER>@<SERVER_TAILSCALE_IP>
```

Do not change every service configuration until the server address issue is
understood.

## Step 18: Roll Back a Failed Application Update

Record the failed version:

```bash
sudo docker inspect gitea \
  --format '{{.Config.Image}}'

sudo docker inspect pihole \
  --format '{{.Config.Image}}'
```

Review logs and release notes.

Set the Compose file back to the previously tested image version:

```bash
sudoedit /srv/docker/<SERVICE>/docker-compose.yml
```

Validate:

```bash
sudo docker compose \
  -f /srv/docker/<SERVICE>/docker-compose.yml config
```

Recreate the service:

```bash
sudo docker compose \
  -f /srv/docker/<SERVICE>/docker-compose.yml up -d
```

Verify service health and application data.

Do not restore application data from backup unless the failure damaged or
migrated the data. An image rollback and a data restore are separate actions.

## Change Record Template

Record each planned change:

```text
Change ID: <CHANGE_ID>
Date: <DATE>
Administrator: <ROLE_OR_ALIAS>
Service: <SERVICE>
Reason: <REASON>
Current version: <CURRENT_VERSION>
Target version: <TARGET_VERSION>
Backup verified: YES / NO
Maintenance window: <TIME_RANGE>
Expected impact: <IMPACT>
Validation plan: <TESTS>
Rollback plan: <ROLLBACK>
Result: SUCCESS / FAILED / ROLLED BACK
Evidence: <SANITIZED_REFERENCE>
```

## Incident Record Template

Record service incidents:

```text
Incident ID: <INCIDENT_ID>
Start time: <TIME>
End time: <TIME>
Affected service: <SERVICE>
User impact: <IMPACT>
Observed symptom: <SYMPTOM>
Initial hypothesis: <HYPOTHESIS>
Diagnostic tests: <TESTS>
Root cause: <CAUSE>
Recovery action: <ACTION>
Verification: <RESULT>
Follow-up action: <ACTION>
Evidence: <SANITIZED_REFERENCE>
```

## Recovery Objectives

Define realistic objectives for this single-server lab:

```text
Target recovery point objective: <RPO>
Target recovery time objective: <RTO>
Maximum acceptable DNS interruption: <DURATION>
Maximum acceptable Git service interruption: <DURATION>
Maximum acceptable file-share interruption: <DURATION>
```

These values are planning targets, not production guarantees.

The actual recovery time depends on backup availability, replacement hardware,
backup transfer speed, disk capacity, configuration accuracy, DNS recovery
requirements, Tailscale reauthentication, and manual validation time.

## Backup Retention

Define a retention policy:

| Backup type         | Suggested retention               |
| ------------------- | --------------------------------- |
| Daily backup        | `<RETENTION>`                     |
| Weekly backup       | `<RETENTION>`                     |
| Monthly backup      | `<RETENTION>`                     |
| Pre-update backup   | Until update is validated         |
| Restore-test backup | Until test is complete            |
| Incident backup     | Until incident review is complete |

Adjust retention to available storage and the value of the data.

Delete old backups only after confirming:

- Newer backups exist.
- Required retention is satisfied.
- At least one independent copy remains.
- No restore investigation depends on the old backup.

## Security Considerations

- Store backups on separate storage.
- Encrypt backups containing passwords, repositories, or private files.
- Restrict backup permissions.
- Do not publish backup archives.
- Do not store secrets in the public repository.
- Protect Pi-hole password backups.
- Protect Gitea data and database backups.
- Review who can access the backup target.
- Do not use a same-disk directory as the only backup.
- Test restore procedures without exposing private data.
- Do not use `--delete` without a reviewed dry run.
- Preserve file ownership and permissions where required.
- Keep service configuration root-owned.
- Document any temporary emergency DNS workaround.
- Remove temporary restore files after validation.

## Verification Checklist

| Check                     | Expected result                                         |
| ------------------------- | ------------------------------------------------------- |
| Maintenance record        | Current state is recorded                               |
| Package updates           | Updates complete without unresolved errors              |
| Docker updates            | Specific versions are recorded                          |
| Service health            | Required services are active                            |
| Storage                   | Adequate free space exists                              |
| Logs                      | Warnings and errors are reviewed                        |
| Backup target             | Separate target is mounted                              |
| Shared files backup       | Backup completes and is readable                        |
| Gitea backup              | Data and configuration are backed up                    |
| Pi-hole backup            | Data and protected secret are backed up                 |
| Host configuration backup | Resolver, Samba, UFW, and Docker settings are backed up |
| Individual restore        | A file restore succeeds                                 |
| Gitea restore             | Gitea restore procedure is tested                       |
| Pi-hole restore           | Pi-hole restore procedure is tested                     |
| Reboot recovery           | Services return after reboot                            |
| Update rollback           | Rollback procedure is documented                        |
| DNS recovery              | Alternate DNS procedure is documented                   |
| Evidence                  | Results are sanitized                                   |
| Retention                 | Backup retention is defined                             |
| Limitations               | Single-server risks are documented                      |

## Evidence To Capture

Capture sanitized evidence showing:

- A completed maintenance record.
- System and service health checks.
- Disk-capacity reviews.
- Container status and image versions.
- Backup target verification without private paths.
- Backup archive listings without sensitive filenames.
- Successful individual-file restore.
- Successful Gitea restore test.
- Successful Pi-hole restore test.
- Reboot recovery.
- Update verification.
- A controlled rollback or recovery exercise.
- Defined recovery objectives.
- Documented service-impact limitations.

Remove or replace:

- Passwords.
- Private keys.
- Access tokens.
- Personal usernames.
- Email addresses.
- Live IP addresses.
- Tailscale device identifiers.
- Private repository names.
- Personal file names.
- DNS query history.
- Backup encryption keys.
- Unsanitized logs.
- Real backup paths.

## Completion Criteria

This module is complete when:

- Routine maintenance tasks are documented.
- Current host and service state can be recorded.
- Ubuntu update procedures are documented.
- Docker and application update procedures are documented.
- Service health and log reviews are documented.
- Shared files are backed up to separate storage.
- Gitea data and configuration are backed up.
- Pi-hole data and protected credentials are backed up.
- Resolver, firewall, Samba, and Docker host configuration are backed up.
- Backup contents can be inspected.
- An individual file restore has been completed.
- Gitea restore has been tested.
- Pi-hole restore has been tested.
- Reboot recovery has been verified.
- Update rollback guidance exists.
- DNS and Docker outage recovery procedures exist.
- Recovery objectives are documented.
- Backup retention is defined.
- Sanitized operational evidence has been captured.
- Single-server limitations are documented honestly.

Next: [Troubleshooting Playbooks](12-troubleshooting-playbooks.md)
