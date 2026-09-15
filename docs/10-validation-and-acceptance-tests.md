# Validation And Acceptance Tests

## Purpose

This module consolidates the validation procedures for the self-hosted server.

It verifies that the server, network, firewall, containers, and hosted services
work together through the intended local and remote access paths.

This module does not replace the detailed implementation procedures. It provides
a repeatable acceptance sequence after changes, upgrades, or recovery events.

## Learning Objectives

After completing this module, an administrator should be able to:

- Validate the server from the operating system layer upward.
- Test local LAN and remote Tailscale access separately.
- Confirm that each service is listening on the intended port.
- Test both successful and denied access.
- Verify DNS filtering through direct and normal resolver queries.
- Confirm service persistence after a reboot.
- Record test results as operational evidence.
- Distinguish failed services from failed network paths.
- Document a controlled failure and recovery.

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

Testing requires:

| Requirement | Purpose |
|---|---|
| Server administrative access | Host validation |
| Local LAN client | Local service tests |
| Authorized Tailscale client | Remote service tests |
| Client on another network | External access test |
| Test Samba account | File-service validation |
| Test Gitea repository | Git-service validation |
| `dig` or equivalent DNS tool | DNS validation |
| `nc` or equivalent port tool | Port validation |
| Maintenance window | Reboot and failure testing |

## Test Environment Record

Record sanitized values before beginning:

```text
Test date: <DATE>
Server hostname: <SERVER_HOSTNAME>
Server LAN address: <SERVER_LAN_IP>
Server Tailscale address: <SERVER_TAILSCALE_IP>
Server interface: <SERVER_INTERFACE>
LAN CIDR: <LAN_CIDR>
Router address: <ROUTER_LAN_IP>
Docker image versions: <IMAGE_VERSIONS>
Client operating systems: <CLIENT_PLATFORMS>
```

Do not record passwords, private keys, device IDs, personal usernames, or
unsanitized DNS query history.

## Test Method

Each test should record:

- Test identifier.
- Objective.
- Test location.
- Command or action.
- Expected result.
- Actual result.
- Pass or fail status.
- Evidence reference.
- Recovery action if the test fails.

Use these result values:

| Result | Meaning |
|---|---|
| PASS | Expected behavior was confirmed |
| FAIL | Expected behavior was not confirmed |
| BLOCKED | Test could not run because a prerequisite failed |
| NOT APPLICABLE | Test does not apply to this environment |

A failed prerequisite should be corrected before dependent tests are run.

## Test Order

Run tests in this order:

```text
1. Server identity and resources
2. Network interface and routing
3. Resolver state
4. Firewall policy
5. System services
6. Docker and containers
7. Local LAN access
8. Remote Tailscale access
9. DNS filtering
10. Security and negative tests
11. Reboot recovery
12. Controlled failure and recovery
13. Evidence review
```

This order narrows failures from the lowest dependency layer upward.

## Test 1: Server Identity and Resources

### 1.1 Operating system

Run on the server:

```bash
lsb_release -a
uname -r
hostnamectl
```

Expected result:

- The expected Ubuntu Server release is installed.
- The expected hostname is configured.
- The running kernel is identified.

### 1.2 Time synchronization

```bash
timedatectl
```

Expected result:

- The timezone is correct.
- The system clock is synchronized or intentionally configured.

### 1.3 Storage

```bash
df -h
lsblk
```

Expected result:

- Required filesystems are mounted.
- `/srv` has adequate free space.
- No filesystem is unexpectedly full.

### 1.4 Memory and system load

```bash
free -h
uptime
```

Expected result:

- Available memory is appropriate for the workload.
- No unexplained sustained load is present.

## Test 2: Network Interface and Routing

### 2.1 Interface state

```bash
ip -br address
```

Expected result:

- The server's wired interface is up.
- The server has the expected LAN address.
- `tailscale0` is present when Tailscale is enabled.
- Docker interfaces are present only as expected.

### 2.2 Routing table

```bash
ip route
```

Expected result:

- The local LAN route uses the expected interface.
- The default route uses the primary router.
- No unexpected default route is present.

### 2.3 Gateway reachability

```bash
ping -c 3 <ROUTER_LAN_IP>
ip route get <ROUTER_LAN_IP>
```

Expected result:

- The server can reach the primary router.
- The route uses the expected interface and source address.

A failed ping does not always prove routing is broken because some networks
block ICMP. Review the route and use application-level tests as additional
evidence.

### 2.4 External IP reachability

```bash
ip route get 1.1.1.1
ping -c 3 1.1.1.1
```

Expected result:

- The route uses the expected default gateway.
- External IP connectivity works where ICMP is permitted.

## Test 3: Resolver State

### 3.1 Resolver configuration

```bash
resolvectl status
ls -l /etc/resolv.conf
```

Expected result:

- Resolver state is understandable.
- `/etc/resolv.conf` points to the intended systemd-resolved file.
- The server's active interface uses the intended resolver path.

### 3.2 Server DNS query

```bash
resolvectl query ubuntu.com
```

Expected result:

- The permitted domain resolves.
- The response is returned through the intended Pi-hole path after Pi-hole is
  deployed.

### 3.3 Direct Pi-hole query

```bash
dig @127.0.0.1 ubuntu.com
```

Expected result:

- Pi-hole answers the query locally.

### 3.4 Direct blocked query

```bash
dig @127.0.0.1 doubleclick.net
```

Expected result:

- The response is blocked by Pi-hole.
- The response is commonly `0.0.0.0` or another locally configured blocked
  response.

### 3.5 Port 53 ownership

```bash
sudo ss -lntup | grep ':53'
```

Expected result:

- Pi-hole owns the intended TCP and UDP DNS listeners.
- The systemd-resolved stub is not competing for port 53.

## Test 4: Firewall Policy

### 4.1 UFW status

```bash
sudo ufw status verbose
sudo ufw status numbered
```

Expected result:

- Incoming traffic is denied by default.
- Outgoing traffic is allowed by default.
- Rules are limited to required services and networks.

### 4.2 Expected port policy

Review the intended access:

| Service | LAN | Tailscale | Public internet |
|---|---|---|---|
| Host SSH, TCP 22 | Allowed | Allowed | Denied |
| Samba, TCP 445 | Allowed | Allowed if required | Denied |
| Gitea web, TCP 3000 | Allowed | Allowed | Denied |
| Gitea SSH, TCP 222 | Allowed | Allowed | Denied |
| DNS, TCP/UDP 53 | Allowed | Allowed | Denied |
| Pi-hole web, TCP 80 | Administrative clients | Administrative clients | Denied |

### 4.3 Listening sockets

```bash
sudo ss -lntup
```

Expected result:

- Every listener has a documented purpose.
- No unexpected administration, database, or application port is exposed.

A listening port alone does not prove that the service is correctly secured.
Test reachability and authorization separately.

## Test 5: System Services

Check each required service:

```bash
sudo systemctl status ssh --no-pager
sudo systemctl status tailscaled --no-pager
sudo systemctl status docker --no-pager
sudo systemctl status smbd --no-pager
sudo systemctl status systemd-resolved --no-pager
```

Expected result:

- Required services are active.
- Services expected to start automatically are enabled.
- No unresolved errors appear in the status output.

Verify startup configuration:

```bash
systemctl is-enabled ssh
systemctl is-enabled tailscaled
systemctl is-enabled docker
systemctl is-enabled smbd
systemctl is-enabled systemd-resolved
```

## Test 6: Docker and Container Health

### 6.1 Docker daemon

```bash
sudo docker version
sudo docker info
sudo docker compose version
```

Expected result:

- The Docker client and daemon respond.
- Docker Compose is available.
- No Docker group membership is required for administration.

### 6.2 Container status

```bash
sudo docker ps
```

Expected result:

- The expected Gitea and Pi-hole containers are running.
- No required container is repeatedly restarting.

### 6.3 Container health

```bash
sudo docker inspect \
  --format '{{.Name}} {{.State.Status}} {{.State.Health.Status}}' \
  gitea pihole
```

Expected result:

- Gitea is running.
- Pi-hole is running and healthy when a health check is configured.

### 6.4 Container logs

```bash
sudo docker logs --tail=100 gitea
sudo docker logs --tail=100 pihole
```

Expected result:

- No unresolved startup, permission, port, or database errors exist.
- Logs are sanitized before being captured as evidence.

### 6.5 Persistent mounts

```bash
sudo docker inspect gitea \
  --format '{{json .Mounts}}'

sudo docker inspect pihole \
  --format '{{json .Mounts}}'
```

Expected result:

- Gitea data is mounted from `/srv/docker/gitea/data`.
- Pi-hole data is mounted from `/srv/docker/pihole/etc-pihole`.
- No required service data exists only in a container writable layer.

## Test 7: Local LAN Access

Run these tests from a client connected to the intended LAN.

### 7.1 Host SSH

```bash
ssh <SERVER_USER>@<SERVER_LAN_IP>
```

Expected result:

- The connection succeeds.
- `hostname` and `whoami` return expected values.

### 7.2 Service port reachability

```bash
nc -vz <SERVER_LAN_IP> 445
nc -vz <SERVER_LAN_IP> 222
nc -vz <SERVER_LAN_IP> 3000
nc -vz <SERVER_LAN_IP> 53
nc -vz <SERVER_LAN_IP> 80
```

Expected result:

- Required TCP services are reachable.
- UDP DNS is tested separately with `dig`.

### 7.3 Samba

```bash
smbclient //<SERVER_LAN_IP>/files -U <SERVER_USER>
```

Test authentication, directory listing, file creation, file read, and file
deletion.

Expected result:

- Authorized users can perform intended operations.
- Guest access is denied.

### 7.4 Gitea web interface

Open:

```text
http://<SERVER_LAN_IP>:3000
```

Expected result:

- The Gitea web interface loads.
- The administrator can sign in.
- The test repository is visible.

### 7.5 Gitea Git SSH

```bash
ssh -p 222 git@<SERVER_LAN_IP>
```

Expected result:

- The Gitea key is accepted.
- An interactive shell is refused as expected.

### 7.6 Gitea repository operation

```bash
git clone \
  ssh://git@<SERVER_LAN_IP>:222/<GITEA_USER>/<REPOSITORY>.git

cd <REPOSITORY>
git ls-remote origin
```

Expected result:

- The repository can be cloned.
- Git can read the remote repository through port 222.

Perform a test commit and push if the test repository is writable:

```bash
printf '%s\n' 'LAN validation' > validation.txt
git add validation.txt
git commit -m "Validate LAN Git access"
git push origin main
```

Remove the test file or repository afterward if it is not intended to remain.

### 7.7 Local DNS

Check the client resolver:

```bash
resolvectl status
```

Query a permitted domain:

```bash
dig ubuntu.com
```

Query a blocked domain:

```bash
dig doubleclick.net
```

Expected result:

- The client uses `<SERVER_LAN_IP>` as DNS.
- Permitted domains resolve.
- Blocked test domains return Pi-hole's blocked response.
- Queries appear in Pi-hole.

## Test 8: Remote Tailscale Access

Run these tests from an authorized client connected to Tailscale.

### 8.1 Tailscale state

```bash
tailscale status
tailscale ping <SERVER_TAILSCALE_IP>
```

Expected result:

- The server appears online.
- The client can reach the server through the tailnet.

### 8.2 Host administration SSH

```bash
ssh <SERVER_USER>@<SERVER_TAILSCALE_IP>
```

Expected result:

- Tailscale SSH authenticates through the tailnet policy.
- The expected Linux user is available.

### 8.3 Remote service ports

```bash
nc -vz <SERVER_TAILSCALE_IP> 445
nc -vz <SERVER_TAILSCALE_IP> 222
nc -vz <SERVER_TAILSCALE_IP> 3000
nc -vz <SERVER_TAILSCALE_IP> 53
nc -vz <SERVER_TAILSCALE_IP> 80
```

Expected result:

- Required remote services are reachable.
- Services not intended for remote access remain unavailable.

### 8.4 Remote Samba

```bash
smbclient //<SERVER_TAILSCALE_IP>/files -U <SERVER_USER>
```

Test authentication, directory listing, file creation, file read, and file
deletion.

Expected result:

- Authorized remote users can access the share.
- Guest access is denied.

### 8.5 Remote Gitea web interface

Open:

```text
http://<SERVER_TAILSCALE_IP>:3000
```

Expected result:

- The Gitea interface loads through Tailscale.
- The administrator can sign in.
- The test repository is available.

### 8.6 Remote Gitea Git SSH

```bash
ssh -p 222 git@<SERVER_TAILSCALE_IP>
```

Expected result:

- The Gitea SSH key is accepted.
- An interactive shell is refused as expected.

### 8.7 Remote Git operation

```bash
git clone \
  ssh://git@<SERVER_TAILSCALE_IP>:222/<GITEA_USER>/<REPOSITORY>.git

cd <REPOSITORY>
git ls-remote origin
```

Expected result:

- Git operations succeed through the encrypted Tailscale path.

### 8.8 Remote DNS

Query Pi-hole directly:

```bash
dig @<SERVER_TAILSCALE_IP> ubuntu.com
dig @<SERVER_TAILSCALE_IP> doubleclick.net
```

Test the normal client resolver:

```bash
resolvectl status
resolvectl query ubuntu.com
resolvectl query doubleclick.net
```

Expected result:

- The client uses the Tailscale DNS policy.
- Permitted domains resolve.
- Blocked test domains are blocked.
- Queries appear in Pi-hole.

## Test 9: External Network Access

Move the authorized client to a network outside the home LAN, such as a phone
hotspot or another trusted network.

Confirm:

```bash
tailscale status
tailscale ping <SERVER_TAILSCALE_IP>
ssh <SERVER_USER>@<SERVER_TAILSCALE_IP>
```

Test Gitea web access, Gitea Git SSH, Samba access if required, direct Pi-hole
DNS, and normal client DNS filtering.

Expected result:

- Authorized Tailscale services work from outside the home LAN.
- No router port forwarding is required.
- The server's LAN address is not used for remote access.

## Test 10: Security and Negative Tests

### 10.1 Guest Samba access

From a client:

```bash
smbclient //<SERVER_LAN_IP>/files -N
```

Expected result:

- Guest access is denied.

### 10.2 Unauthorized Samba user

Use an account that is not in the `fileshare` group:

```bash
smbclient //<SERVER_LAN_IP>/files -U <UNAUTHORIZED_USER>
```

Expected result:

- Authentication or share authorization fails.

### 10.3 Unauthorized Gitea repository access

Use a Gitea account or key without repository permission.

Expected result:

- The account cannot read or write the restricted repository.

### 10.4 Unauthenticated Gitea administration

Open the Gitea interface in a private browser session.

Expected result:

- Administrative pages require authentication.
- Open registration is disabled unless intentionally required.

### 10.5 Public exposure review

Review the router configuration:

- No SSH port forward.
- No Samba port forward.
- No Gitea port forward.
- No DNS port forward.
- No Pi-hole web port forward.

Review server listeners:

```bash
sudo ss -lntup
sudo ufw status verbose
```

Expected result:

- Services are reachable only through intended LAN and Tailscale paths.

### 10.6 Docker privilege review

```bash
id
getent group docker
```

Expected result:

- The administrative user is not in the `docker` group.
- Docker administration uses explicit `sudo`.

## Test 11: Reboot Recovery

Before rebooting, record:

```bash
sudo systemctl status docker --no-pager
sudo systemctl status smbd --no-pager
sudo systemctl status tailscaled --no-pager
sudo docker ps
sudo ufw status verbose
sudo ss -lntup
resolvectl status
```

Reboot during a planned maintenance window:

```bash
sudo reboot
```

After the server returns, verify:

```bash
sudo systemctl status ssh --no-pager
sudo systemctl status tailscaled --no-pager
sudo systemctl status docker --no-pager
sudo systemctl status smbd --no-pager
sudo systemctl status systemd-resolved --no-pager
sudo docker ps
```

Repeat local host SSH, remote Tailscale SSH, Samba access, Gitea web access,
Gitea Git SSH, permitted DNS, blocked DNS, port, and firewall checks.

Expected result:

- Required services return automatically.
- Persistent data remains available.
- Pi-hole owns port 53 after reboot.
- No manual service startup is required.

## Test 12: Controlled Failure and Recovery

Perform this test only during a maintenance window.

Use Gitea as the controlled failure target because stopping it does not
interrupt network-wide DNS.

Record the current state:

```bash
sudo docker ps --filter name=gitea
curl -I http://127.0.0.1:3000
```

Stop Gitea:

```bash
sudo docker compose \
  -f /srv/docker/gitea/docker-compose.yml stop
```

Verify the expected failure:

```bash
sudo docker ps --filter name=gitea
curl -I http://127.0.0.1:3000
```

Expected result:

- The Gitea container is stopped.
- The Gitea web interface is unavailable.
- Pi-hole, Samba, and host administration remain available.

Recover Gitea:

```bash
sudo docker compose \
  -f /srv/docker/gitea/docker-compose.yml start
```

Verify recovery:

```bash
sudo docker ps --filter name=gitea
sudo docker logs --tail=100 gitea
curl -I http://127.0.0.1:3000
```

Repeat a repository access test.

Record:

- The symptom.
- The failure introduced.
- The diagnostic commands.
- The recovery command.
- The verification result.
- The evidence location.

Do not stop Pi-hole during a failure test unless an alternate DNS path and a
planned client-impact window are available.

## Test Results Template

Use a table like this for the completed test record:

| ID | Test | Location | Expected result | Actual result | Status | Evidence |
|---|---|---|---|---|---|---|
| NET-01 | Gateway reachability | Server | Gateway responds | `<RESULT>` | PASS/FAIL | `<REF>` |
| DNS-01 | Server permitted query | Server | Domain resolves | `<RESULT>` | PASS/FAIL | `<REF>` |
| DNS-02 | Server blocked query | Server | Domain is blocked | `<RESULT>` | PASS/FAIL | `<REF>` |
| FW-01 | UFW policy | Server | Expected rules exist | `<RESULT>` | PASS/FAIL | `<REF>` |
| DOC-01 | Docker health | Server | Containers healthy | `<RESULT>` | PASS/FAIL | `<REF>` |
| SMB-01 | LAN share access | LAN client | Authorized access works | `<RESULT>` | PASS/FAIL | `<REF>` |
| GIT-01 | LAN Git SSH | LAN client | Clone succeeds | `<RESULT>` | PASS/FAIL | `<REF>` |
| TS-01 | Tailscale connectivity | Remote client | Overlay responds | `<RESULT>` | PASS/FAIL | `<REF>` |
| SMB-02 | Remote share access | Remote client | Authorized access works | `<RESULT>` | PASS/FAIL | `<REF>` |
| GIT-02 | Remote Git SSH | Remote client | Clone succeeds | `<RESULT>` | PASS/FAIL | `<REF>` |
| DNS-03 | Remote blocked query | Remote client | Domain is blocked | `<RESULT>` | PASS/FAIL | `<REF>` |
| SEC-01 | Guest Samba access | Client | Access denied | `<RESULT>` | PASS/FAIL | `<REF>` |
| SEC-02 | Public exposure review | Router/server | No port forwards | `<RESULT>` | PASS/FAIL | `<REF>` |
| REC-01 | Reboot recovery | Server/clients | Services recover | `<RESULT>` | PASS/FAIL | `<REF>` |
| REC-02 | Controlled failure | Server | Service recovers | `<RESULT>` | PASS/FAIL | `<REF>` |

## Failure Interpretation

| Symptom | First diagnostic layer |
|---|---|
| No server address | Interface, cable, DHCP, router |
| Cannot reach gateway | Link, subnet, route, router |
| IP works but domain fails | Resolver, Pi-hole, upstream DNS |
| Port connection refused | Service state or listener |
| Port times out | Firewall, route, interface, client network |
| LAN works but Tailscale fails | Tailscale state, policy, `tailscale0`, UFW |
| Tailscale works but service fails | Service port, Docker, UFW, application |
| Share reachable but access denied | Samba account, group, filesystem permissions |
| Gitea web works but Git fails | Key, port 222, repository permissions |
| Services fail after reboot | Enablement, restart policy, dependencies |
| Pi-hole unhealthy | Port 53 conflict, configuration, upstream DNS |
| Only some clients bypass filtering | Static DNS, VPN, encrypted DNS, private relay |

Do not change multiple unrelated layers at once during troubleshooting. Test one
hypothesis, record the result, and then continue.

## Evidence Requirements

Capture sanitized evidence for:

- Server identity and operating system.
- Network interface and route.
- Resolver status.
- UFW policy.
- Listening ports.
- Docker and container status.
- Samba local and remote access.
- Gitea web and Git SSH access.
- Pi-hole permitted and blocked queries.
- Tailscale connectivity.
- Reboot recovery.
- Controlled failure and recovery.
- Absence of public port forwarding.

Evidence should include enough context to reproduce the result without exposing
secrets or live infrastructure details.

Remove or replace:

- Passwords.
- Private keys.
- Access tokens.
- Personal usernames.
- Email addresses.
- Live IP addresses.
- Device IDs.
- Host fingerprints.
- Personal filenames.
- Full DNS query history.
- Unsanitized logs.

## Final Acceptance Criteria

The project passes acceptance when:

- The server boots and required services start automatically.
- The server has the expected network address and default route.
- The resolver state is consistent and understandable.
- Pi-hole owns host port 53 without resolver conflict.
- UFW denies unsolicited incoming traffic by default.
- Required ports are limited to intended networks and interfaces.
- Docker containers are running and persistent.
- Samba works for authorized LAN users.
- Samba works for authorized Tailscale users when remote access is required.
- Guest Samba access is denied.
- Gitea web access works locally and remotely.
- Gitea Git SSH works on port 222.
- Repository clone, commit, and push operations work.
- Permitted DNS queries resolve.
- Blocked DNS queries are filtered locally and remotely.
- Tailscale SSH works for authorized administrators.
- No public router port forwarding is required.
- Services recover after reboot.
- At least one controlled failure and recovery has been documented.
- Backup and restore procedures exist for persistent service data.
- Sanitized acceptance evidence has been recorded.
- Known limitations are documented honestly.

## Acceptance Summary

Complete this summary after testing:

```text
Overall status: PASS / FAIL / CONDITIONAL

Passed tests: <COUNT>
Failed tests: <COUNT>
Blocked tests: <COUNT>
Not applicable: <COUNT>

Known issues:
- <ISSUE>

Recovery actions required:
- <ACTION>

Evidence location:
- <SANITIZED_REFERENCE>

Reviewer:
- <ROLE_OR_ALIAS>

Review date:
- <DATE>
```

Next: [Operations, Backups, and Recovery](11-operations-backups-and-recovery.md)
