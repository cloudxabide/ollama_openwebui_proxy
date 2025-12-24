# HAProxy in Front of AI Nodes

This repository documents an approach for implementing HAProxy as a reverse proxy in front of AI nodes (Ollama/OpenWebUI), enabling secure, authenticated access from multiple networks with a consistent interface.

![Cartoon](Images/high-level-cartoon-by_Gemini.png)

## The Problem

I run several AI nodes in my homelab, each hosting Ollama (for LLM inference) and OpenWebUI (for browser-based interaction). While I have VPN access to my environment, I wanted a more elegant solution for command-line and programmatic access that works the same way whether I'm on my home network, another internal network, or accessing from the internet.

**The challenge**: How do you provide unified, secure access to multiple AI backends without maintaining different connection methods for different network contexts?

## The Solution: HAProxy as an AI Gateway

HAProxy acts as a single entry point that provides:

**Hostname-based routing** - Direct traffic to different AI nodes based on the requested hostname
**SSL/TLS termination** - Handle certificate management in one place rather than on each AI node
**Authentication layer** - Add auth to Ollama (which doesn't provide it natively)
**Consistent access pattern** - Same API endpoints work from anywhere

## How It Works

### Network Architecture

Traffic flow through the architecture:

1. **External requests** hit a border gateway/firewall at standard ports (11434 for Ollama API, 12000 for OpenWebUI)
2. **DNAT rules** forward these ports to the HAProxy node's Virtual IP (VIP)
3. **HAProxy** terminates SSL and routes based on hostname to the appropriate backend AI node
4. **Backend nodes** receive proxied requests on their internal network interfaces

![Network Overview](Images/high-level-ollama_openwebui.png)

### Dual Hostname Pattern

Each AI node has two hostnames serving different access patterns:

**Direct access** (internal only):
- `hal9000.kubernerdes.com` - Direct connection to the node's primary interface
- `brainiac.kubernerdes.com` - Direct connection to another AI node
- `kitt.kubernerdes.com` - Direct connection to a third AI node

**Proxied access** (works from anywhere):
- `hal9000-api.kubernerdes.com` - Routes through HAProxy (internal or internet)
- `brainiac-api.kubernerdes.com` - Routes through HAProxy
- `kitt-api.kubernerdes.com` - Routes through HAProxy

This pattern means your code and CLI tools use the same `-api` hostname regardless of where you're connecting from. HAProxy handles the routing complexity.

### Example: Accessing Ollama

From anywhere (home network, remote network, or internet):
```bash
# Same command works from any network
curl https://hal9000-api.kubernerdes.com:11434/api/generate \
  -d '{"model": "llama2", "prompt": "Hello"}'
```

HAProxy receives the request, validates SSL, checks authentication, and forwards to the actual hal9000 node.

## Key Components

**HAProxy** - Reverse proxy providing hostname-based routing and SSL termination
**Keepalived** - Manages a Virtual IP (VIP) for potential future high-availability setup
**Let's Encrypt** - Automated SSL/TLS certificates via DNS-01 challenge (Route53)
**Border Firewall** - DNAT rules forward public traffic to the HAProxy VIP

## Design Decisions

**Why a VIP with Keepalived?**
While you could bind HAProxy directly to the node's primary IP, using a separate VIP managed by Keepalived provides architectural flexibility. If you later want high availability, you can add a second proxy node that shares the same VIP - Keepalived handles the failover automatically.

**Why SSL termination at the proxy?**
Rather than managing certificates on each AI node, centralizing SSL at HAProxy means one place to handle certificate renewals and one place to configure TLS settings. The internal network traffic between HAProxy and backends can remain unencrypted since it never leaves your trusted network.

**Why hostname-based routing?**
Each AI node may have different capabilities (different GPUs, different models loaded, different performance characteristics). Hostname routing lets you direct requests to specific nodes while presenting a unified API surface to clients.

## What's Covered

- [Proxy Setup](Proxy_Setup.md) - HAProxy and Keepalived configuration concepts
- [SSL/TLS Setup](LetsEncryptYall.md) - Let's Encrypt certificate workflow

## Future Enhancements

**FAIL2BAN integration** - Monitor for repeated failed authentication attempts and temporarily block offending IPs






