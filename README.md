# Self-Hosted Server Lab

A documented homelab project demonstrating Linux server administration, networking, service deployment, security, troubleshooting, and operational support.

## Project Summary

This project configures a small computer as a self-hosted Ubuntu Server platform. The server provides:

- Samba network file sharing
- Gitea Git hosting
- Pi-hole DNS filtering
- Docker-based service isolation
- SSH administration
- Tailscale remote access
- Network-level firewall and DNS configuration

The documentation is written as both a rebuildable technical reference and a portfolio of entry-level IT support skills.

## Project Objectives

- Install and administer Ubuntu Server.
- Configure secure remote administration with SSH.
- Apply updates, firewall rules, users, permissions, and service management.
- Design and document a small IPv4 network.
- Configure DHCP reservations, DNS, routing, and NAT considerations.
- Deploy and maintain services with Docker Compose.
- Diagnose service, connectivity, authentication, and name-resolution problems.
- Document changes, verification steps, limitations, and recovery procedures.

## Architecture

```text
Internet
   |
Home gateway/router
   |
LAN: 192.168.1.0/24
   |
   +-- Server: <SERVER_LAN_IP>
   |     +-- Samba file service
   |     +-- Gitea Git service
   |     +-- Pi-hole DNS service
   |     +-- Docker
   |     +-- SSH
   |
   +-- Client devices

Remote clients
   |
Tailscale encrypted network
   |
Server: <SERVER_TAILSCALE_IP>
```

The documentation uses placeholders instead of real addresses, credentials, device IDs, or other sensitive infrastructure details.

## Skills Demonstrated

| Area | Demonstrated skills |
|---|---|
| Linux administration | Ubuntu installation, packages, users, permissions, systemd, storage, logs, and SSH |
| Networking | IPv4 addressing, DHCP, DNS, routing, NAT, ports, firewall rules, and VPN connectivity |
| Service support | Samba, Gitea, Pi-hole, Docker Compose, health checks, and logs |
| Security | Least-privilege access, SSH keys, UFW, restricted service exposure, and secret handling |
| Troubleshooting | Reproducing symptoms, testing hypotheses, reading logs, applying fixes, and verifying results |
| Operations | Updates, backups, recovery procedures, maintenance, documentation, and escalation boundaries |

## Documentation Path

Read the core modules in this order:

1. [Project Overview](docs/00-project-overview.md)
2. [Architecture and Network Plan](docs/01-architecture-and-network-plan.md)
3. [Ubuntu Server Installation](docs/02-install-ubuntu-server.md)
4. [Linux Baseline, Access, and Firewall](docs/03-linux-baseline-access-and-firewall.md)
5. [Networking, DNS, and Router Configuration](docs/04-networking-dns-and-router.md)
6. [SSH and Tailscale](docs/05-ssh-and-tailscale.md)
7. [Docker Host](docs/06-docker-host.md)
8. [Samba File Service](docs/07-samba-file-service.md)
9. [Gitea Git Service](docs/08-gitea-git-service.md)
10. [Pi-hole DNS Service](docs/09-pihole-dns-service.md)
11. [Validation and Acceptance Tests](docs/10-validation-and-acceptance-tests.md)
12. [Operations, Backups, and Recovery](docs/11-operations-backups-and-recovery.md)
13. [Troubleshooting Playbooks](docs/12-troubleshooting-playbooks.md)

Optional home-lab extensions are documented separately:

- [Optional Home-Lab Extensions](docs/13-optional-home-lab-extensions.md)

## Evidence Approach

Each module records:

- The problem or support scenario.
- The technical objective.
- The implementation decisions.
- The commands or configuration used.
- The expected verification results.
- Common failure modes and diagnostic steps.
- Security and recovery considerations.
- Sanitized evidence suitable for a public portfolio.

## Scope and Limitations

This is a single-server homelab project, not a production environment. It does not provide high availability, enterprise monitoring, redundant storage, or a production service-level agreement.

The documentation will identify these limitations and explain possible future improvements rather than presenting the lab as production-ready.

## Privacy and Security

This project must not contain:

- Passwords or private keys.
- Real router credentials.
- Personal usernames or email addresses.
- Tailscale device IDs.
- Unnecessary live IP addresses or hostnames.
- Unsanitized logs or screenshots.

Use placeholders for live values and remove sensitive information before sharing documentation.

## Project Status

This project is maintained as a modular documentation set. Core procedures are validated and updated as each module is completed.
