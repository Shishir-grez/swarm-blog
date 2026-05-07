# Docker Swarm Multi-Node Architecture Requirements vs Docker Desktop Limitations
## Comprehensive Technical Analysis Report

**Document Version:** 1.0  
**Date:** February 3, 2026  
**Author:** Technical Analysis Team  
**Subject:** Why Docker Desktop Cannot Support Multi-Node Docker Swarm Clusters

---

## Executive Summary

This report provides a comprehensive technical analysis of Docker Swarm's architectural requirements and demonstrates how Docker Desktop's design fundamentally prevents multi-node cluster formation. Through detailed mapping of each Swarm requirement to specific Docker Desktop architectural decisions, this document explains not just what blocks multi-node Swarm, but precisely why each blocker exists and why it cannot be easily circumvented.

**Key Finding:** Docker Desktop's architecture employs four interconnected design decisions that collectively make multi-node Swarm impossible: (1) AF_VSOCK shared memory communication, (2) VPNKit userspace networking, (3) slirp-proxy port forwarding, and (4) non-routable VM IP addressing. Each of these was deliberately chosen to optimize single-machine container operations but creates insurmountable barriers for distributed systems.

---

## Table of Contents

1. [Introduction and Scope](#introduction)
2. [Docker Swarm Behavioral Requirements](#swarm-requirements)
3. [VXLAN Overlay Network Requirements](#vxlan-requirements)
4. [Docker Desktop Architecture Overview](#desktop-architecture)
5. [Detailed Requirement-to-Blocker Mapping](#requirement-mapping)
6. [Technical Deep-Dive: Why Each Blocker Prevents Swarm](#technical-deepdive)
7. [Conclusion and Alternatives](#conclusion)

---

## 1. Introduction and Scope {#introduction}

### 1.1 Purpose

This report establishes a complete understanding of the architectural incompatibilities between Docker Desktop and Docker Swarm mode by analyzing the fundamental requirements of distributed container orchestration against the design constraints of Docker Desktop's single-host virtualization approach.

### 1.2 Methodology

Our analysis draws from official sources including Docker documentation, IETF RFC 7348 (VXLAN specification), Docker Desktop networking documentation, and VPNKit implementation details. Each requirement has been traced through the networking stack to identify the precise point of failure.

### 1.3 Definitions

**Docker Swarm**: Native clustering and orchestration solution built into Docker Engine that enables multiple Docker hosts to function as a single virtual system.

**VXLAN**: Virtual eXtensible Local Area Network, a network virtualization technology defined in RFC 7348 that uses UDP-based MAC-in-UDP encapsulation to create Layer 2 overlay networks over Layer 3 infrastructure.

**Docker Desktop**: An integrated development environment for building, shipping, and running containerized applications on Windows and macOS, using a virtualized Linux environment.

---

## 2. Docker Swarm Behavioral Requirements {#swarm-requirements}

Based on official Docker documentation, Docker Swarm requires each participating node (host) to exhibit specific networking behaviors. These requirements are mandatory and non-negotiable for cluster functionality.

### 2.1 Network Port Requirements

Docker Swarm mandates that nodes communicate over specific ports with specific protocols:

**Port 2377/TCP**: Cluster management communications
- Purpose: Raft consensus protocol for cluster state management
- Direction: Bidirectional between manager nodes
- Protocol characteristics: Stateful TCP, requires correct source IP identification
- Traffic pattern: Heartbeats, state synchronization, leader election

**Port 7946/TCP and UDP**: Node discovery and overlay network control plane
- Purpose: Gossip-based membership protocol (SWIM - Scalable Weakly-consistent Infection-style Process Group Membership)
- Direction: Bidirectional between all nodes (managers and workers)
- Protocol characteristics: Both connection-oriented (TCP) and connectionless (UDP)
- Traffic pattern: Membership updates, failure detection, metadata propagation

**Port 4789/UDP**: Overlay network data plane (VXLAN)
- Purpose: Encapsulated container-to-container traffic across hosts
- Direction: Bidirectional between all nodes
- Protocol characteristics: Stateless UDP datagrams
- Traffic pattern: High-frequency packet forwarding, no connection setup

**Protocol 50 (ESP)**: IPSec encrypted overlay networks (optional)
- Purpose: Encrypted overlay network traffic
- Direction: Bidirectional when encryption is enabled
- Protocol characteristics: IP protocol, not TCP/UDP

### 2.2 IP Address Requirements

**Requirement 2.2.1: Routable IP Addresses**

Each Docker Engine participating in the swarm must have an IP address that is reachable by all other swarm nodes. This address must exist in the routing tables of peer nodes.

Per Docker official documentation: "The IP address must be assigned to a network interface available to the host operating system. All nodes in the swarm need to connect to the manager at the IP address."

**Requirement 2.2.2: Fixed IP Addresses**

Docker documentation states: "Because other nodes contact the manager node on its IP address, you should use a fixed IP address." This requirement extends to all nodes to ensure stable cluster membership.

**Requirement 2.2.3: Advertised Address Binding**

When initializing a swarm, the advertise address must bind to an actual network interface:

```bash
docker swarm init --advertise-addr 192.168.99.100
```

This IP must be an address that the Docker Engine process can actually bind to and that remote nodes can route packets to.

### 2.3 Direct Peer-to-Peer Communication

**Requirement 2.3.1: Unsolicited Inbound Connections**

Swarm nodes must accept inbound connections from peer nodes without prior outbound connection establishment. This is fundamentally different from client-server models where the client always initiates.

Example scenario:
- Node A initializes swarm
- Node B joins swarm (B connects to A)
- Node A must now be able to send gossip messages to Node B without B having initiated a connection for this specific flow
- Both nodes must accept connections from each other independently

**Requirement 2.3.2: Source IP Preservation**

When Node A receives a packet from Node B, it must see Node B's actual IP address as the source. This is critical for:
- Maintaining accurate cluster membership lists
- Implementing failure detection (knowing which specific node failed)
- VXLAN forwarding table construction
- Load balancing decisions

### 2.4 Kernel-Level Network Stack Access

**Requirement 2.4.1: Direct UDP Socket Management**

Docker Engine must be able to create UDP sockets bound to port 4789 and send/receive datagrams directly without userspace proxying. This is because VXLAN processing happens in the kernel's networking stack, not in userspace applications.

**Requirement 2.4.2: Kernel Network Interface Creation**

Docker Engine must be able to create kernel-level network interfaces (VXLAN interfaces) using commands like:

```bash
ip link add vxlan0 type vxlan id 100 remote 192.168.99.101 dstport 4789 dev eth0
```

This creates an actual kernel network device, not a userspace abstraction.

### 2.5 Continuous Bidirectional Communication

**Requirement 2.5.1: Simultaneous Send and Receive**

All three Swarm ports must support simultaneous bidirectional communication. At any given moment:
- Node A may send to Node B on port 7946
- Node B may send to Node A on port 7946
- Node C may send to Node A on port 4789
- Node A may send to Node D on port 4789

All of these must occur concurrently without connection setup delays.

**Requirement 2.5.2: Low-Latency Path**

VXLAN overlay traffic (port 4789) requires low-latency packet forwarding because every container-to-container packet traverses this path. Userspace proxying adds 50-100+ microseconds per packet, which accumulates to unacceptable latency for high-throughput applications.

---

## 3. VXLAN Overlay Network Requirements {#vxlan-requirements}

VXLAN is defined in IETF RFC 7348. Docker Swarm uses VXLAN to implement overlay networks. Understanding VXLAN's requirements is essential to understanding Swarm's limitations.

### 3.1 VXLAN Tunnel Endpoint (VTEP) Requirements

**Requirement 3.1.1: Kernel-Space VXLAN Module**

Per RFC 7348, VXLAN encapsulation and decapsulation must occur in the networking stack. Linux implements this in the kernel's VXLAN module (drivers/net/vxlan.c). The module must:
- Intercept outbound packets destined for the overlay network
- Encapsulate them with VXLAN header + UDP header + IP header
- Send via the underlay network interface
- Receive inbound UDP packets on port 4789
- Decapsulate and extract the original Ethernet frame
- Deliver to the appropriate virtual network interface

This processing happens entirely in kernel space for performance reasons.

**Requirement 3.1.2: VTEP IP Address**

Each VTEP must have a real IP address on the underlay network. This is the "outer IP" used in VXLAN encapsulation. RFC 7348 states that VTEPs communicate using standard IP routing, meaning these addresses must be routable within the network.

**Requirement 3.1.3: Direct UDP Transmission**

VTEPs must send and receive UDP datagrams on port 4789 directly to/from peer VTEPs. Per RFC 7348: "A well-known UDP port (4789) has been assigned by the IANA in the Service Name and Transport Protocol Port Number Registry for VXLAN."

### 3.2 VXLAN Frame Format Requirements

**Requirement 3.2.1: Proper Encapsulation Format**

VXLAN frames must follow this exact structure (from RFC 7348):

```
Outer Ethernet Header:
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|             Outer Destination MAC Address                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|             Outer Source MAC Address                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Outer IP Header:
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|       Outer Source IP         |      Outer Dest IP            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Outer UDP Header:
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Source Port (entropy)       |       Dest Port = 4789        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

VXLAN Header:
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|R|R|R|R|I|R|R|R|            Reserved                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                VXLAN Network Identifier (VNI)                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Inner Ethernet Frame (original container packet):
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|             Inner Destination MAC Address                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|             Inner Source MAC Address                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Inner Payload                            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

This encapsulation MUST be performed by the kernel's VXLAN module for every packet.

**Requirement 3.2.2: Outer IP Address Integrity**

The "Outer Source IP" in the VXLAN packet must be the actual IP address of the sending VTEP. The "Outer Dest IP" must be the actual IP address of the receiving VTEP. These addresses are used to maintain the VXLAN forwarding database (FDB), which maps inner MAC addresses to outer IP addresses.

### 3.3 VXLAN Forwarding Database Requirements

**Requirement 3.3.1: FDB Construction**

Each VTEP maintains a forwarding database that maps container MAC addresses to VTEP IP addresses:

```
Inner MAC Address    →    Outer VTEP IP
02:42:0a:00:00:02   →    192.168.99.101
02:42:0a:00:00:03   →    192.168.99.102
```

This database is populated by:
- Learning: When a VXLAN packet arrives, the VTEP learns "packets from inner MAC X come from outer IP Y"
- Control plane: Docker's gossip protocol distributes MAC-to-IP mappings

**Requirement 3.3.2: Accurate Source IP for Learning**

The learning mechanism requires that when VTEP-A receives a packet, it sees VTEP-B's real IP as the source. If the source IP is wrong or proxied, the FDB becomes corrupted, and return traffic cannot be routed.

### 3.4 VXLAN Performance Requirements

**Requirement 3.4.1: Kernel-Space Processing**

To achieve acceptable performance, VXLAN encapsulation/decapsulation must occur entirely in kernel space. Userspace processing would require:
- Kernel → userspace context switch for every packet
- Userspace processing
- Userspace → kernel context switch to send

For a 10 Gbps network link, this could mean millions of context switches per second, creating unacceptable CPU overhead.

**Requirement 3.4.2: Zero-Copy Forwarding**

Modern kernel implementations of VXLAN support zero-copy forwarding where packet data is not copied between buffers during encapsulation. This requires direct kernel network stack access.

---

## 4. Docker Desktop Architecture Overview {#desktop-architecture}

Docker Desktop implements a virtualized Docker environment on Windows and macOS. Its architecture was designed for single-host container development, not distributed systems.

### 4.1 Architectural Components

Per Docker's official documentation, Docker Desktop on both Windows and macOS uses the following architecture:

```
Physical Machine (Windows/Mac)
│
├── Host Operating System (Windows 10/11 or macOS)
│   │
│   ├── Docker Desktop Application
│   │   ├── com.docker.backend (Go process)
│   │   │   └── Port forwarding management
│   │   │   └── API proxying
│   │   │   └── VM lifecycle management
│   │   │
│   │   └── com.docker.vpnkit (OCaml process)
│   │       └── Userspace TCP/IP stack
│   │       └── Connection proxying
│   │       └── DNS/DHCP/HTTP services
│   │
│   └── Hypervisor
│       ├── Windows: Hyper-V or WSL2
│       └── macOS: Hypervisor.framework (HyperKit)
│
└── Linux VM (DockerDesktopVM or docker-desktop WSL2 distribution)
    │
    ├── Networking
    │   ├── eth0: 192.168.65.3 (internal IP, non-routable)
    │   ├── docker0 bridge: 172.17.0.1
    │   └── AF_VSOCK interface (for host communication)
    │
    ├── Docker Engine (dockerd)
    │   └── Running containers
    │
    └── Helper Services
        ├── vpnkit-forwarder
        └── slirp-proxy instances
```

### 4.2 Component Purposes

**Component 4.2.1: AF_VSOCK / Hypervisor Sockets**

From Docker's documentation: "Host/VM communication uses AF_VSOCK hypervisor sockets (shared memory) rather than network switches or interfaces."

Purpose: Provide high-performance, low-latency communication between the host OS and the VM without using TCP/IP networking.

Mechanism: Shared memory regions that both the hypervisor and VM can access, with interrupt-based signaling.

**Component 4.2.2: VPNKit**

From Docker's blog post on networking: "Docker Desktop avoids this problem by forwarding all traffic at user-level via vpnkit, a TCP/IP stack written in OCaml on top of the network protocol libraries of the MirageOS Unikernel project."

Purpose: Intercept VM network traffic and recreate connections on the host to bypass VPN and firewall restrictions.

Mechanism: Reads raw Ethernet frames from the VM via AF_VSOCK, parses them, creates corresponding sockets on the host, and proxies data between the VM's virtual connection and the host's real connection.

**Component 4.2.3: slirp-proxy (Port Forwarding)**

From VPNKit documentation: "In Docker for Mac and Docker for Windows we configure the docker daemon to use a userspace proxy, and we provide our own custom implementation."

Purpose: Forward published container ports to the host OS for external access.

Mechanism: When a port is published with `docker run -p`, the slirp-proxy process:
1. Listens on the requested host port
2. Accepts connections
3. Forwards them via AF_VSOCK to the VM
4. Proxies to the container's internal IP and port

**Component 4.2.4: Non-Routable VM IP**

The VM receives an IP address (typically 192.168.65.3) via DHCP from VPNKit. This address:
- Exists only within the hypervisor's context
- Cannot be reached from the physical network
- Is not in the host's routing table as a routable destination
- Is used for VM-internal networking only

### 4.3 Design Intentions

Docker Desktop's architecture was intentionally designed to:
1. Work on corporate VPNs that block VM traffic
2. Function without requiring administrator privileges
3. Provide seamless internet access to containers
4. Allow port publishing for local development
5. Maintain security isolation between the VM and host
6. Simplify the user experience (no network configuration required)

These design goals are perfectly achieved for single-host development but create fundamental barriers for multi-host orchestration.

---

## 5. Detailed Requirement-to-Blocker Mapping {#requirement-mapping}

This section maps each Docker Swarm/VXLAN requirement to the specific Docker Desktop architectural component(s) that prevent it from being satisfied.

### Mapping Table

| Requirement ID | Requirement Description | Primary Blocker | Secondary Blockers | Severity |
|---------------|------------------------|----------------|-------------------|----------|
| 2.1-2377 | Port 2377/TCP cluster management | AF_VSOCK, Port Forwarding | Non-routable IP, VPNKit | CRITICAL |
| 2.1-7946 | Port 7946/TCP+UDP node discovery | AF_VSOCK, Port Forwarding | Non-routable IP, VPNKit | CRITICAL |
| 2.1-4789 | Port 4789/UDP VXLAN data plane | AF_VSOCK, VPNKit | Port Forwarding, Non-routable IP | CRITICAL |
| 2.2.1 | Routable IP addresses | Non-routable IP | AF_VSOCK | CRITICAL |
| 2.2.2 | Fixed IP addresses | VPNKit DHCP | Non-routable IP | HIGH |
| 2.2.3 | Advertise address binding | Non-routable IP | AF_VSOCK | CRITICAL |
| 2.3.1 | Unsolicited inbound connections | VPNKit, Port Forwarding | AF_VSOCK | CRITICAL |
| 2.3.2 | Source IP preservation | Port Forwarding, AF_VSOCK | VPNKit | CRITICAL |
| 2.4.1 | Direct UDP socket management | VPNKit, AF_VSOCK | - | CRITICAL |
| 2.4.2 | Kernel network interface creation | AF_VSOCK | Non-routable IP | CRITICAL |
| 2.5.1 | Simultaneous bidirectional comm | VPNKit, Port Forwarding | AF_VSOCK | HIGH |
| 2.5.2 | Low-latency path | VPNKit, Port Forwarding | - | HIGH |
| 3.1.1 | Kernel-space VXLAN module | AF_VSOCK | VPNKit | CRITICAL |
| 3.1.2 | VTEP IP address | Non-routable IP | AF_VSOCK | CRITICAL |
| 3.1.3 | Direct UDP transmission | AF_VSOCK, VPNKit | - | CRITICAL |
| 3.2.2 | Outer IP address integrity | Port Forwarding, AF_VSOCK | VPNKit | CRITICAL |
| 3.3.1 | FDB construction | Port Forwarding | VPNKit, Non-routable IP | CRITICAL |
| 3.3.2 | Accurate source IP for learning | Port Forwarding | AF_VSOCK | CRITICAL |
| 3.4.1 | Kernel-space processing | VPNKit, Port Forwarding | AF_VSOCK | HIGH |
| 3.4.2 | Zero-copy forwarding | VPNKit, Port Forwarding | - | MEDIUM |

---

## 6. Technical Deep-Dive: Why Each Blocker Prevents Swarm {#technical-deepdive}

This section provides detailed technical explanations for why each Docker Desktop architectural component prevents Swarm from functioning.

### 6.1 AF_VSOCK Blocker Analysis

#### 6.1.1 What AF_VSOCK Is

AF_VSOCK (Address Family Virtual Sockets) is a Linux socket address family specifically designed for communication between a virtual machine and its hypervisor. From the Linux manual page (vsock.7):

"The VSOCK address family facilitates communication between virtual machines and the host they are running on. This address family is used by guest agents and hypervisor services that need a communications channel that is independent of virtual machine network configuration."

Technical implementation:
- Uses Context IDs (CIDs) instead of IP addresses
- Host always has CID = 2
- Each VM gets a unique CID (e.g., 3, 4, 5)
- Communication occurs through shared memory regions
- No involvement of the network stack (TCP/IP, routing, NICs)

#### 6.1.2 Why AF_VSOCK Blocks Swarm Requirement 2.1 (Network Ports)

**The fundamental problem:** AF_VSOCK can only facilitate communication between a VM and its hypervisor. It cannot transmit packets to other physical machines.

**Technical explanation:**

When Docker Swarm on Node A tries to communicate with Node B on port 2377:

```
Node A (Machine 1):
  Docker Engine binds to: 192.168.65.3:2377
  Tries to send packet to: Node B at 192.168.99.100:2377
  
  Packet flow:
  1. Docker Engine creates TCP packet: 192.168.65.3:2377 → 192.168.99.100:2377
  2. Kernel routing lookup: "Where is 192.168.99.100?"
  3. VM's routing table has default route via 192.168.65.1 (VPNKit gateway)
  4. Packet sent to eth0 interface
  5. eth0 is actually connected to AF_VSOCK, not a real network
  6. Packet transmitted via AF_VSOCK to host
  
  At the host:
  7. VPNKit receives the packet via AF_VSOCK
  8. VPNKit sees destination: 192.168.99.100 (an external IP)
  9. VPNKit would need to forward this to Machine 2
  10. FAILURE: VPNKit has no mechanism to forward to other machines
```

**Why this occurs:** AF_VSOCK uses shared memory. Machine 1's RAM and Machine 2's RAM are physically separate. There is no shared memory between them. AF_VSOCK cannot create a communication channel across separate physical memory spaces.

The Linux kernel's AF_VSOCK implementation (net/vmw_vsock/af_vsock.c) only supports these address types:
- VMADDR_CID_HOST (2): The host machine
- VMADDR_CID_LOCAL (1): Loopback within the VM
- Specific CIDs assigned by the local hypervisor

There is no mechanism for "remote CID" or "CID on another physical machine."

**Why it affects these requirements:**
- **2.1-2377, 2.1-7946, 2.1-4789:** All three ports require sending packets to remote machines, which AF_VSOCK cannot do
- **2.2.3:** Cannot bind to an address reachable from other machines
- **2.3.1:** Cannot receive unsolicited packets from other machines via AF_VSOCK
- **2.3.2:** Source IP is lost because AF_VSOCK uses CIDs, not IPs
- **2.4.2:** VXLAN interfaces need real network devices, not AF_VSOCK channels
- **3.1.1, 3.1.3:** VXLAN module needs to send UDP to remote IPs, not CIDs

#### 6.1.3 Why Modifying AF_VSOCK Won't Work

Some might ask: "Can't we extend AF_VSOCK to support remote VMs?"

The answer is no, for several reasons:

**Reason 1: Physical Memory Limitation**

AF_VSOCK uses shared memory. The hypervisor allocates memory regions that both the host and VM can access:

```
Machine 1's RAM:
  [Host Memory] [Shared Region A] [VM Memory]
       ↑              ↑               ↑
    Host only    Both access      VM only
```

Machine 2 cannot access Machine 1's RAM. This is a hardware limitation.

**Reason 2: CID Scope**

CIDs are assigned by each hypervisor independently:
```
Machine 1's Hypervisor:
  CID 2 = Host
  CID 3 = Docker Desktop VM
  
Machine 2's Hypervisor:
  CID 2 = Host
  CID 3 = Docker Desktop VM
```

Both VMs have CID 3. CIDs are not globally unique. If you tried to extend AF_VSOCK to work remotely, you'd need a global CID registry, defeating the entire purpose of AF_VSOCK's simplicity.

**Reason 3: Design Philosophy**

AF_VSOCK was designed to avoid network configuration. Adding remote support would require:
- Network addresses
- Routing
- Discovery mechanisms
- Firewall traversal

At which point you've reimplemented TCP/IP networking, making AF_VSOCK pointless.

### 6.2 VPNKit Blocker Analysis

#### 6.2.1 What VPNKit Is

From Docker's official blog: "Docker Desktop avoids this problem by forwarding all traffic at user-level via vpnkit, a TCP/IP stack written in OCaml on top of the network protocol libraries of the MirageOS Unikernel project."

VPNKit is a userspace network stack that intercepts network traffic from the VM and recreates connections on the host. It was designed to solve a specific problem: corporate VPNs that block VM traffic.

**Technical implementation:**

```
Container in VM wants to access google.com:

1. Container creates packet: 172.17.0.2:45678 → 172.217.164.46:80
2. VM's kernel processes packet normally
3. Packet reaches VM's eth0 interface
4. eth0 sends packet via AF_VSOCK to VPNKit
5. VPNKit receives raw Ethernet frame
6. VPNKit parses: Ethernet → IP → TCP
7. VPNKit understands: "This is a SYN packet to google.com:80"
8. VPNKit calls socket() on host OS
9. VPNKit calls connect(google.com, 80) on host OS
10. Host OS connection succeeds (VPN allows host traffic)
11. VPNKit proxies data between VM's virtual connection and host's real connection
```

This is **connection recreation**, not routing or NAT.

#### 6.2.2 Why VPNKit Blocks Swarm Requirement 2.1 (Network Ports)

**The fundamental problem:** VPNKit is designed for VM-initiated outbound connections. It cannot handle peer-to-peer bidirectional communication.

**Technical explanation for Port 4789/UDP (VXLAN):**

```
Node A tries to send VXLAN packet to Node B:

1. Node A's VXLAN module creates encapsulated packet:
   Outer: 192.168.65.3:4789 → 192.168.99.100:4789
   Inner: [Container A's Ethernet frame]

2. Packet sent to eth0
3. eth0 transmits via AF_VSOCK to VPNKit

4. VPNKit receives packet
5. VPNKit parses UDP packet
6. VPNKit sees destination: 192.168.99.100:4789

7. PROBLEM: VPNKit doesn't know about 192.168.99.100
   - VPNKit has no list of peer nodes
   - VPNKit has no mechanism to discover peers
   - 192.168.99.100 is not an Internet address
   
8. VPNKit's logic:
   if (destination is Internet):
       create_connection_on_host()
   else if (destination is internal service):
       forward_to_vpnkit_service()
   else:
       ???  // No handler for peer VM addresses
       
9. Packet dropped
```

**Why this occurs:** VPNKit was designed with a specific traffic model:

```
VPNKit's Expected Model:
  VM → Internet (outbound)
  Internet → VM (response to outbound)
```

VPNKit maintains state for each outbound connection and matches responses:

```
VPNKit Connection Table:
  VM:45678 ↔ VPNKit:host_socket ↔ Google:80
  VM:45679 ↔ VPNKit:host_socket ↔ GitHub:443
```

**What Swarm needs:**

```
Swarm's Required Model:
  Node A → Node B (peer-initiated)
  Node B → Node A (peer-initiated)
  Node C → Node A (peer-initiated)
  Node A → Node C (peer-initiated)
```

This is fundamentally different. There's no "outbound" or "inbound" concept—all nodes are equal peers.

**Why VPNKit cannot be adapted:**

VPNKit would need to:

1. **Maintain peer registry:**
   ```
   Peer List:
     192.168.99.100 (Node B) → How to reach it?
     192.168.99.101 (Node C) → How to reach it?
   ```
   But VPNKit has no mechanism to discover or store this information.

2. **Handle unsolicited UDP:**
   ```
   Current VPNKit UDP model:
     VM sends UDP → VPNKit creates socket → Waits for response
     
   Swarm UDP model:
     Node B sends UDP to Node A (unsolicited)
     Node A has never sent to Node B before
     No existing socket to receive this packet
   ```

3. **Create bidirectional forwarding:**
   ```
   For each peer node:
     Create permanent UDP socket
     Maintain forwarding state
     Handle concurrent packets
   ```
   This is essentially implementing a VPN tunnel, completely different from VPNKit's design.

**Why it affects these requirements:**
- **2.1-4789:** UDP VXLAN traffic cannot be forwarded to peer nodes
- **2.3.1:** Cannot receive unsolicited packets from peers
- **2.5.1:** Cannot handle simultaneous bidirectional peer communication
- **2.4.1:** Interposes userspace proxy instead of direct UDP socket access
- **3.1.3:** Cannot provide direct UDP transmission to peer VTEPs
- **3.4.1:** Forces userspace processing instead of kernel-space

#### 6.2.3 The Performance Problem

Even if VPNKit could be modified to handle peer traffic, it would create unacceptable latency:

**Normal kernel-to-kernel VXLAN:**
```
Container A → Kernel VXLAN module → NIC → Network → NIC → Kernel VXLAN module → Container B

Latency: ~2-5 microseconds (kernel processing)
```

**VPNKit-proxied VXLAN:**
```
Container A → Kernel → AF_VSOCK → VPNKit (userspace) → Parse/Process → Create socket → Send to network → Network → Receive → VPNKit → Parse/Process → AF_VSOCK → Kernel → Container B

Latency: ~50-200 microseconds (userspace context switches, parsing, socket operations)
```

For a 10 Gbps network pushing small packets (1500 bytes), that's:
- Normal: ~830,000 packets/second
- VPNKit: ~50,000 packets/second (16x slower)

### 6.3 Port Forwarding (slirp-proxy) Blocker Analysis

#### 6.3.1 What Port Forwarding Does

From VPNKit documentation: "For example, after running docker run -p 8080:80 nginx on the host, inside the VM we can see a process: /usr/bin/slirp-proxy -proto tcp -host-ip 0.0.0.0 -host-port 8080 -container-ip 172.17.0.2 -container-port 80"

Port forwarding allows external clients to access containers by forwarding connections from the host to the VM.

**Technical implementation:**

```
User runs: docker run -p 8080:80 nginx

1. Docker Engine creates port forward request
2. com.docker.backend receives request
3. com.docker.backend listens on host:8080
4. External client connects to host:8080
5. com.docker.backend accepts connection
6. com.docker.backend opens connection via AF_VSOCK to VM
7. com.docker.backend proxies data:
   - Read from client socket → Write to AF_VSOCK
   - Read from AF_VSOCK → Write to client socket
```

#### 6.3.2 Why Port Forwarding Blocks Swarm Requirement 2.3.2 (Source IP Preservation)

**The fundamental problem:** Port forwarding is a userspace proxy that establishes separate connections on each side. The original source IP is lost.

**Technical explanation:**

```
External Node B tries to connect to Node A's Swarm port:

Hypothetical scenario (even if ports were published):

1. Node B: connect(192.168.1.100:2377)  // Node A's host IP
   Source: 192.168.99.100:54321
   Dest: 192.168.1.100:2377

2. Packet arrives at Node A's host (192.168.1.100)

3. com.docker.backend has listener on port 2377
   accept() returns new socket

4. com.docker.backend sees:
   Peer address: 192.168.99.100:54321
   
5. com.docker.backend connects via AF_VSOCK to VM
   Uses CID 3 (VM), port 2377
   
6. Inside VM, connection appears to come from:
   CID 2 (the host), not from 192.168.99.100

7. Docker Engine sees:
   Source: 127.0.0.1 or internal host address
   NOT: 192.168.99.100
   
8. Swarm membership database corrupted:
   Node B's IP: ??? (Unknown)
   Cannot route packets back to Node B
```

**Why this occurs:** The proxy creates two separate connections:

```
Connection 1 (External):
  Node B (192.168.99.100:54321) ←→ Node A Host (192.168.1.100:2377)
  
Connection 2 (Internal):
  Host (CID 2) ←→ VM (CID 3, port 2377)
```

These are **two different TCP connections**. The Docker Engine only sees Connection 2, which has no information about Node B's IP address.

**Metadata preservation attempt:**

The slirp-proxy does send metadata about the connection:

```
{
  "target_ip": "172.17.0.2",
  "target_port": 80,
  "original_source": "192.168.99.100:54321"
}
```

However, this metadata is consumed by the forwarding infrastructure. When the connection reaches the Docker Engine, it appears as a regular TCP connection from the host. The Docker Engine's socket API returns the peer address as the host, not the original client.

**Why it affects these requirements:**
- **2.3.2:** Source IP is not preserved
- **3.2.2:** Outer IP in VXLAN packets would be wrong
- **3.3.1, 3.3.2:** FDB construction fails due to incorrect source IPs
- **2.1-2377, 2.1-7946:** Cluster management sees all connections from localhost

#### 6.3.3 The UDP Problem

Port forwarding is even more problematic for UDP:

**TCP Port Forwarding:**
```
Connection-oriented, easy to track:
  Client A connects → Backend creates state → Proxies until close
  Client B connects → Backend creates separate state → Proxies until close
```

**UDP Port Forwarding:**
```
Connectionless, ambiguous:
  Packet from 192.168.99.100:54321 arrives
  Packet from 192.168.99.101:54322 arrives
  Packet from 192.168.99.100:54321 arrives again
  
  Questions:
    - Which packets belong together?
    - When to timeout a "session"?
    - How to handle bidirectional flows?
    - How to handle simultaneous packets?
```

For VXLAN port 4789, this is catastrophic:
- High-frequency packet arrival (millions per second)
- No concept of connections or sessions
- Bidirectional flows from/to multiple peers simultaneously
- Strict latency requirements

Port forwarding UDP would require:
- Per-packet inspection
- Session tracking heuristics
- Significant CPU overhead
- Still wouldn't preserve source IPs correctly

### 6.4 Non-Routable IP Blocker Analysis

#### 6.4.1 What Non-Routable IP Means

The Docker Desktop VM receives an IP address from VPNKit's DHCP server, typically 192.168.65.3. This address:

- Exists only in the VM's network namespace
- Is not in the host's routing table
- Cannot be reached from other machines
- Is not advertised via any routing protocol

**Technical verification:**

On the host machine:
```bash
# Windows
route print
# Does not show route to 192.168.65.3

# macOS
netstat -nr
# Does not show route to 192.168.65.3
```

The address is purely internal to the hypervisor's virtual networking.

#### 6.4.2 Why Non-Routable IP Blocks Swarm Requirement 2.2.1 (Routable Addresses)

**The fundamental problem:** Other machines cannot send packets to an IP address they cannot route to.

**Technical explanation:**

```
Node A: 192.168.65.3 (VM IP, non-routable)
Node B: 192.168.99.100 (Real machine IP)

Node A initializes Swarm:
docker swarm init --advertise-addr 192.168.65.3

Node B tries to join:
docker swarm join --token <token> 192.168.65.3:2377

Node B's kernel:
1. Routing table lookup for 192.168.65.3
2. Check local subnets:
   - 192.168.99.0/24 → Local subnet
   - 192.168.65.3 not in local subnet
3. Check routing table entries:
   - Default route: via 192.168.99.1
   - No specific route to 192.168.65.0/24
4. Forward to default gateway

Default gateway (192.168.99.1):
1. Receives packet for 192.168.65.3
2. Routing table lookup
3. Check connected networks: Not connected
4. Check routes: No route
5. Response: ICMP Network Unreachable
6. Or: Silent drop (many routers do this for RFC 1918 addresses)

Node B receives ICMP Network Unreachable
connect() system call returns: ENETUNREACH
```

**Why this occurs:** IP routing is hierarchical. Each router must know where to forward packets. For 192.168.65.3 to be routable:

```
Required routing table entries:

Node B's routing table:
  192.168.65.0/24 via <some gateway>

Default gateway's routing table:
  192.168.65.0/24 via <some path>

All intermediate routers:
  192.168.65.0/24 via <next hop>
```

But these entries don't exist because 192.168.65.3 is not a real network address. It's a virtual address inside a VM.

**Even on the same LAN:**

```
Node A Host: 192.168.1.100 (real)
Node A VM: 192.168.65.3 (virtual, inside hypervisor)
Node B: 192.168.1.101 (real)

Node B sends to 192.168.65.3:
1. ARP request: "Who has 192.168.65.3?"
2. Broadcast on LAN: ff:ff:ff:ff:ff:ff
3. All devices receive ARP request
4. Node A Host's NIC receives request
5. Node A Host's kernel checks: "Do I have 192.168.65.3?"
6. No, I have 192.168.1.100
7. No ARP reply sent
8. Node B: "Cannot reach 192.168.65.3" (no ARP reply)
```

The VM's IP is inside the hypervisor's private network space. It never appears on physical network interfaces.

**Why it affects these requirements:**
- **2.2.1:** Other nodes cannot route to the VM's IP
- **2.2.3:** Cannot advertise a non-routable address
- **3.1.2:** VTEP IP must be routable, but VM IP is not

#### 6.4.3 Workaround Attempts and Why They Fail

**Attempt 1: Use host's IP**

```
docker swarm init --advertise-addr 192.168.1.100  # Host's IP

Problems:
1. Docker Engine binds to its own interfaces
2. Cannot bind to 192.168.1.100 (that's the host's IP, not VM's)
3. bind() fails: EADDRNOTAVAIL (address not available)
```

**Attempt 2: Port mapping**

```
Publish ports 2377, 7946, 4789 to host

Problems:
1. These are Engine ports, not container ports
2. Port publishing is for containers
3. Even if published, source IP problem remains
4. UDP 4789 forwarding performance unacceptable
5. Bidirectional communication broken
```

**Attempt 3: Bridge networking**

```
Configure VM with bridged networking instead of AF_VSOCK

Problems:
1. Docker Desktop doesn't support this
2. VM is locked down, can't reconfigure
3. Would require complete redesign of Docker Desktop
4. Would break VPN compatibility (the whole reason for VPNKit)
```

### 6.5 Combined Effect: Why All Four Blockers Matter

The four blockers are not independent—they form an integrated system. Fixing one alone doesn't enable Swarm.

#### 6.5.1 Blocker Interdependencies

```
Blocker Dependency Graph:

Non-Routable IP
    ↓ (requires)
AF_VSOCK (because real IP would bypass isolation)
    ↓ (requires)
VPNKit (because AF_VSOCK can't reach internet directly)
    ↓ (requires)
Port Forwarding (because VPNKit is unidirectional)
```

Each component was chosen because of constraints imposed by the others.

**Example Scenario: Fix Non-Routable IP**

Suppose we give the VM a routable IP (192.168.1.150):

```
Problem 1: How does VM access internet?
  - Can't use AF_VSOCK anymore (it's for VM↔Host only)
  - VM would need real network interface
  - Bridged networking required
  
Problem 2: How does port publishing work?
  - Can't use slirp-proxy (requires AF_VSOCK)
  - Would need actual port forwarding (iptables)
  - Requires host to have iptables access to VM's network
  
Problem 3: How does VPN compatibility work?
  - Real network traffic would be blocked by VPN
  - Back to the original problem VPNKit was designed to solve!
```

**Example Scenario: Replace AF_VSOCK with Bridge Network**

```
Change VM networking to bridged mode:

Problem 1: VPN compatibility lost
  - Corporate VPNs block VM traffic
  - Main use case for Docker Desktop broken
  
Problem 2: VPNKit no longer applicable
  - VPNKit requires AF_VSOCK to intercept traffic
  - Would need complete rewrite
  
Problem 3: Port forwarding changes needed
  - Can no longer use AF_VSOCK-based forwarding
  - Requires iptables or similar
  - Requires admin privileges
  
Problem 4: Security model changes
  - VM now on physical network
  - Requires firewall configuration
  - Loses isolation benefits
```

**The Architecture is Coherent**

Docker Desktop's architecture makes perfect sense for its intended use case:

```
Design Goal: Single-host container development on Windows/Mac
Constraints:
  - Corporate VPNs
  - No admin privileges required
  - Simple user experience
  - Strong isolation

Solution:
  ✓ Isolated VM (non-routable IP)
  ✓ AF_VSOCK for VM↔Host (fast, secure)
  ✓ VPNKit for VM→Internet (bypasses VPN)
  ✓ Port forwarding for Host→Container (user-friendly)

Result: Perfect for intended use case
        Incompatible with distributed systems
```

---

## 7. Conclusion and Alternatives {#conclusion}

### 7.1 Summary of Findings

Docker Desktop's architecture creates four interconnected barriers to multi-node Swarm:

1. **AF_VSOCK**: Shared memory communication limited to VM↔Host, cannot reach remote machines
2. **VPNKit**: Userspace network proxy designed for VM→Internet, cannot handle peer-to-peer
3. **Port Forwarding**: Userspace TCP proxy that loses source IP information
4. **Non-Routable IP**: VM IP address not present in network routing tables

Each requirement from Docker Swarm and VXLAN is blocked by one or more of these components. The blockers are interdependent—fixing one creates problems with the others. The entire architecture would need replacement to support multi-node Swarm.

### 7.2 Why This Cannot Be Fixed with Minor Changes

Enabling multi-node Swarm on Docker Desktop would require:

```
Required Changes:
1. Replace AF_VSOCK with bridged networking
2. Remove VPNKit (incompatible with bridged networking)
3. Remove custom port forwarding (incompatible with bridged networking)
4. Assign routable IP to VM
5. Implement iptables or equivalent for port mapping
6. Handle VPN compatibility differently (or abandon it)
7. Require admin privileges for network configuration
8. Redesign security model for exposed VM
```

At this point, you've essentially rebuilt WSL2 with Docker Engine, not Docker Desktop.

### 7.3 What Docker Desktop Can Do

Docker Desktop works perfectly for:
- Single-node development
- Running containers locally
- Testing applications
- Learning Docker
- Kubernetes (single-node via Docker Desktop's built-in k8s)
- Docker Compose
- Single-node Swarm (for testing Swarm concepts)

### 7.4 Alternatives for Multi-Node Swarm

To run multi-node Docker Swarm, use one of these alternatives:

**Option 1: Cloud VMs**
- AWS EC2, DigitalOcean Droplets, Azure VMs, GCP Compute
- Each VM has real IP, full network access
- Full Docker Engine installation
- Production-ready

**Option 2: Local VMs with Bridged Networking**
- VMware Workstation, VirtualBox, Hyper-V
- Configure bridged networking
- Install Docker Engine on each VM
- VMs get IPs on your physical network

**Option 3: Physical Machines**
- Dedicated Linux servers
- Raspberry Pi cluster
- Full network control
- Best performance

**Option 4: WSL2 + Docker Engine + Tailscale**
- Install Docker Engine directly in WSL2 (not Docker Desktop)
- Use Tailscale for virtual networking
- Provides routable IPs via Tailscale mesh
- Works across NAT/firewalls
- More complex setup, full Linux control

### 7.5 Final Recommendations

**For Learning Swarm:**
- Use single-node Swarm in Docker Desktop to learn concepts
- Understand services, stacks, routing mesh
- Test basic functionality

**For Testing Swarm:**
- Use 2-3 cloud VMs
- Small instance sizes (1-2 GB RAM each)
- Minimal cost for testing
- Tear down when not needed

**For Production:**
- Cloud provider managed services (ECS, AKS, GKE)
- Or self-managed VMs with proper networking
- Consider Kubernetes for larger deployments
- Implement monitoring, logging, backups

**Do Not Attempt:**
- Hacking Docker Desktop to support multi-node Swarm
- Creating complex network overlays or tunnels
- Running production workloads on single-node Swarm

### 7.6 Understanding the Design Trade-Off

Docker Desktop optimizes for developer experience on single workstations. Its architecture trades distributed systems capability for:
- VPN compatibility
- Simple installation
- No admin privileges needed
- Security isolation
- Reliable networking

This is the correct trade-off for a desktop development tool. For distributed systems, use tools designed for that purpose (cloud VMs, Kubernetes clusters, etc.).

---

## Appendix A: Reference Architecture Diagrams

### A.1 Docker Desktop Architecture (Current)

```
┌─────────────────────────────────────────────────┐
│ Physical Machine (Windows/Mac)                  │
│                                                 │
│ ┌─────────────────────────────────────────────┐ │
│ │ Host OS                                     │ │
│ │                                             │ │
│ │ ┌─────────────────────────────────────────┐ │ │
│ │ │ Docker Desktop Processes                │ │ │
│ │ │                                         │ │ │
│ │ │ com.docker.backend                      │ │ │
│ │ │   • Port forwarding                     │ │ │
│ │ │   • API proxy                           │ │ │
│ │ │                                         │ │ │
│ │ │ com.docker.vpnkit                       │ │ │
│ │ │   • Userspace TCP/IP stack              │ │ │
│ │ │   • Connection proxy                    │ │ │
│ │ │   • DNS/DHCP/HTTP                       │ │ │
│ │ └─────────────────────────────────────────┘ │ │
│ │                    ↕                        │ │
│ │           AF_VSOCK (Shared Memory)          │ │
│ │                    ↕                        │ │
│ │ ┌─────────────────────────────────────────┐ │ │
│ │ │ Hypervisor (Hyper-V / HyperKit)         │ │ │
│ │ │                                         │ │ │
│ │ │  ┌───────────────────────────────────┐  │ │ │
│ │ │  │ Linux VM (DockerDesktopVM)        │  │ │ │
│ │ │  │                                   │  │ │ │
│ │ │  │ eth0: 192.168.65.3 (non-routable) │  │ │ │
│ │ │  │                                   │  │ │ │
│ │ │  │ Docker Engine (dockerd)           │  │ │ │
│ │ │  │   ├─ Container 1                  │  │ │ │
│ │ │  │   ├─ Container 2                  │  │ │ │
│ │ │  │   └─ Container 3                  │  │ │ │
│ │ │  └───────────────────────────────────┘  │ │ │
│ │ └─────────────────────────────────────────┘ │ │
│ │                                             │ │
│ │ Physical NIC: 192.168.1.100                 │ │
│ └─────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
         ↕
    Physical Network
```

### A.2 Required Architecture for Multi-Node Swarm

```
Machine 1                              Machine 2
┌────────────────────┐                 ┌────────────────────┐
│ Linux OS           │                 │ Linux OS           │
│                    │                 │                    │
│ eth0: 192.168.1.100│←─────────────→│ eth0: 192.168.1.101│
│                    │  Physical Net   │                    │
│ Docker Engine      │                 │ Docker Engine      │
│  ├─ vxlan0 (4789)  │                 │  ├─ vxlan0 (4789)  │
│  ├─ Container 1    │                 │  ├─ Container 3    │
│  └─ Container 2    │                 │  └─ Container 4    │
└────────────────────┘                 └────────────────────┘
```

Note the critical differences:
- Docker Engine runs directly on host OS
- Real, routable IP addresses
- No virtualization layer between Engine and physical network
- Direct kernel access for VXLAN

---

## Appendix B: Glossary of Technical Terms

**AF_VSOCK**: Address Family Virtual Sockets, a Linux socket family for VM-to-host communication

**CID**: Context ID, used in AF_VSOCK instead of IP addresses

**FDB**: Forwarding Database, maps MAC addresses to VTEP IPs in VXLAN

**Hypervisor**: Software that creates and runs virtual machines

**Overlay Network**: Virtual network built on top of physical network infrastructure

**RFC 7348**: IETF specification defining VXLAN protocol

**Source IP Preservation**: Maintaining original sender's IP address through proxies/NAT

**Userspace**: Memory and CPU mode where regular applications run (vs kernel space)

**VNI**: VXLAN Network Identifier, identifies specific overlay networks

**VPNKit**: Docker Desktop's userspace network stack for VM→Internet connectivity

**VTEP**: VXLAN Tunnel Endpoint, terminates VXLAN tunnels

**VXLAN**: Virtual eXtensible LAN, MAC-in-UDP encapsulation for overlay networks

---

## Appendix C: Port Mapping Reference

| Port | Protocol | Purpose | Required On | Direction |
|------|----------|---------|-------------|-----------|
| 2377 | TCP | Cluster management (Raft) | Manager nodes | Bidirectional between managers |
| 7946 | TCP | Control plane gossip | All nodes | Bidirectional between all |
| 7946 | UDP | Control plane gossip | All nodes | Bidirectional between all |
| 4789 | UDP | VXLAN overlay data plane | All nodes | Bidirectional between all |
| 50 | ESP | Encrypted overlay (optional) | All nodes | Bidirectional when encryption enabled |

---

**End of Report**
