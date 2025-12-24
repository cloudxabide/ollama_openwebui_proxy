# Proxy Node Architecture

This document explains the architectural decisions and configuration concepts for the HAProxy proxy node, including why we use a Virtual IP and how to handle the timing challenges between Keepalived and HAProxy during system startup.

![Network Overview](Images/proxy_host_with_vip.png)

## The Virtual IP Pattern

The proxy node uses **two IP addresses**:
- **Host IP** - The node's primary network interface address
- **Virtual IP (VIP)** - A floating IP managed by Keepalived

### Why Use a VIP?

You might wonder why we don't just bind HAProxy directly to the host's primary IP. While that would work, using a VIP provides architectural flexibility:

**High Availability Path**: If you later want to add a second proxy node for redundancy, both nodes can be configured to manage the same VIP. Keepalived handles failover automatically - if the primary node fails, the secondary takes over the VIP within seconds. Your clients never need to know which physical node is handling requests.

**Service Isolation**: The VIP can be moved between nodes without changing the host IP, making maintenance and upgrades cleaner. You can take a node offline for updates while the VIP fails over to another node.

**DNAT Stability**: Your border firewall DNAT rules point to the VIP, not a specific node. This means your firewall configuration never needs to change, even if you rebuild or replace proxy nodes.

For a single-node setup, Keepalived might seem like overkill, but it future-proofs the architecture at minimal cost.

## The Startup Timing Challenge

When the proxy node boots, there's a race condition between two services:

1. **Keepalived** starts up and assigns the VIP to the network interface
2. **HAProxy** starts up and tries to bind to the VIP address

If HAProxy starts before Keepalived has assigned the VIP, HAProxy's bind will fail because it's trying to attach to an IP address that doesn't yet exist on the system.

### The Solution: Non-Local Bind

Linux provides a kernel parameter that allows processes to bind to IP addresses that aren't currently configured on any interface: `net.ipv4.ip_nonlocal_bind`

When enabled, HAProxy can bind to the VIP address even if Keepalived hasn't assigned it yet. Once Keepalived brings up the VIP, traffic flows normally.

**Example configuration:**
```bash
# /etc/sysctl.d/10-haproxy.conf
net.ipv4.ip_nonlocal_bind = 1
```

Apply immediately without reboot:
```bash
sudo sysctl -p /etc/sysctl.d/10-haproxy.conf
```

This setting essentially tells the kernel: "Trust that this IP will exist - let the process bind to it now."

## Architecture Components

**HAProxy** - Listens on the VIP at ports 11434 (Ollama) and 12000 (OpenWebUI), performs hostname-based routing to backend AI nodes, terminates SSL/TLS connections, and validates authentication.

**Keepalived** - Manages the VIP assignment, monitors the health of the proxy node, and provides VRRP (Virtual Router Redundancy Protocol) for potential multi-node setups.

**Backend AI Nodes** - Individual nodes (hal9000, brainiac, kitt) running Ollama and OpenWebUI, receiving proxied traffic from HAProxy on their internal network interfaces.

## Configuration Principles

### Hostname-Based Routing

HAProxy uses the SNI (Server Name Indication) field from the TLS handshake to route before decryption, then validates against the hostname after SSL termination. This allows routing decisions based on the requested hostname (hal9000-api vs brainiac-api vs kitt-api).

### SSL Certificate Management

Each backend hostname needs its own certificate. HAProxy expects certificates in PEM format with both the certificate chain and private key concatenated into a single file:

```bash
# Example structure (not a command to run)
/etc/haproxy/certs/
├── hal9000-api.kubernerdes.com.pem
├── brainiac-api.kubernerdes.com.pem
└── kitt-api.kubernerdes.com.pem
```

HAProxy automatically selects the correct certificate based on the requested hostname.

### Backend Health Checks

HAProxy continuously monitors backend nodes to ensure they're responsive. If a backend becomes unavailable, HAProxy can return meaningful errors rather than timing out or routing to a dead node.

## Requirements Summary

- **Linux host** with administrative access
- **Two IP addresses**: One primary (host IP), one virtual (VIP)
- **Packages**: haproxy, keepalived
- **Network access**: Ability to route traffic between the VIP and backend nodes
- **DNS**: Entries for both direct and -api hostnames pointing to appropriate IPs
