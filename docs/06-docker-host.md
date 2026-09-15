# Docker Host

## Purpose

This module installs and validates Docker on the Ubuntu server.

Docker provides the container runtime used by later service modules, including
Gitea and Pi-hole. This module establishes the host foundation without
deploying application-specific containers.

It covers:

- Installing Docker Engine from the official Docker repository.
- Installing the Docker Compose plugin.
- Verifying the Docker service and container runtime.
- Using explicit `sudo` for Docker administration.
- Establishing the server storage layout.
- Understanding Docker networking and published ports.
- Diagnosing common Docker host failures.

## Learning Objectives

After completing this module, an administrator should be able to:

- Explain the role of Docker Engine, containerd, Buildx, and Compose.
- Install Docker using an approved package repository.
- Verify the Docker daemon and Compose plugin.
- Run and remove a test container.
- Explain why Docker daemon access is a privileged operation.
- Organize persistent application data under `/srv/docker`.
- Inspect Docker storage, networks, images, and containers.
- Identify how Docker-published ports interact with the host firewall.
- Capture sanitized evidence of a working Docker host.

## Prerequisites

| Requirement | Purpose |
|---|---|
| Ubuntu Server 24.04 LTS | Docker host |
| Working local SSH access | Administration and recovery |
| Working Tailscale access | Optional remote administration |
| Administrative user with `sudo` | Package and service changes |
| Working DNS and internet access | Repository and image downloads |
| Adequate disk space | Images, containers, logs, and application data |
| Completed network and firewall modules | Addressing and access baseline |

Complete these modules first:

- [Ubuntu Server Installation](02-install-ubuntu-server.md)
- [Linux Baseline, Access, and Firewall](03-linux-baseline-access-and-firewall.md)
- [Networking, DNS, and Router Configuration](04-networking-dns-and-router.md)
- [SSH and Tailscale](05-ssh-and-tailscale.md)

## Docker Host Design

The server uses Docker Engine with Docker Compose for application services.

```text
Ubuntu Server
    |
    +-- Docker Engine
    |     |
    |     +-- containerd
    |     +-- Buildx
    |     +-- Docker Compose plugin
    |
    +-- Gitea container
    +-- Pi-hole container
    |
    +-- Persistent data under /srv/docker
```

Application containers are separated from the Ubuntu base system. Persistent
application data remains on the server filesystem rather than inside
temporary container layers.

This module does not create the Gitea or Pi-hole Compose files. Those
configurations belong to their individual service modules.

## Design Decisions

| Decision | Reason |
|---|---|
| Docker's official Ubuntu repository | Provides current Docker packages and the Compose plugin |
| Docker Engine package installation | Uses the standard supported container runtime |
| Rootful Docker | Compatible with the planned host-networked Pi-hole deployment |
| No Docker group membership | Avoids granting persistent access to a root-equivalent socket |
| Explicit `sudo` commands | Makes each privileged Docker operation visible |
| Root-owned Compose files | Protects service definitions and possible secrets |
| `/srv/docker` for application data | Separates persistent service data from the operating system |
| Separate service directories | Limits accidental changes between applications |
| No application containers in this module | Keeps host setup separate from service deployment |

Rootful Docker is convenient for this lab but has important security
implications. The Docker daemon runs with root-level privileges. This module
does not add the administrative user to the `docker` group, because that group
provides persistent root-equivalent access through the Docker socket.

## Step 1: Review the Host Before Installation

Check the operating system:

```bash
lsb_release -a
uname -r
```

Check available storage:

```bash
df -h
lsblk
```

Check memory and system load:

```bash
free -h
uptime
```

Check the server's network and DNS state:

```bash
ip -br address
ip route
resolvectl status
```

Docker image downloads require working outbound connectivity and DNS. Resolve
a known Docker domain before starting:

```bash
resolvectl query download.docker.com
```

Record any storage or memory limitations that may affect container reliability.

At minimum, reserve space for:

- Docker images.
- Container writable layers.
- Gitea repositories and database files.
- Pi-hole configuration and logs.
- System logs and updates.
- Backups.

## Step 2: Check for Conflicting Packages

Ubuntu may provide packages with names that overlap with Docker's official
packages. Remove conflicting packages only if they are installed and are not
required for another workload:

```bash
sudo apt remove docker.io docker-doc docker-compose podman-docker containerd runc
```

Removing packages is a change to the host. Review the package list before
confirming the operation.

Do not remove unrelated container tools or data without first determining
whether they are in use.

## Step 3: Install Repository Prerequisites

Install the packages required to trust and use the Docker repository:

```bash
sudo apt update
sudo apt install ca-certificates curl
```

Create the keyring directory:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

Download Docker's official repository key:

```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
```

Set readable permissions on the key:

```bash
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Add the Docker repository:

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Refresh package metadata:

```bash
sudo apt update
```

Confirm that the Docker repository is available:

```bash
apt-cache policy docker-ce
```

The output should show a candidate package from `download.docker.com`.

## Step 4: Install Docker Engine and Compose

Install Docker Engine and the supporting components:

```bash
sudo apt install docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

The installed components include:

| Package | Role |
|---|---|
| `docker-ce` | Docker Engine daemon |
| `docker-ce-cli` | Docker command-line client |
| `containerd.io` | Container runtime |
| `docker-buildx-plugin` | Image build functionality |
| `docker-compose-plugin` | Compose integration through `docker compose` |

Verify the installed versions:

```bash
docker --version
docker compose version
containerd --version
```

The Compose command uses a space:

```bash
docker compose
```

The legacy hyphenated command may not be installed:

```bash
docker-compose
```

Use the Compose plugin documented in this project.

## Step 5: Verify the Docker Service

Check the Docker service:

```bash
sudo systemctl status docker --no-pager
```

Enable Docker at boot and start it now:

```bash
sudo systemctl enable --now docker
```

Verify its state:

```bash
systemctl is-active docker
systemctl is-enabled docker
```

Expected results:

```text
active
enabled
```

Check the container runtime:

```bash
sudo systemctl status containerd --no-pager
```

If Docker fails to start, inspect the service logs:

```bash
sudo journalctl -u docker --no-pager -n 100
```

## Step 6: Test Docker With a Temporary Container

Run Docker's test image:

```bash
sudo docker run --rm hello-world
```

The command should download the image if necessary, start the container, print
the Docker welcome message, and remove the temporary container.

The `--rm` option removes the stopped test container. It does not remove the
downloaded image.

Inspect local images:

```bash
sudo docker image ls
```

Inspect all containers:

```bash
sudo docker ps -a
```

The test container should not remain in the stopped-container list because
`--rm` was used.

Confirm that the daemon responds:

```bash
sudo docker info
```

## Step 7: Use Explicit Docker Privileges

Do not add the administrative user to the `docker` group. Docker commands that
access the daemon should require explicit `sudo`:

```bash
sudo docker ps
sudo docker info
sudo docker compose version
```

Verify the administrative user's groups:

```bash
id
getent group docker
```

The `docker` group may exist because the Docker package creates it. Its
existence does not require adding users to it.

Membership in the `docker` group grants control over the Docker daemon. A user
with Docker access can generally mount host filesystems, start privileged
containers, and obtain root-level access to the host.

Rootless Docker is a possible future design, but it is not selected here. The
planned Pi-hole deployment uses host networking and low-numbered host ports,
which require additional rootless configuration and compatibility testing.

## Step 8: Establish the Storage Layout

Create the top-level directories with root ownership:

```bash
sudo install -d -o root -g root -m 0750 /srv/docker
sudo install -d -o root -g root -m 0750 /srv/files
```

Create the application parent directories with root ownership:

```bash
sudo install -d -o root -g root -m 0750 /srv/docker/gitea
sudo install -d -o root -g root -m 0750 /srv/docker/pihole
```

The planned layout is:

```text
/srv/
  |-- docker/
  |    |-- gitea/
  |    |    |-- docker-compose.yml
  |    |    +-- data/
  |    +-- pihole/
  |         |-- docker-compose.yml
  |         |-- etc-pihole/
  |         +-- etc-dnsmasq.d/
  |
  +-- files/
       Samba file share data
```

Use `sudoedit` to create or modify root-owned Compose files:

```bash
sudoedit /srv/docker/gitea/docker-compose.yml
sudoedit /srv/docker/pihole/docker-compose.yml
```

The service modules will create their data directories and apply
service-specific ownership and permissions.

Do not recursively change ownership of all of `/srv` after containers have
been deployed. Different services may require different users, groups, and
permission modes. The Samba module will define the permissions required for
`/srv/files`.

Review the resulting permissions:

```bash
sudo ls -ld /srv /srv/docker /srv/docker/gitea /srv/docker/pihole /srv/files
```

The storage layout is not a backup. Data under `/srv` still requires a separate
backup and restore procedure.

## Step 9: Inspect Docker Storage and Networks

Inspect Docker's disk usage:

```bash
sudo docker system df
```

List images:

```bash
sudo docker image ls
```

List volumes:

```bash
sudo docker volume ls
```

List networks:

```bash
sudo docker network ls
```

The default networks commonly include:

- `bridge`
- `host`
- `none`

Later Compose projects may create project-specific networks.

Do not remove images, volumes, or networks until their purpose and data impact
are understood.

The following command can remove unused objects and may delete data that is
not currently attached to a container:

```bash
sudo docker system prune
```

Do not run `docker system prune` as routine maintenance without reviewing its
confirmation prompt and expected impact.

## Docker Networking and Firewall Behavior

Docker can publish container ports on the host:

```text
Host port -> Container port
```

For example:

```text
3000:3000
```

means that host port `3000` is forwarded to container port `3000`.

A published port may be reachable through more interfaces than intended. Docker
manages firewall and NAT rules for container networking, and published ports
can bypass assumptions based only on ordinary UFW rules.

Therefore:

- Do not assume a UFW rule alone describes all Docker exposure.
- Review published ports with `sudo docker ps` and `sudo docker port`.
- Test access from the LAN and Tailscale client paths.
- Avoid publishing ports that are not required.
- Do not publish database or administration ports without a documented need.
- Consider Docker's `DOCKER-USER` firewall chain for stricter filtering.
- Keep application-specific port decisions in the service modules.

Inspect published ports:

```bash
sudo docker ps --format 'table {{.Names}}\t{{.Ports}}\t{{.Status}}'
```

Inspect a specific container later:

```bash
sudo docker port <CONTAINER_NAME>
```

At the end of this module, no application containers are required. Application
ports are introduced when Gitea and Pi-hole are deployed.

## Docker Service Lifecycle

The Docker daemon should start at boot:

```bash
systemctl is-enabled docker
```

Container restart behavior is controlled by each Compose project's restart
policy. A running Docker daemon does not guarantee that every application is
healthy.

Later service modules should verify:

- The Compose file is valid.
- The container starts.
- The restart policy is appropriate.
- Persistent directories are mounted.
- Published ports are intentional.
- Health checks and logs are available.
- The service recovers after a reboot.

## Maintenance Baseline

Review package updates:

```bash
sudo apt update
apt list --upgradable
```

Check Docker versions:

```bash
docker --version
docker compose version
```

Check daemon status:

```bash
sudo systemctl status docker --no-pager
```

Check disk use:

```bash
df -h /srv
sudo docker system df
```

View recent Docker logs:

```bash
sudo journalctl -u docker --no-pager -n 100
```

Do not update application images from this module. Gitea and Pi-hole image
updates are documented with their service-specific procedures.

Before Docker or kernel updates:

1. Confirm that current service data is backed up.
2. Review available disk space.
3. Record the current container and image state.
4. Apply updates during a planned maintenance window.
5. Verify application health afterward.

## Verification Checklist

| Check | Expected result |
|---|---|
| Docker repository | Official repository is configured |
| Docker Engine | Package is installed |
| Docker CLI | `docker --version` succeeds |
| Compose plugin | `docker compose version` succeeds |
| Docker service | Service is active |
| Docker startup | Service is enabled at boot |
| Container runtime | `containerd` is active |
| Test container | `hello-world` completes successfully |
| Docker daemon | `sudo docker info` succeeds |
| Privilege model | Docker commands require explicit `sudo` |
| Docker group access | Administrative user is not in the `docker` group |
| Storage directories | `/srv/docker` and `/srv/files` exist |
| Service directories | Gitea and Pi-hole parent directories exist |
| Directory ownership | Docker configuration directories are root-owned |
| Disk capacity | Adequate free space is available |
| Docker networking | Default networks are visible |
| Published ports | No unplanned application ports are exposed |
| Firewall relationship | Docker and UFW interaction is understood |

## Common Problems

### Docker package installation fails

Check package metadata and repository configuration:

```bash
sudo apt update
apt-cache policy docker-ce
```

Inspect the repository file:

```bash
cat /etc/apt/sources.list.d/docker.list
```

Confirm that the Ubuntu release and architecture match the repository.

Check the keyring:

```bash
ls -l /etc/apt/keyrings/docker.asc
```

Do not bypass package signature verification or install Docker packages from
untrusted repositories.

### The Docker service is not active

Check the service:

```bash
sudo systemctl status docker --no-pager
```

Inspect recent logs:

```bash
sudo journalctl -u docker --no-pager -n 100
```

Check the container runtime:

```bash
sudo systemctl status containerd --no-pager
```

Common causes include invalid daemon configuration, insufficient storage,
container runtime failure, package problems, and filesystem errors.

### Docker reports permission denied

If this command fails:

```bash
docker ps
```

Use explicit privilege:

```bash
sudo docker ps
```

Do not change the Docker socket permissions or add the user to the Docker group
just to avoid using `sudo`.

### `docker compose` is not available

Check the plugin:

```bash
docker compose version
```

Check installed packages:

```bash
apt list --installed 2>/dev/null | grep docker
```

The expected package is:

```text
docker-compose-plugin
```

Use `docker compose`, not the legacy `docker-compose`, for this project.

### The test image cannot be downloaded

Check network routing and DNS:

```bash
ip route
resolvectl status
resolvectl query registry-1.docker.io
```

Check Docker service logs:

```bash
sudo journalctl -u docker --no-pager -n 100
```

Test the image pull explicitly:

```bash
sudo docker pull hello-world
```

Possible causes include DNS failure, no default route, upstream connectivity
failure, proxy requirements, registry availability, and firewall restrictions.

### Docker runs out of disk space

Check filesystem capacity:

```bash
df -h
```

Check Docker usage:

```bash
sudo docker system df
```

Find large directories:

```bash
sudo du -sh /var/lib/docker /srv/docker /srv/files
```

Do not remove volumes or application data without identifying their owners and
confirming that backups exist.

### A container's port is reachable unexpectedly

List published ports:

```bash
sudo docker ps --format 'table {{.Names}}\t{{.Ports}}\t{{.Status}}'
```

Inspect the relevant Compose file and Docker network configuration.

Review UFW:

```bash
sudo ufw status verbose
```

Docker-published ports may not behave like ordinary host services under UFW.
Use the Docker `DOCKER-USER` chain or a more restrictive binding and firewall
design when required.

### A container starts and immediately exits

List all containers:

```bash
sudo docker ps -a
```

Inspect the container state:

```bash
sudo docker inspect <CONTAINER_NAME>
```

View its logs:

```bash
sudo docker logs --tail=200 <CONTAINER_NAME>
```

Check the exit code, image architecture, required environment variables,
mounted directory permissions, port conflicts, configuration syntax, and disk
space.

Application-specific recovery procedures belong in the relevant service
module.

### Docker does not recover correctly after reboot

Check the daemon:

```bash
sudo systemctl status docker --no-pager
```

List containers:

```bash
sudo docker ps -a
```

Inspect each Compose project's restart policy and service status. A container
must have an appropriate restart policy to start automatically after the
Docker daemon restarts.

Do not assume that a container being present in `sudo docker ps -a` means that
it is healthy.

## Security Considerations

- Install Docker packages from the official repository.
- Keep Docker and Ubuntu packages updated.
- Do not add routine administrative users to the `docker` group.
- Keep the Docker socket restricted to its default privileged access.
- Use `sudo` explicitly for Docker daemon operations.
- Do not expose the Docker API socket over TCP.
- Do not make `/var/run/docker.sock` world-readable.
- Do not run privileged containers without a documented requirement.
- Do not mount the host root filesystem into containers without a documented
  requirement.
- Publish only required application ports.
- Review Docker-published ports separately from UFW rules.
- Keep application data under documented persistent directories.
- Back up `/srv/docker` and `/srv/files` using a separate storage target.
- Do not store secrets directly in publicly published Compose files.
- Use service-specific permissions instead of recursively changing all of
  `/srv`.

## Evidence To Capture

Capture sanitized evidence showing:

- The Ubuntu release and architecture.
- The configured Docker repository without credentials.
- Installed Docker and Compose versions.
- Docker and containerd service status.
- Successful `hello-world` execution.
- The administrative user's group membership without exposing personal details.
- The explicit-`sudo` Docker privilege model.
- The `/srv/docker` and `/srv/files` directory layout.
- Root ownership of Docker configuration directories.
- Available filesystem capacity.
- Docker disk-usage output.
- Docker network list.
- Published-port output after later services are deployed.
- The relationship between Docker networking and UFW.

Remove or replace:

- Usernames.
- Hostnames.
- Live IP addresses.
- Registry credentials.
- Private repository URLs.
- Secrets from environment variables.
- Unsanitized logs.
- Host-specific filesystem details that are not needed.

## Completion Criteria

This module is complete when:

- The official Docker repository is configured.
- Docker Engine is installed.
- The Docker service is active and enabled at boot.
- containerd is active.
- Docker Compose is available through `docker compose`.
- The `hello-world` test completes successfully.
- Docker daemon access and its security implications are understood.
- The administrative user is not in the `docker` group.
- Docker administration uses explicit `sudo`.
- `/srv/docker` and `/srv/files` exist.
- Gitea and Pi-hole parent directories are prepared.
- Docker configuration directories are root-owned.
- Available disk capacity has been recorded.
- Docker networks and storage usage have been reviewed.
- Published-port and UFW interactions are understood.
- Sanitized evidence has been captured.

Next: [Samba File Service](07-samba-file-service.md)
