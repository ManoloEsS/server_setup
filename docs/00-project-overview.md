# Project Overview

## Project At A Glance

| Item | Description |
|---|---|
| Project type | Single-server homelab and IT support portfolio |
| Hardware | Small-form-factor PC or mini PC |
| Operating system | Ubuntu Server 24.04 LTS |
| Core services | Samba, Gitea, Pi-hole, Docker, SSH, and Tailscale |
| Local network | Private IPv4 LAN with DHCP reservation |
| Remote access | Tailscale encrypted network |
| Deployment tools | `apt`, `systemd`, Docker Compose, and configuration files |
| Primary audience | Entry-level IT support and junior systems roles |

## Project Context

The goal of this project is to build, operate, troubleshoot, and document a small self-hosted server environment.

The server provides shared files, private Git hosting, and network-level DNS filtering. It is administered remotely through SSH and Tailscale. The surrounding network includes an upstream gateway and a primary home router that provides LAN connectivity, DHCP, and client DNS configuration.

This project is intentionally presented as a homelab rather than production infrastructure. The value of the project is the documented process: planning the environment, applying changes safely, verifying behavior, diagnosing failures, and communicating the results clearly.

## Objectives

This project is designed to demonstrate the ability to:

- Install and configure a Linux server from a clean system.
- Use package management, users, groups, permissions, storage, and systemd.
- Administer a server through SSH.
- Plan a small IPv4 network and identify the role of each device.
- Understand DHCP, DNS, routing, NAT, ports, and firewall rules.
- Deploy services with Docker Compose.
- Configure file sharing, Git hosting, and DNS filtering.
- Test services from both local and remote clients.
- Investigate failures using logs, status commands, and network diagnostics.
- Document changes, results, limitations, and recovery steps.

## Scope

### Core scope

The core documentation covers:

- Server hardware and installation assumptions.
- Ubuntu Server installation.
- Linux system baseline and administrative access.
- SSH and firewall configuration.
- Local network addressing and router configuration.
- DNS resolution and Pi-hole integration.
- Tailscale remote access.
- Docker and Docker Compose.
- Samba file sharing.
- Gitea Git hosting.
- Service validation and acceptance testing.
- Maintenance, backups, recovery, and troubleshooting.

### Optional scope

The following topics are useful home-lab extensions but are not required to understand the core server:

- Syncthing device and folder synchronization.
- New-client onboarding.
- Remote workstation file access.
- Personal desktop, editor, and development-tool configuration.

### Out of scope

This project does not attempt to provide:

- High availability or automatic failover.
- Enterprise identity management.
- Production-grade monitoring and alerting.
- Redundant storage.
- A public-facing internet service.
- A production service-level agreement.
- A replacement for vendor documentation.

## Architecture Summary

```text
Internet
   |
Upstream gateway
   |
Primary home router
   |
Private LAN: 192.168.1.0/24
   |
   +-- Server
   |     +-- Ubuntu Server
   |     +-- SSH
   |     +-- Docker
   |     +-- Samba
   |     +-- Gitea
   |     +-- Pi-hole
   |
   +-- Local client devices

Remote client
   |
Tailscale encrypted network
   |
Server
```

The upstream gateway and primary router create a double-NAT environment. This is accepted for the lab because the upstream gateway supports other household equipment. Tailscale provides remote access without requiring inbound port forwarding on either router.

The detailed addressing, port, dependency, and traffic-flow plan is documented in [Architecture and Network Plan](01-architecture-and-network-plan.md).

## Design Decisions

| Decision | Reason | Tradeoff |
|---|---|---|
| Ubuntu Server LTS | Stable, widely documented, and suitable for learning Linux administration | Some procedures may differ on other distributions |
| Wired server connection | More reliable for file sharing, DNS, and service access | Requires physical network cabling |
| DHCP reservation | Keeps the server address stable while retaining centralized address management | Depends on the router's DHCP service |
| Docker Compose | Separates application services and makes deployments repeatable | Adds container, volume, and networking concepts |
| Tailscale for remote access | Avoids router port forwarding and exposes services only to authorized devices | Depends on the Tailscale client and account |
| Pi-hole for DNS filtering | Demonstrates DNS resolution, client configuration, and service dependencies | DNS failure can affect the entire network |
| Samba and Gitea as separate services | Demonstrates multiple service types and access models | Requires separate authentication and troubleshooting |
| Sanitized examples | Keeps configurations understandable without exposing sensitive infrastructure details | Requires placeholders and careful redaction |

## Skills And Evidence

| Competency | Evidence to capture |
|---|---|
| Linux administration | Installation notes, package and service status, permissions, storage checks, and logs |
| Networking | Sanitized topology, address plan, routing information, DNS tests, port checks, and firewall rules |
| Service management | Successful starts, restarts, health checks, configuration validation, and reboot testing |
| Container administration | Docker installation, Compose configuration, volume layout, container status, and logs |
| Security | SSH key use, UFW policy, restricted service access, and secret-handling decisions |
| Troubleshooting | Incident-style records showing symptom, hypothesis, test, fix, and verification |
| Operations | Update procedure, backup plan, restore test, maintenance schedule, and documented limitations |
| Technical communication | Clear prerequisites, expected results, warnings, escalation points, and change summaries |

Evidence must be sanitized before publication. Live passwords, private keys, personal identifiers, device IDs, and unnecessary infrastructure details must be removed from examples, logs, and screenshots.

## Success Criteria

The project is considered successful when:

- A reader can understand the architecture and purpose without access to the live system.
- A new administrator can follow the core modules in dependency order.
- The server can be administered locally and remotely through documented access methods.
- Firewall rules expose only the services required by the lab.
- Each service has a documented health check and client-side test.
- The services recover or can be restored after a server reboot.
- Backup and recovery procedures are documented and tested.
- Common failures have reproducible diagnostic procedures.
- The documentation identifies risks and limitations honestly.

## Limitations And Future Improvements

The current design uses one server and one primary storage location. A hardware failure, disk failure, router failure, or DNS service failure can affect multiple services at once.

Possible future improvements include:

- Automated backups to separate storage.
- Regular restore testing.
- Disk health monitoring.
- Centralized logs and alerting.
- More restrictive network segmentation.
- Separate databases for services that outgrow SQLite.
- Infrastructure-as-code or repeatable provisioning scripts.
- A documented recovery test from a clean system.

These improvements are intentionally separated from the initial entry-level scope.

## Documentation Status

This project is organized as modular documentation. Each module can be reviewed, validated, and maintained independently.

Examples use sanitized placeholders, and sensitive values are omitted from the documentation.
