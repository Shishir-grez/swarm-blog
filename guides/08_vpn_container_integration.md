# VPN Integration with Containers: Complete Guide

## What You'll Learn
- Different approaches to VPN + container integration
- VPN on host vs VPN in container
- Tailscale-specific patterns
- Subnet routing
- Trade-offs of each approach

**Prerequisites**: Understanding of Docker networking (see [Part 5](05_docker_swarm_mode.md)) and VPNs (see [Part 4](04_tailscale_fundamentals.md))

> **Note**: This guide builds on concepts from Parts 4 and 5. Terms like containers, namespaces, VPNs, and network interfaces are explained there.

---

## Part 1: The Challenge

### The Problem
You want containers to communicate over a VPN (like Tailscale), but where should the VPN run?

```
Option 1: VPN on Host
Option 2: VPN in Container
Option 3: VPN in Sidecar
Option 4: Subnet Routing
```

Each has different trade-offs.

---

## Part 2: Approach 1 - VPN on Host

### Architecture
```
┌────────────────────────────────────┐
│           Host                     │
│                                    │
│  Tailscale (100.64.0.1)            │
│                                    │
│  ┌──────────────────────────────┐  │
│  │  Container 1 (172.17.0.2)    │  │
│  │  - Uses host network         │  │
│  │  - Or NAT through host       │  │
│  └──────────────────────────────┘  │
└────────────────────────────────────┘
```

### Implementation
```bash
# VPN runs on host
sudo tailscale up

# Container uses host network
docker run --network host myapp

# Or container uses bridge (NAT)
docker run -p 8080:80 myapp
```

### Pros
- ✅ Simple setup
- ✅ All containers share VPN
- ✅ Works with Docker Desktop (for outbound)

### Cons
- ❌ Can't bind to VPN IP from containers (Docker Desktop)
- ❌ No isolation (all containers use same VPN)
- ❌ Doesn't work for Docker Swarm advertise-addr

### Best For
- Simple use cases
- Outbound VPN access only
- Docker Desktop users (limited functionality)

---

## Part 3: Approach 2 - VPN in Container

### Architecture
```
┌────────────────────────────────────┐
│           Host                     │
│                                    │
│  ┌──────────────────────────────┐  │
│  │  Container                   │  │
│  │  - Tailscale daemon          │  │
│  │  - tailscale0 (100.64.0.1)   │  │
│  │  - Application               │  │
│  └──────────────────────────────┘  │
└────────────────────────────────────┘
```

### Implementation
```dockerfile
FROM ubuntu:22.04

# Install Tailscale
RUN curl -fsSL https://tailscale.com/install.sh | sh

# Install your app
COPY app /app

# Start script
COPY start.sh /start.sh
RUN chmod +x /start.sh

CMD ["/start.sh"]
```

```bash
#!/bin/bash
# start.sh

# Start Tailscale
tailscaled &
tailscale up --authkey=${TAILSCALE_AUTHKEY}

# Start your app
exec /app
```

### Running It
```bash
docker run -d \
  --name myapp \
  --cap-add=NET_ADMIN \
  --device=/dev/net/tun \
  -e TAILSCALE_AUTHKEY=tskey-xxx \
  myapp
```

### Pros
- ✅ Container has its own VPN identity
- ✅ Can bind to Tailscale IP
- ✅ Isolation between containers

### Cons
- ❌ Requires privileged capabilities
- ❌ Each container needs its own Tailscale node
- ❌ More complex setup

### Best For
- Containers that need their own VPN identity
- Microservices with separate VPN access
- When you need to bind to VPN IPs

---

## Part 4: Approach 3 - Sidecar Pattern

### Architecture
```
┌────────────────────────────────────┐
│           Pod/Compose              │
│                                    │
│  ┌──────────────┐  ┌────────────┐  │
│  │  Tailscale   │  │  App       │  │
│  │  Sidecar     │  │  Container │  │
│  │              │  │            │  │
│  │  100.64.0.1  │←→│  localhost │  │
│  └──────────────┘  └────────────┘  │
└────────────────────────────────────┘
```

### Docker Compose Example
```yaml
version: '3.8'

services:
  tailscale:
    image: tailscale/tailscale
    hostname: myapp-vpn
    environment:
      - TS_AUTHKEY=${TAILSCALE_AUTHKEY}
      - TS_STATE_DIR=/var/lib/tailscale
    volumes:
      - tailscale-state:/var/lib/tailscale
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - NET_ADMIN
    network_mode: service:app

  app:
    image: myapp
    depends_on:
      - tailscale

volumes:
  tailscale-state:
```

### How It Works
- Tailscale sidecar creates the VPN interface
- App container shares the network namespace (`network_mode: service:app`)
- App sees the Tailscale interface as if it were local

### Pros
- ✅ Separation of concerns (VPN vs app)
- ✅ App doesn't need VPN code
- ✅ Easy to swap VPN providers

### Cons
- ❌ Requires orchestration (Compose, K8s)
- ❌ More complex than single container

### Best For
- Kubernetes/Compose environments
- Clean separation of VPN and app logic
- When multiple apps share one VPN connection

---

## Part 5: Approach 4 - Subnet Routing

### Architecture
```
┌────────────────────────────────────┐
│           Host (Subnet Router)     │
│                                    │
│  Tailscale (100.64.0.1)            │
│  - Advertises 172.17.0.0/16        │
│                                    │
│  ┌──────────────────────────────┐  │
│  │  Container (172.17.0.2)      │  │
│  │  - No Tailscale              │  │
│  │  - Routed via host           │  │
│  └──────────────────────────────┘  │
└────────────────────────────────────┘
```

### Setup
```bash
# On the host, enable IP forwarding
sudo sysctl -w net.ipv4.ip_forward=1

# Start Tailscale as subnet router
sudo tailscale up --advertise-routes=172.17.0.0/16

# On Tailscale admin console, approve the route
```

### How It Works
```
Remote Device (100.64.0.2)
      ↓
Wants to reach 172.17.0.2
      ↓
Tailscale routes to 100.64.0.1 (subnet router)
      ↓
Host forwards to container (172.17.0.2)
```

### Pros
- ✅ Containers don't need Tailscale
- ✅ Simple container setup
- ✅ Entire subnet accessible

### Cons
- ❌ Containers can't initiate VPN connections
- ❌ Only inbound access
- ❌ Host is a single point of failure

### Best For
- Accessing containers from outside
- Legacy apps that can't run Tailscale
- Simple remote access scenarios

---

## Part 6: Tailscale-Specific Patterns

### Pattern 1: Ephemeral Nodes
```bash
# Container gets temporary VPN identity
docker run \
  --cap-add=NET_ADMIN \
  --device=/dev/net/tun \
  -e TS_AUTHKEY=tskey-xxx \
  -e TS_EPHEMERAL=true \
  tailscale/tailscale
```

**Ephemeral** = Node disappears from tailnet when container stops.

### Pattern 2: Tagged Nodes
```bash
# Use tags for ACL management
tailscale up --authkey=tskey-xxx --advertise-tags=tag:container,tag:prod
```

### Pattern 3: MagicDNS
```bash
# Containers can use DNS names
curl http://other-container.tailnet.ts.net
```

---

## Part 7: Docker Swarm + Tailscale

### The Challenge
```
Manager (100.64.0.1):
  docker swarm init --advertise-addr 100.64.0.1

Worker (100.64.0.2):
  docker swarm join --advertise-addr 100.64.0.2 100.64.0.1:2377
```

**Requirement**: Tailscale must be in the same namespace as Docker.

### Solution: Native Docker in WSL2
```bash
# In Ubuntu WSL2
# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up

# Install Docker Engine (not Desktop)
curl -fsSL https://get.docker.com | sh

# Now both are in the same namespace
docker swarm init --advertise-addr $(tailscale ip -4)
```

### Why Docker Desktop Fails
```
Windows: Tailscale (100.64.0.1)
         ↕ (different namespace)
docker-desktop VM: No Tailscale
```

Docker can't bind to an IP in a different namespace.

---

## Part 8: Kubernetes + Tailscale

### Approach 1: Tailscale Operator
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  annotations:
    tailscale.com/expose: "true"
spec:
  containers:
  - name: app
    image: myapp
```

The operator injects a Tailscale sidecar automatically.

### Approach 2: Manual Sidecar
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: tailscale
    image: tailscale/tailscale
    env:
    - name: TS_AUTHKEY
      valueFrom:
        secretKeyRef:
          name: tailscale-auth
          key: authkey
    securityContext:
      capabilities:
        add:
        - NET_ADMIN
  - name: app
    image: myapp
```

---

## Part 9: Security Considerations

### Capabilities Required
```bash
--cap-add=NET_ADMIN  # Required for creating network interfaces
--device=/dev/net/tun  # Required for TUN device
```

**Risk**: These are privileged capabilities.

### Mitigation
- Use sidecar pattern (isolate VPN from app)
- Use subnet routing (no capabilities needed in containers)
- Use Tailscale ACLs to limit access

### Auth Keys
```bash
# Generate ephemeral, reusable auth key
tailscale up --authkey=tskey-xxx
```

**Best practice**: Store in secrets, not environment variables.

---

## Part 10: Performance Considerations

### Latency
```
Direct connection:
  App → Tailscale → Peer
  Latency: ~5-20ms (depending on distance)

Relayed connection (DERP):
  App → Tailscale → DERP → Peer
  Latency: ~50-200ms
```

### Throughput
```
Direct: ~500 Mbps - 1 Gbps (WireGuard overhead ~5%)
DERP: ~100-500 Mbps (depends on relay location)
```

### Optimization
```bash
# Force direct connections (disable DERP)
tailscale up --accept-routes --shields-up=false

# Use specific DERP region
tailscale up --advertise-exit-node --exit-node-allow-lan-access
```

---

## Part 11: Comparison Table

| Approach | Complexity | Isolation | Swarm Support | Best For |
|----------|------------|-----------|---------------|----------|
| **VPN on Host** | Low | None | No (Desktop) | Simple outbound |
| **VPN in Container** | Medium | High | Yes | Per-container VPN |
| **Sidecar** | Medium | Medium | Yes | Clean separation |
| **Subnet Routing** | Low | None | No | Remote access |

---

## Part 12: Hands-On Example

### Scenario: Multi-Node App with Tailscale

#### Node 1 (Manager)
```bash
# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up

# Install Docker
curl -fsSL https://get.docker.com | sh

# Init Swarm
docker swarm init --advertise-addr $(tailscale ip -4)
```

#### Node 2 (Worker)
```bash
# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up

# Install Docker
curl -fsSL https://get.docker.com | sh

# Join Swarm
docker swarm join --advertise-addr $(tailscale ip -4) <manager-tailscale-ip>:2377
```

#### Deploy Service
```bash
# On manager
docker service create \
  --name web \
  --replicas 2 \
  --publish 80:80 \
  nginx
```

Now both nodes communicate over Tailscale!

---

## Key Takeaways

1. **VPN on host** = Simple but limited (especially with Docker Desktop)
2. **VPN in container** = Full control, requires privileges
3. **Sidecar pattern** = Clean separation, needs orchestration
4. **Subnet routing** = Easy remote access, one-way only
5. **Docker Swarm needs VPN in same namespace** as Docker Engine

---

## Further Reading
- [Tailscale Docker Guide](https://tailscale.com/kb/1282/docker/)
- [Tailscale Kubernetes Operator](https://tailscale.com/kb/1236/kubernetes-operator/)
- [Docker Network Drivers](https://docs.docker.com/network/)
