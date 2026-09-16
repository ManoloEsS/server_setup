# Architecture And Network Plan

## Purpose

This document defines the network design for the self-hosted server environment.

It explains:

- The role of each network device.
- How local and remote clients reach the server.
- How addressing, DHCP, DNS, routing, and NAT interact.
- Which services are exposed and on which ports.
- Where firewall and trust boundaries exist.
- What evidence will demonstrate networking knowledge.

## Network Goals

The network must:

- Provide reliable connectivity for the server and local clients.
- Give the server a stable local address.
- Allow clients to resolve DNS through Pi-hole.
- Allow authorized remote clients to reach services through Tailscale.
- Avoid exposing server services directly to the public internet.
- Make service dependencies and failure impact understandable.
- Support clear troubleshooting from the client, router, server, and service
  layers.

## Topology

```text
                              Internet
                                  |
                                  |
                       Upstream gateway/router
                       Network: 10.0.0.0/24
                                  |
                       WAN connection to primary router
                                  |
                         Primary home router
                         LAN: 192.168.1.0/24
                         DHCP, routing, and Wi-Fi
                                  |
              +-------------------+-------------------+
              |                   |                   |
       Ubuntu server       Local client devices   Other LAN devices
       <SERVER_LAN_IP>     DHCP-assigned IPs
              |
       +------+--------+----------------+
       |               |                |
     Samba           Gitea           Pi-hole
   file sharing    Git hosting      DNS filtering

                    Tailscale overlay
                           |
                    Authorized remote clients
                           |
                    <SERVER_TAILSCALE_IP>
```

The upstream gateway and primary router create a double-NAT topology. This is
acceptable for this lab because the upstream gateway supports other network
equipment. Tailscale provides remote access without requiring inbound port
forwarding.

## Device Roles

| Device           | Primary role                 | Addressing                            | Responsibilities                                               |
| ---------------- | ---------------------------- | ------------------------------------- | -------------------------------------------------------------- |
| Upstream gateway | Internet gateway             | Upstream private network              | Internet access, upstream NAT, and provider-specific equipment |
| Primary router   | LAN router                   | LAN gateway at `<ROUTER_LAN_IP>`      | DHCP, LAN routing, Wi-Fi, and client DNS distribution          |
| Ubuntu server    | Service host                 | DHCP reservation at `<SERVER_LAN_IP>` | SSH, Docker, Samba, Gitea, Pi-hole, and Tailscale              |
| Local clients    | Service consumers            | DHCP-assigned LAN addresses           | Access file shares, Git services, and DNS                      |
| Remote clients   | Authorized service consumers | Tailscale addresses                   | Access selected services through the encrypted overlay         |

## Addressing Plan

| Network or device        | Example value           | Purpose                                                 |
| ------------------------ | ----------------------- | ------------------------------------------------------- |
| Upstream network         | `10.0.0.0/24`           | Network between the upstream gateway and primary router |
| Primary router LAN       | `192.168.1.1`           | Default gateway for local clients                       |
| Local LAN                | `192.168.1.0/24`        | Private network for the server and local clients        |
| Server LAN address       | `192.168.1.100`         | Stable address supplied through a DHCP reservation      |
| Tailscale network        | `100.64.0.0/10`         | Overlay network for authorized devices                  |
| Server Tailscale address | `<SERVER_TAILSCALE_IP>` | Remote address for the server                           |

The server uses a DHCP reservation rather than a manually configured static
address. The router remains the source of address allocation while ensuring that
the server receives the same address after reboots.

The address plan must be updated consistently if the LAN subnet or server
reservation changes.

## Traffic Flows

### Local service access

```text
Local client
    |
    | TCP/UDP service request
    v
Primary router LAN
    |
    v
Server LAN address
    |
    +-- Samba
    +-- Gitea
    +-- Pi-hole
```

Local clients reach the server directly through the LAN. The primary router does
not need port forwarding for local traffic.

### Remote service access

```text
Remote client
    |
    | Tailscale encrypted connection
    v
Tailscale overlay
    |
    v
Server Tailscale address
    |
    +-- SSH
    +-- Gitea
    +-- Samba
    +-- Pi-hole DNS
```

Remote access uses Tailscale rather than exposing service ports on the upstream
gateway or primary router. Both endpoints must be connected to the same
authorized Tailscale network.

### DNS resolution

```text
Local client
    |
    | DNS request
    v
Primary router DHCP configuration
    |
    | Pi-hole server address
    v
Pi-hole
    |
    | Upstream DNS request
    v
Configured upstream DNS providers
```

The server itself uses the local Pi-hole service for DNS resolution. Remote
Tailscale clients can use the server's Tailscale address as their DNS nameserver
when Tailscale DNS is configured.

DNS is a critical dependency. If Pi-hole is unavailable, clients may lose name
resolution even when basic IP connectivity still works.

## Service Port Plan

| Service               | Protocol and port     | Intended access                            | Purpose                         |
| --------------------- | --------------------- | ------------------------------------------ | ------------------------------- |
| SSH                   | TCP 22                | Local LAN and Tailscale                    | Server administration           |
| Samba                 | TCP 445               | Local LAN and authorized remote clients    | File sharing                    |
| Samba legacy session  | TCP 139               | Only if required by the client environment | Legacy SMB compatibility        |
| Gitea web interface   | TCP 3000              | Local LAN and Tailscale                    | Repository management           |
| Gitea SSH             | TCP 222               | Local LAN and Tailscale                    | Git operations over SSH         |
| DNS                   | UDP and TCP 53        | Local LAN and Tailscale                    | Name resolution through Pi-hole |
| Pi-hole web interface | TCP 80                | Administrative clients                     | DNS filtering administration    |
| Tailscale             | Overlay-managed ports | Authorized Tailscale devices               | Encrypted remote connectivity   |

The exact firewall rules are documented in
[Linux Baseline, Access, and Firewall](03-linux-baseline-access-and-firewall.md).

No service in this design requires a public router port forward.

## Trust Boundaries

| Boundary                           | Control                              | Reason                                               |
| ---------------------------------- | ------------------------------------ | ---------------------------------------------------- |
| Internet to upstream gateway       | Stateful router firewall             | Blocks unsolicited inbound traffic                   |
| Upstream gateway to primary router | Double-NAT boundary                  | Prevents direct exposure of the downstream LAN       |
| Primary router to LAN              | Router firewall and Wi-Fi security   | Controls local network access                        |
| LAN to server                      | UFW and service authentication       | Limits access to required services                   |
| Tailscale network to server        | Tailscale identity and host firewall | Limits remote access to authorized devices           |
| Service to service data            | Docker volumes and file permissions  | Separates application data and administrative access |

The server firewall is a second layer of control. Router isolation should not be
treated as a replacement for host-level firewall rules.

## Service Dependencies

```text
Internet connectivity
    |
    +-- Primary router
          |
          +-- Stable server address
          |     |
          |     +-- SSH administration
          |     +-- Docker
          |           |
          |           +-- Gitea
          |           +-- Pi-hole
          |
          +-- DHCP client configuration
                |
                +-- Pi-hole DNS address
```

Important dependencies include:

- The server requires a working LAN connection before any service can be
  reached.
- Pi-hole requires a free host port 53 and working upstream DNS.
- Gitea and Pi-hole require Docker and persistent data directories.
- Samba requires valid filesystem permissions and Samba authentication.
- Remote access requires Tailscale on both the client and server.
- Client-wide DNS filtering requires the router or Tailscale client to
  distribute Pi-hole as the DNS server.

## Failure Impact

| Failure                    | Expected impact                                       | First diagnostic area                          |
| -------------------------- | ----------------------------------------------------- | ---------------------------------------------- |
| Server powered off         | All hosted services unavailable                       | Hardware, power, and local connectivity        |
| DHCP reservation fails     | Server address may change                             | Router lease and address reservation           |
| UFW blocks a service       | Service may work locally but fail from clients        | Firewall rules and listening ports             |
| Pi-hole stops              | DNS filtering and client name resolution may fail     | Container health and port 53 ownership         |
| Docker stops               | Gitea and Pi-hole become unavailable                  | Docker service and container status            |
| Tailscale disconnects      | Remote access and remote DNS fail                     | Tailscale status and client connectivity       |
| Samba authentication fails | File share is reachable but access is denied          | Samba account, permissions, and configuration  |
| Primary router fails       | Local routing, DHCP, Wi-Fi, and DNS distribution fail | Router status and client address configuration |

## Design Decisions

### DHCP reservation instead of manual server addressing

A DHCP reservation centralizes address management in the router while providing
a predictable address for services that clients need to find.

### No public port forwarding

Directly forwarding SSH, Samba, Gitea, or DNS from the internet would increase
exposure and create additional security requirements. Tailscale provides an
authorized overlay for remote access instead.

### Pi-hole as a local DNS service

Centralizing DNS filtering at Pi-hole allows multiple clients to use the same
filtering policy. It also creates an important troubleshooting scenario because
DNS failure can affect many devices at once.

### Host firewall plus router firewall

The router controls the network boundary, while UFW protects the server
independently. This provides defense in depth and makes service exposure
explicit.

### Double NAT accepted as a lab constraint

Double NAT can complicate inbound connections and network troubleshooting, but
it is retained because the upstream gateway supports other equipment. Remote
access is designed around Tailscale rather than router port forwarding.

## Validation Evidence

The completed project should capture sanitized evidence for:

- The final topology diagram.
- The server's local address and default route.
- The router's DHCP reservation.
- The server's listening ports.
- The UFW policy and allowed interfaces.
- Successful local DNS resolution.
- Successful remote Tailscale connectivity.
- Successful client access to each service.
- A documented failure and recovery for at least one network-related issue.

Useful validation commands will be introduced in the implementation modules and
consolidated in
[Validation and Acceptance Tests](10-validation-and-acceptance-tests.md).

## Network Limitations

This design depends on:

- A functioning upstream internet connection.
- A functioning primary router.
- A single server and network interface.
- Correct DHCP and DNS configuration.
- Tailscale availability for remote access.
- Service-specific authentication and firewall rules.

The design is appropriate for a controlled homelab. It should not be treated as
a complete enterprise network architecture.
