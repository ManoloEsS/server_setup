# Networking, DNS, And Router Configuration

## Purpose

This module documents the local network configuration required for the server
and its clients.

It covers:

- IPv4 addressing and interface identification.
- DHCP reservations.
- Default gateways and routing.
- Router-side DNS configuration.
- Linux resolver behavior.
- Pi-hole's role in the local DNS path.
- Double-NAT considerations.
- Network and DNS troubleshooting.

Tailscale remote access and remote DNS configuration are covered in the next
modules.

## Learning Objectives

After completing this module, an administrator should be able to:

- Identify the server's active network interface.
- Explain the roles of a gateway, DHCP server, DNS server, and router.
- Use a DHCP reservation to provide a stable server address.
- Inspect Linux addresses, routes, and resolver settings.
- Configure clients to use an internal DNS service.
- Distinguish network connectivity failures from DNS failures.
- Explain how double NAT affects the design.
- Validate network changes from both the server and client sides.

## Network Assumptions

| Item                     | Example value                |
| ------------------------ | ---------------------------- |
| Upstream gateway network | `10.0.0.0/24`                |
| Primary router LAN       | `192.168.1.1`                |
| Local LAN                | `192.168.1.0/24`             |
| Server address           | `192.168.1.100`              |
| Server interface         | `<SERVER_INTERFACE>`         |
| Local DNS service        | Pi-hole at `<SERVER_LAN_IP>` |
| DHCP authority           | Primary router               |

Replace example values with the values used by the environment. Use placeholders
in published documentation and evidence.

## Network Roles

| Component        | Role                                                           |
| ---------------- | -------------------------------------------------------------- |
| Upstream gateway | Provides upstream internet access and may perform NAT          |
| Primary router   | Provides LAN routing, DHCP, Wi-Fi, and client network settings |
| Ubuntu server    | Hosts applications and the local DNS filtering service         |
| DHCP             | Assigns addresses, gateways, and DNS settings to clients       |
| DNS              | Resolves hostnames into IP addresses                           |
| Default gateway  | Routes traffic outside the local subnet                        |
| Pi-hole          | Filters DNS queries and forwards permitted queries upstream    |

A device can reach an IP address without being able to resolve hostnames.
Network connectivity and DNS resolution must be tested separately.

## Step 1: Identify the Server Network Configuration

Inspect the server's interfaces and addresses:

```bash
ip -br address
```

Inspect the routing table:

```bash
ip route
```

Inspect the interface hardware address:

```bash
ip link show <SERVER_INTERFACE>
```

Inspect the route selected for an external destination:

```bash
ip route get 1.1.1.1
```

Record the following values:

```text
Active interface: <SERVER_INTERFACE>
MAC address: <SERVER_MAC_ADDRESS>
Server address: <SERVER_LAN_IP>
Network prefix: <LAN_CIDR>
Default gateway: <ROUTER_LAN_IP>
```

The active interface should have an address in the local LAN range and a default
route through the primary router.

## Step 2: Configure a DHCP Reservation

Configure a DHCP reservation for the server in the primary router.

Use:

- The server's MAC address.
- A reserved address outside the router's ordinary dynamic allocation range when
  possible.
- A descriptive device name.
- The server's intended hostname.

Example:

```text
Device name: <SERVER_HOSTNAME>
MAC address: <SERVER_MAC_ADDRESS>
Reserved address: <SERVER_LAN_IP>
```

A DHCP reservation is preferred over manually assigning an address on the server
because address management remains centralized at the router.

After applying the reservation:

1. Restart the server's network connection or reboot the server.
2. Confirm that it receives the reserved address.
3. Confirm that the default route remains correct.
4. Confirm that the router shows the expected lease.

Verify from the server:

```bash
ip -br address
ip route
```

Verify from the router's DHCP client or lease table that the reservation is
active.

## Step 3: Validate Basic Routing

Test the local gateway:

```bash
ping -c 3 <ROUTER_LAN_IP>
```

Check the route to the gateway:

```bash
ip route get <ROUTER_LAN_IP>
```

Check the route to an external IP:

```bash
ip route get 1.1.1.1
```

If permitted by the network, test external IP connectivity:

```bash
ping -c 3 1.1.1.1
```

A failed ping does not always prove that routing is broken because some networks
block ICMP. Use the routing table and application-level tests as additional
evidence.

The expected route should include:

- The local LAN interface.
- The local source address.
- The primary router as the default gateway for external traffic.

## Step 4: Understand the DNS Path

Before Pi-hole is active, the server and clients use the DNS service supplied by
the router or another configured upstream resolver.

After Pi-hole is deployed, the intended local DNS path is:

```text
Local client
    |
    | DNS request
    v
Primary router DHCP settings
    |
    | Pi-hole address
    v
Pi-hole on the Ubuntu server
    |
    | Permitted query forwarding
    v
Configured upstream DNS provider
```

The router distributes the Pi-hole address to clients through DHCP. Clients
should not need individual manual DNS settings when they use the primary
router's DHCP service.

The server's own resolver configuration is handled separately because the server
hosts Pi-hole. The final server DNS configuration should be applied only after
Pi-hole is running and listening on port 53.

## Step 5: Inspect Linux Resolver State

Inspect the current resolver configuration:

```bash
resolvectl status
```

Query a known domain:

```bash
resolvectl query ubuntu.com
```

Check the resolver file:

```bash
ls -l /etc/resolv.conf
```

The resolver configuration must be consistent with the active network-management
method. Avoid manually editing `/etc/resolv.conf` if it is managed by
`systemd-resolved`, Netplan, NetworkManager, or systemd-networkd.

The final resolver path should be documented after Pi-hole is installed.
Depending on the design, the server may use:

- Pi-hole through `127.0.0.1`.
- Pi-hole through the server's LAN address.
- A resolver configuration managed by `systemd-resolved`.

The selected approach must leave only one clear owner for host port 53.

## Step 6: Configure Router DNS

After Pi-hole is installed and verified, configure the primary router's LAN or
DHCP DNS settings to provide:

```text
Primary DNS: <SERVER_LAN_IP>
```

Do not add a public DNS server as a secondary client DNS server when the
requirement is for all client queries to pass through Pi-hole. A client may use
the secondary resolver when the primary is slow or unavailable, bypassing
filtering.

The router itself may use a separate upstream DNS configuration depending on its
firmware and operating mode. The important client-side requirement is that DHCP
distributes the Pi-hole address.

After applying the router configuration:

1. Renew a client DHCP lease.
2. Confirm the client received the Pi-hole address.
3. Query a normal domain.
4. Query a known blocked test domain.
5. Confirm the query appears in the Pi-hole dashboard or logs.

## Step 7: Configure the Server Resolver After Pi-hole Deployment

This step is performed after the Pi-hole module has deployed and verified the
DNS service.

Enable `systemd-resolved` if required by the Ubuntu installation:

```bash
sudo systemctl enable --now systemd-resolved
```

Configure the resolver so that its local stub does not compete with Pi-hole for
port 53:

```bash
sudo mkdir -p /etc/systemd/resolved.conf.d
```

Create a resolver drop-in using the system's approved configuration method:

```ini
[Resolve]
DNS=127.0.0.1
DNSStubListener=no
```

Ensure `/etc/resolv.conf` points to the resolver file intended by the selected
design, then restart the resolver:

```bash
sudo systemctl restart systemd-resolved
```

Verify:

```bash
resolvectl status
resolvectl query ubuntu.com
sudo ss -lntup | grep ':53'
```

Pi-hole should be the intended listener for host port 53. If `systemd-resolved`
still owns the local stub port, resolve that conflict before continuing.

The exact persistent resolver configuration belongs in the Pi-hole service
module because it depends on the container being available.

## Step 8: Account for Double NAT

In this design:

```text
Internet
    |
Upstream gateway
    |
Primary router
    |
Local server and clients
```

The upstream gateway and primary router both perform routing and NAT. This
creates a double-NAT environment.

Expected implications include:

- Local LAN traffic continues to work normally.
- Router administration and troubleshooting involve two network devices.
- Inbound port forwarding is more complicated.
- Services should not depend on public inbound port forwarding.
- Tailscale is preferred for remote access.

Do not enable bridge mode or change the upstream gateway's operating mode unless
the network requirements have been reviewed. The chosen mode may support other
equipment or services that depend on the upstream gateway.

## Step 9: Verify Client Network Settings

On a Linux client, inspect the address, route, and resolver:

```bash
ip -br address
ip route
resolvectl status
```

On other operating systems, use the platform's network status tools.

Confirm:

- The client has an address in the expected LAN range.
- The default gateway is the primary router.
- The DNS server is `<SERVER_LAN_IP>` after Pi-hole is configured.
- The client is connected to the intended router rather than an isolated
  upstream network.

Test DNS resolution:

```bash
resolvectl query ubuntu.com
```

If available, inspect the DNS response path:

```bash
dig ubuntu.com
dig doubleclick.net
```

A blocked domain should return the response configured by Pi-hole, commonly
`0.0.0.0` or another local blocked response.

## DNS Troubleshooting Sequence

When a client cannot resolve a domain, test in this order:

1. Confirm the client has a valid IP address.
2. Confirm the client has a default route.
3. Ping or otherwise test the default gateway.
4. Test the server's LAN address.
5. Confirm the client received `<SERVER_LAN_IP>` as its DNS server.
6. Confirm Pi-hole is running.
7. Confirm port 53 is listening on the server.
8. Test DNS directly against the server.
9. Check UFW rules for UDP and TCP port 53.
10. Check Pi-hole logs and upstream resolver status.

Useful commands include:

```bash
ip -br address
ip route
resolvectl status
resolvectl query ubuntu.com
sudo ss -lntup | grep ':53'
sudo ufw status verbose
```

Test both UDP and TCP DNS when investigating unusual failures. DNS commonly uses
UDP, but TCP is required for some responses and fallback behavior.

## Failure Scenarios

| Symptom                             | Likely areas to inspect                                                    |
| ----------------------------------- | -------------------------------------------------------------------------- |
| Server address changes after reboot | DHCP reservation, MAC address, router lease                                |
| Server cannot reach the gateway     | Ethernet link, interface state, subnet, router port                        |
| Server reaches IPs but not domains  | Resolver configuration or DNS service                                      |
| Local clients use public DNS        | Router DHCP settings or stale client lease                                 |
| Pi-hole receives no client queries  | Router DNS setting, client resolver cache, wrong network                   |
| Pi-hole cannot start                | Port 53 conflict or container configuration                                |
| Only some clients bypass filtering  | Static DNS, encrypted DNS, VPN, private relay, or stale DHCP settings      |
| Internet fails after DNS changes    | Pi-hole health, upstream DNS, resolver loop, or firewall                   |
| Remote clients cannot use local DNS | Tailscale DNS configuration, overlay connectivity, or Pi-hole access rules |

## Verification Checklist

| Check                 | Expected result                                               |
| --------------------- | ------------------------------------------------------------- |
| Active interface      | Correct interface is up and has a LAN address                 |
| DHCP reservation      | Server receives the expected address after renewal or reboot  |
| Default route         | Traffic leaves through the primary router                     |
| Gateway connectivity  | Server can reach the router                                   |
| External connectivity | Server can reach an external test destination where permitted |
| Resolver status       | Linux resolver state is understandable and consistent         |
| Router DHCP DNS       | Clients receive the Pi-hole address after configuration       |
| Local DNS             | Clients can resolve permitted domains through Pi-hole         |
| Blocked DNS           | Test domains are blocked as expected                          |
| Port 53 ownership     | Pi-hole and the host resolver do not conflict                 |
| Double NAT            | Remote access does not depend on public port forwarding       |

## Evidence To Capture

Capture sanitized evidence showing:

- The topology and address plan.
- The server's active interface and route.
- The DHCP reservation configuration without exposing identifying hardware
  details.
- The server's resolver status.
- A client receiving the intended DNS server.
- Successful gateway, IP, and DNS tests.
- A blocked DNS query.
- A documented DNS failure and recovery process.

## Completion Criteria

This module is complete when:

- The server has a stable address through a DHCP reservation.
- The server's interface, route, and gateway are verified.
- The local DNS path is documented.
- The router distributes Pi-hole as the client DNS server.
- The server resolver configuration is compatible with Pi-hole.
- Clients can resolve permitted domains.
- Clients receive expected blocked responses for filtered domains.
- DNS and routing failure diagnostics are documented.
- Double-NAT limitations are understood and recorded.

Next: [SSH and Tailscale](05-ssh-and-tailscale.md)
