# Docker Swarm Architecture Diagrams
## Visual Packet Flow Analysis: Docker Desktop vs Multi-Node Swarm Requirements

**Document Purpose:** This document provides visual representations of network packet flows to illustrate exactly where and why Docker Desktop's architecture fails to support multi-node Docker Swarm. Each diagram shows the complete journey of a packet through the system, making it easy to identify the precise failure points.

---

## Table of Contents

1. [Successful Swarm Communication (Standard Linux)](#successful-swarm)
2. [Failed Swarm Communication (Docker Desktop)](#failed-swarm)
3. [Container Internet Access (Docker Desktop - Works)](#internet-access)
4. [Container-to-Container Across Nodes (Swarm - Fails in Docker Desktop)](#cross-node-container)
5. [Port Publishing (Docker Desktop - Works for Single Host)](#port-publishing)
6. [Architecture Comparison: Standard vs Docker Desktop](#architecture-comparison)
7. [Component Interaction Diagrams](#component-interaction)

---

## 1. Successful Swarm Communication (Standard Linux) {#successful-swarm}

This diagram shows how Docker Swarm communication works on standard Linux hosts with Docker Engine installed directly. This is the baseline that demonstrates what Swarm needs to function correctly.

### 1.1 VXLAN Packet Flow Between Two Swarm Nodes (Working)

```
┌─────────────────────────────────────────────────────────────────────┐
│ Machine A: 192.168.1.100                                            │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ Container A (172.18.0.2)                                      │ │
│  │                                                               │ │
│  │  Application sends data to Container B at 172.18.0.3         │ │
│  │  Creates packet: [SRC: 172.18.0.2] [DST: 172.18.0.3]         │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                            ↓                                        │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ Docker Bridge (docker_gwbridge)                               │ │
│  │                                                               │ │
│  │  Packet forwarded to overlay network interface               │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                            ↓                                        │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ VXLAN Interface (vxlan0) - KERNEL SPACE                       │ │
│  │                                                               │ │
│  │  Step 1: Lookup destination MAC in FDB                       │ │
│  │    Inner MAC 02:42:ac:12:00:03 → Outer IP 192.168.1.101      │ │
│  │                                                               │ │
│  │  Step 2: Encapsulate with VXLAN header                       │ │
│  │    [VXLAN Header: VNI=256, Flags=0x08]                        │ │
│  │    [Inner Ethernet: 02:42:ac:12:00:02 → 02:42:ac:12:00:03]   │ │
│  │    [Inner IP: 172.18.0.2 → 172.18.0.3]                        │ │
│  │    [Application Data]                                         │ │
│  │                                                               │ │
│  │  Step 3: Encapsulate with UDP header                         │ │
│  │    [UDP: SRC=45678 (entropy), DST=4789 (VXLAN port)]         │ │
│  │                                                               │ │
│  │  Step 4: Encapsulate with outer IP header                    │ │
│  │    [Outer IP: SRC=192.168.1.100, DST=192.168.1.101]          │ │
│  │                                                               │ │
│  │  Complete packet structure:                                  │ │
│  │  ┌──────────────────────────────────────────────────────┐    │ │
│  │  │ Outer Ethernet Header                                │    │ │
│  │  ├──────────────────────────────────────────────────────┤    │ │
│  │  │ Outer IP: 192.168.1.100 → 192.168.1.101              │    │ │
│  │  ├──────────────────────────────────────────────────────┤    │ │
│  │  │ Outer UDP: 45678 → 4789                              │    │ │
│  │  ├──────────────────────────────────────────────────────┤    │ │
│  │  │ VXLAN Header (VNI=256)                               │    │ │
│  │  ├──────────────────────────────────────────────────────┤    │ │
│  │  │ Inner Ethernet: Container A MAC → Container B MAC    │    │ │
│  │  ├──────────────────────────────────────────────────────┤    │ │
│  │  │ Inner IP: 172.18.0.2 → 172.18.0.3                    │    │ │
│  │  ├──────────────────────────────────────────────────────┤    │ │
│  │  │ TCP/Application Data                                 │    │ │
│  │  └──────────────────────────────────────────────────────┘    │ │
│  │                                                               │ │
│  │  Processing time: ~2-5 microseconds (all in kernel)          │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                            ↓                                        │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ Physical NIC (eth0) - KERNEL SPACE                            │ │
│  │                                                               │ │
│  │  Kernel sends packet directly to NIC driver                  │ │
│  │  Zero userspace involvement - maximum performance            │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                            ↓                                        │
└─────────────────────────────────────────────────────────────────────┘
                             ↓
                    Physical Network
                   (Ethernet Switch/Router)
                             ↓
┌─────────────────────────────────────────────────────────────────────┐
│ Machine B: 192.168.1.101                                            │
│                            ↑                                        │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ Physical NIC (eth0) - KERNEL SPACE                            │ │
│  │                                                               │ │
│  │  Packet arrives at NIC with destination IP 192.168.1.101     │ │
│  │  NIC driver passes to kernel network stack                   │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                            ↑                                        │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ Kernel IP Stack                                               │ │
│  │                                                               │ │
│  │  Routing decision: Packet destined for local IP              │ │
│  │  Checks UDP destination port: 4789 → VXLAN handler           │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                            ↑                                        │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ VXLAN Interface (vxlan0) - KERNEL SPACE                       │ │
│  │                                                               │ │
│  │  Step 1: Receive UDP packet on port 4789                     │ │
│  │  Step 2: Extract VXLAN header, verify VNI=256                │ │
│  │  Step 3: Extract inner Ethernet frame                        │ │
│  │  Step 4: Learn FDB entry:                                    │ │
│  │    Inner SRC MAC 02:42:ac:12:00:02 → 192.168.1.100           │ │
│  │    (Now knows how to reach Container A)                      │ │
│  │  Step 5: Decapsulate and forward inner frame                 │ │
│  │                                                               │ │
│  │  Processing time: ~2-5 microseconds (all in kernel)          │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                            ↑                                        │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ Docker Bridge (docker_gwbridge)                               │ │
│  │                                                               │ │
│  │  Routes packet to correct container based on inner dst MAC   │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                            ↑                                        │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ Container B (172.18.0.3)                                      │ │
│  │                                                               │ │
│  │  Application receives data from Container A                  │ │
│  │  Packet: [SRC: 172.18.0.2] [DST: 172.18.0.3]                 │ │
│  │                                                               │ │
│  │  ✅ SUCCESS - Packet delivered correctly                      │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

KEY OBSERVATIONS:
• All processing happens in kernel space (fast, efficient)
• Real IP addresses used (192.168.1.100, 192.168.1.101)
• Source IP preserved throughout (enables FDB learning)
• Direct UDP transmission on port 4789
• No userspace proxying (no performance penalty)
• Total latency: ~10-20 microseconds (including network transit)
```

### 1.2 Swarm Control Plane Communication (Port 2377)

```
┌─────────────────────────────────────────────────────────────────────┐
│ Machine A: 192.168.1.100 (Manager Node)                            │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ Docker Engine (dockerd) - KERNEL SPACE SOCKET                 │ │
│  │                                                               │ │
│  │  Listening on: 0.0.0.0:2377 (bound to all interfaces)        │ │
│  │  Actual binding: 192.168.1.100:2377                          │ │
│  │                                                               │ │
│  │  Socket API:                                                 │ │
│  │    sock = socket(AF_INET, SOCK_STREAM, 0)                    │ │
│  │    bind(sock, 192.168.1.100:2377)                            │ │
│  │    listen(sock, backlog)                                     │ │
│  │    accept() → waits for incoming connections                 │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                            ↑                                        │
└─────────────────────────────────────────────────────────────────────┘
                             ↑
                    Physical Network
                             ↑
┌─────────────────────────────────────────────────────────────────────┐
│ Machine B: 192.168.1.101 (Worker Node)                             │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ Docker Engine (dockerd) - Joining Swarm                       │ │
│  │                                                               │ │
│  │  Command: docker swarm join --token <token> \                │ │
│  │           192.168.1.100:2377                                  │ │
│  │                                                               │ │
│  │  Socket API:                                                 │ │
│  │    sock = socket(AF_INET, SOCK_STREAM, 0)                    │ │
│  │    connect(sock, 192.168.1.100:2377)                         │ │
│  │                                                               │ │
│  │  TCP handshake:                                              │ │
│  │    ┌──────────────────────────────────────────────────────┐  │ │
│  │    │ SYN packet created:                                  │  │ │
│  │    │   SRC: 192.168.1.101:54321 (ephemeral port)          │  │ │
│  │    │   DST: 192.168.1.100:2377                            │  │ │
│  │    │   Flags: SYN                                         │  │ │
│  │    └──────────────────────────────────────────────────────┘  │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                            ↓                                        │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ Kernel Network Stack                                          │ │
│  │                                                               │ │
│  │  Routing lookup: 192.168.1.100 → via eth0                    │ │
│  │  ARP lookup: 192.168.1.100 → MAC address                     │ │
│  │  Send packet via physical NIC                                │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                            ↓                                        │
│                    Physical Network                                 │
│                            ↓                                        │
└─────────────────────────────────────────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────────────┐
│ Machine A: 192.168.1.100 (Manager Node)                            │
│                            ↑                                        │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ Kernel Network Stack                                          │ │
│  │                                                               │ │
│  │  Packet arrives: 192.168.1.101:54321 → 192.168.1.100:2377    │ │
│  │  Port lookup: 2377 → Docker Engine is listening              │ │
│  │  TCP handshake completion:                                   │ │
│  │    ← SYN-ACK sent back to 192.168.1.101:54321                │ │
│  │    → ACK received, connection established                    │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                            ↑                                        │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ Docker Engine - accept() returns                              │ │
│  │                                                               │ │
│  │  getsockname() returns peer address:                         │ │
│  │    Peer IP: 192.168.1.101  ✅ Correct!                        │ │
│  │    Peer Port: 54321                                          │ │
│  │                                                               │ │
│  │  Swarm cluster state updated:                                │ │
│  │    Node added: 192.168.1.101                                 │ │
│  │    Node role: Worker                                         │ │
│  │    Reachable at: 192.168.1.101:7946 (gossip)                 │ │
│  │                                                               │ │
│  │  ✅ SUCCESS - Node joined cluster                             │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

KEY OBSERVATIONS:
• Direct TCP connection between nodes
• Source IP preserved (192.168.1.101 visible to Manager)
• No proxying or translation
• Bidirectional communication established
• Both nodes can now initiate connections to each other
```

---

## 2. Failed Swarm Communication (Docker Desktop) {#failed-swarm}

Now let's see what happens when we try the same operations on Docker Desktop. This series of diagrams shows exactly where and why the communication fails.

### 2.1 VXLAN Packet Flow Attempt (Docker Desktop - FAILS)

```
┌─────────────────────────────────────────────────────────────────────┐
│ Machine A - Docker Desktop on Windows/Mac                          │
│                                                                     │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ Host OS (Windows 10 / macOS)                                    │ │
│ │   IP: 192.168.1.100 (real, routable)                            │ │
│ │                                                                 │ │
│ │ ┌─────────────────────────────────────────────────────────────┐ │ │
│ │ │ VPNKit Process (userspace)                                  │ │ │
│ │ │   Waiting for packets from VM via AF_VSOCK...               │ │ │
│ │ └─────────────────────────────────────────────────────────────┘ │ │
│ │                          ↕                                      │ │
│ │                   AF_VSOCK Channel                              │ │
│ │                 (Shared Memory Region)                          │ │
│ │                          ↕                                      │ │
│ │ ┌─────────────────────────────────────────────────────────────┐ │ │
│ │ │ Hypervisor (Hyper-V / HyperKit)                             │ │ │
│ │ │                                                             │ │ │
│ │ │  ┌───────────────────────────────────────────────────────┐  │ │ │
│ │ │  │ Linux VM (DockerDesktopVM)                            │  │ │ │
│ │ │  │   eth0: 192.168.65.3 (non-routable, internal only)    │  │ │ │
│ │ │  │                                                       │  │ │ │
│ │ │  │ ┌──────────────────────────────────────────────────┐ │  │ │ │
│ │ │  │ │ Container A (172.18.0.2)                         │ │  │ │ │
│ │ │  │ │                                                  │ │  │ │ │
│ │ │  │ │ Application tries to send to Container B at      │ │  │ │ │
│ │ │  │ │ 172.18.0.3 (on Machine B)                        │ │  │ │ │
│ │ │  │ │                                                  │ │  │ │ │
│ │ │  │ │ Packet: [SRC: 172.18.0.2] [DST: 172.18.0.3]     │ │  │ │ │
│ │ │  │ └──────────────────────────────────────────────────┘ │  │ │ │
│ │ │  │                        ↓                              │  │ │ │
│ │ │  │ ┌──────────────────────────────────────────────────┐ │  │ │ │
│ │ │  │ │ Docker Bridge                                    │ │  │ │ │
│ │ │  │ │                                                  │ │  │ │ │
│ │ │  │ │ Forwards to overlay network interface            │ │  │ │ │
│ │ │  │ └──────────────────────────────────────────────────┘ │  │ │ │
│ │ │  │                        ↓                              │  │ │ │
│ │ │  │ ┌──────────────────────────────────────────────────┐ │  │ │ │
│ │ │  │ │ VXLAN Interface (vxlan0)                         │ │  │ │ │
│ │ │  │ │                                                  │ │  │ │ │
│ │ │  │ │ FDB lookup: Inner MAC → Outer IP                │ │  │ │ │
│ │ │  │ │   02:42:ac:12:00:03 → 192.168.65.4               │ │  │ │ │
│ │ │  │ │   (Machine B's VM IP - should be routable)       │ │  │ │ │
│ │ │  │ │                                                  │ │  │ │ │
│ │ │  │ │ Creates VXLAN encapsulated packet:               │ │  │ │ │
│ │ │  │ │   Outer IP: 192.168.65.3 → 192.168.65.4          │ │  │ │ │
│ │ │  │ │   Outer UDP: random → 4789                       │ │  │ │ │
│ │ │  │ │   VXLAN Header (VNI=256)                         │ │  │ │ │
│ │ │  │ │   Inner frame...                                 │ │  │ │ │
│ │ │  │ └──────────────────────────────────────────────────┘ │  │ │ │
│ │ │  │                        ↓                              │  │ │ │
│ │ │  │ ┌──────────────────────────────────────────────────┐ │  │ │ │
│ │ │  │ │ VM Kernel Routing                                │ │  │ │ │
│ │ │  │ │                                                  │ │  │ │ │
│ │ │  │ │ Destination: 192.168.65.4                        │ │  │ │ │
│ │ │  │ │ Routing table lookup:                            │ │  │ │ │
│ │ │  │ │   192.168.65.0/24 → via 192.168.65.1 (gateway)   │ │  │ │ │
│ │ │  │ │                                                  │ │  │ │ │
│ │ │  │ │ Forward packet to eth0 → gateway                 │ │  │ │ │
│ │ │  │ └──────────────────────────────────────────────────┘ │  │ │ │
│ │ │  │                        ↓                              │  │ │ │
│ │ │  │ ┌──────────────────────────────────────────────────┐ │  │ │ │
│ │ │  │ │ eth0 Interface                                   │ │  │ │ │
│ │ │  │ │                                                  │ │  │ │ │
│ │ │  │ │ ⚠️  CRITICAL: eth0 is NOT a real NIC!            │ │  │ │ │
│ │ │  │ │ It's connected to AF_VSOCK, not physical network │ │  │ │ │
│ │ │  │ │                                                  │ │  │ │ │
│ │ │  │ │ Packet transmitted via shared memory to host     │ │  │ │ │
│ │ │  │ └──────────────────────────────────────────────────┘ │  │ │ │
│ │ │  └───────────────────────────────────────────────────────┘  │ │ │
│ │ └─────────────────────────────────────────────────────────────┘ │ │
│ │                          ↑                                      │ │
│ │                   AF_VSOCK Channel                              │ │
│ │                 (Shared Memory Region)                          │ │
│ │                          ↑                                      │ │
│ │ ┌─────────────────────────────────────────────────────────────┐ │ │
│ │ │ VPNKit Process Receives Packet                              │ │ │
│ │ │                                                             │ │ │
│ │ │ Step 1: Parse Ethernet frame from AF_VSOCK                  │ │ │
│ │ │ Step 2: Parse IP header                                     │ │ │
│ │ │   Source: 192.168.65.3                                      │ │ │
│ │ │   Dest: 192.168.65.4                                        │ │ │
│ │ │ Step 3: Parse UDP header                                    │ │ │
│ │ │   Dest Port: 4789 (VXLAN)                                   │ │ │
│ │ │                                                             │ │ │
│ │ │ ❌ FAILURE POINT 1: VPNKit Decision Logic                   │ │ │
│ │ │                                                             │ │ │
│ │ │ if (destination is internet address):                       │ │ │
│ │ │     create_connection_on_host()  // For 8.8.8.8, etc.      │ │ │
│ │ │ else if (destination is VPNKit service):                    │ │ │
│ │ │     route_to_internal_service()  // For DNS, etc.          │ │ │
│ │ │ else if (destination is 192.168.65.x):                      │ │ │
│ │ │     ❌ NO HANDLER - This is another VM's address!           │ │ │
│ │ │     ❌ VPNKit doesn't know about peer VMs                   │ │ │
│ │ │     ❌ VPNKit has no registry of 192.168.65.4               │ │ │
│ │ │                                                             │ │ │
│ │ │ Result: PACKET DROPPED                                      │ │ │
│ │ │                                                             │ │ │
│ │ │ Even if VPNKit tried to forward this packet:               │ │ │
│ │ │   - Where would it send it?                                │ │ │
│ │ │   - 192.168.65.4 is not on the physical network            │ │ │
│ │ │   - It's inside another machine's VM                       │ │ │
│ │ │   - No routing path exists                                 │ │ │
│ │ └─────────────────────────────────────────────────────────────┘ │ │
│ └─────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘

❌ COMMUNICATION FAILS - Container B never receives packet

WHY THIS FAILS:
1. VM's eth0 is NOT a real network interface - it's an AF_VSOCK channel
2. Packets go to VPNKit instead of physical network
3. VPNKit has no knowledge of peer VMs or their addresses
4. Even if VPNKit knew about 192.168.65.4, it's not routable
5. No mechanism exists to forward to Machine B

ROOT CAUSES:
• AF_VSOCK: Can only communicate between VM and its host
• Non-Routable IP: 192.168.65.4 doesn't exist on network
• VPNKit Design: Built for VM→Internet, not VM→VM
```

### 2.2 Swarm Join Attempt (Docker Desktop - FAILS)

```
┌─────────────────────────────────────────────────────────────────────┐
│ Machine A - Docker Desktop (Manager)                               │
│   Host IP: 192.168.1.100                                           │
│   VM IP: 192.168.65.3                                              │
│                                                                     │
│  Swarm initialized:                                                 │
│    docker swarm init --advertise-addr 192.168.65.3                 │
│                                                                     │
│  Docker Engine (in VM) listening on:                               │
│    192.168.65.3:2377 ✅ (inside VM only)                            │
│    192.168.65.3:7946 ✅ (inside VM only)                            │
│    192.168.65.3:4789 ✅ (inside VM only)                            │
└─────────────────────────────────────────────────────────────────────┘

                             ❌
                    Cannot communicate
                             ❌

┌─────────────────────────────────────────────────────────────────────┐
│ Machine B - Docker Desktop (Trying to join)                        │
│   Host IP: 192.168.1.101                                           │
│   VM IP: 192.168.65.3 (same as Machine A!)                         │
│                                                                     │
│  Command executed:                                                  │
│    docker swarm join --token <token> 192.168.65.3:2377             │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ Docker Engine (in Machine B's VM)                             │ │
│  │                                                               │ │
│  │ Attempt to connect to 192.168.65.3:2377                       │ │
│  │                                                               │ │
│  │ Socket API call:                                              │ │
│  │   sock = socket(AF_INET, SOCK_STREAM, 0)                     │ │
│  │   connect(sock, "192.168.65.3", 2377)                        │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                            ↓                                        │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ VM Kernel - Routing Decision                                  │ │
│  │                                                               │ │
│  │ Destination: 192.168.65.3:2377                                │ │
│  │                                                               │ │
│  │ Routing table lookup:                                         │ │
│  │   127.0.0.0/8    → lo (loopback)                              │ │
│  │   192.168.65.0/24 → eth0                                      │ │
│  │   default         → 192.168.65.1 (via eth0)                   │ │
│  │                                                               │ │
│  │ ❌ FAILURE POINT 1: Routing Confusion                         │ │
│  │                                                               │ │
│  │ Question: Is 192.168.65.3 local or remote?                    │ │
│  │                                                               │ │
│  │ Case 1: If kernel thinks it's local (192.168.65.3 = self)    │ │
│  │   → Routes to loopback                                        │ │
│  │   → Connects to its own Docker Engine                         │ │
│  │   → Wrong! Should connect to Machine A                        │ │
│  │                                                               │ │
│  │ Case 2: If kernel thinks it's remote                         │ │
│  │   → Routes via eth0 (AF_VSOCK channel)                        │ │
│  │   → Packet goes to VPNKit on THIS machine                     │ │
│  │   → VPNKit has no route to Machine A's VM                     │ │
│  │   → Packet dropped                                            │ │
│  │                                                               │ │
│  │ Either way: ❌ CONNECTION FAILS                                │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  Result returned to Docker Engine:                                  │
│    connect() fails with: ENETUNREACH (Network unreachable)         │
│    or ETIMEDOUT (Connection timed out)                             │
│                                                                     │
│  Error message displayed:                                           │
│    Error response from daemon: manager 192.168.65.3:2377           │
│    is unreachable                                                  │
└─────────────────────────────────────────────────────────────────────┘

❌ JOIN FAILS - Node cannot reach manager

WHY THIS FAILS:
1. Both VMs have same IP (192.168.65.3) - not globally unique
2. No routing path between different machines' VMs
3. AF_VSOCK only works VM↔Host, not VM↔VM across machines
4. Even with unique IPs, they're non-routable

EVEN IF WE TRIED TO USE HOST IP:
┌─────────────────────────────────────────────────────────────────────┐
│ Attempt: docker swarm join --token <token> 192.168.1.100:2377      │
│ (Using Machine A's host IP instead of VM IP)                       │
│                                                                     │
│  ❌ FAILURE POINT 2: Docker Engine Not Listening on Host IP         │
│                                                                     │
│  Machine A's Docker Engine is listening on: 192.168.65.3:2377      │
│  Packet arrives at: 192.168.1.100:2377 (host)                      │
│                                                                     │
│  Host kernel checks: "Is anyone listening on port 2377?"           │
│  Answer: NO - Docker Engine is in the VM, not on host              │
│                                                                     │
│  Result: Connection refused                                         │
│                                                                     │
│  Even if we published the port:                                     │
│    docker run -p 2377:2377 ... (doesn't work for Engine ports)     │
│                                                                     │
│  Port publishing is for CONTAINERS, not Docker Engine itself        │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Container Internet Access (Docker Desktop - Works) {#internet-access}

To understand what Docker Desktop CAN do, let's see how outbound internet access works perfectly. This shows VPNKit working as designed.

### 3.1 Successful Outbound Connection

```
┌─────────────────────────────────────────────────────────────────────┐
│ Docker Desktop VM                                                   │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ Container: nginx                                               │ │
│  │   IP: 172.17.0.2                                               │ │
│  │                                                                │ │
│  │ Application wants to download from: google.com:80              │ │
│  │                                                                │ │
│  │ Step 1: DNS Resolution                                         │ │
│  │   getaddrinfo("google.com") → 172.217.164.46                   │ │
│  │                                                                │ │
│  │ Step 2: Create TCP connection                                  │ │
│  │   socket(AF_INET, SOCK_STREAM, 0)                             │ │
│  │   connect(172.217.164.46, 80)                                 │ │
│  │                                                                │ │
│  │ Creates packet:                                                │ │
│  │   [Ethernet: Container MAC → Gateway MAC]                      │ │
│  │   [IP: 172.17.0.2 → 172.217.164.46]                           │ │
│  │   [TCP: ephemeral port → 80, SYN flag]                        │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                            ↓                                        │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ Docker Bridge (docker0)                                        │ │
│  │                                                                │ │
│  │ Forwards packet to VM's network stack                          │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                            ↓                                        │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ VM Network Stack                                               │ │
│  │                                                                │ │
│  │ NAT Translation:                                               │ │
│  │   Source: 172.17.0.2 → 192.168.65.3 (VM's IP)                 │ │
│  │                                                                │ │
│  │ Packet now:                                                    │ │
│  │   [IP: 192.168.65.3 → 172.217.164.46]                         │ │
│  │   [TCP: 45678 → 80, SYN flag]                                 │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                            ↓                                        │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ eth0 Interface (connected to AF_VSOCK)                         │ │
│  │                                                                │ │
│  │ Sends raw Ethernet frame via shared memory to host             │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                            ↓                                        │
└─────────────────────────────────────────────────────────────────────┘
                             ↓
                      AF_VSOCK Channel
                    (Shared Memory Transfer)
                             ↓
┌─────────────────────────────────────────────────────────────────────┐
│ Host OS - VPNKit Process (userspace)                               │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ Step 1: Receive Frame from AF_VSOCK                            │ │
│  │                                                                │ │
│  │   Raw Ethernet frame received via shared memory                │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                            ↓                                        │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ Step 2: Parse and Understand Traffic                           │ │
│  │                                                                │ │
│  │   Ethernet Layer Parser:                                       │ │
│  │     Type: IPv4 (0x0800)                                        │ │
│  │                                                                │ │
│  │   IP Layer Parser:                                             │ │
│  │     Source: 192.168.65.3                                       │ │
│  │     Dest: 172.217.164.46 ✅ Internet address!                  │ │
│  │     Protocol: TCP (6)                                          │ │
│  │                                                                │ │
│  │   TCP Layer Parser:                                            │ │
│  │     Source Port: 45678                                         │ │
│  │     Dest Port: 80                                              │ │
│  │     Flags: SYN ✅ Connection request                           │ │
│  │                                                                │ │
│  │ ✅ VPNKit recognizes: Outbound connection to Internet          │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                            ↓                                        │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ Step 3: Create Real Connection on Host OS                      │ │
│  │                                                                │ │
│  │   VPNKit creates actual socket:                                │ │
│  │     host_sock = socket(AF_INET, SOCK_STREAM, 0)               │ │
│  │     connect(host_sock, 172.217.164.46, 80)                    │ │
│  │                                                                │ │
│  │   This appears to corporate VPN as:                            │ │
│  │     "VPNKit.exe is making a connection to Google"              │ │
│  │     Source: Host's real IP (after VPN translation)             │ │
│  │     Dest: 172.217.164.46:80                                    │ │
│  │                                                                │ │
│  │   ✅ VPN ALLOWS THIS - It's normal host traffic                │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                            ↓                                        │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ Step 4: Connection Established                                 │ │
│  │                                                                │ │
│  │   TCP handshake completes:                                     │ │
│  │     → SYN sent to Google                                       │ │
│  │     ← SYN-ACK received from Google                             │ │
│  │     → ACK sent to Google                                       │ │
│  │                                                                │ │
│  │   VPNKit now maintains two connections:                        │ │
│  │                                                                │ │
│  │   Virtual Connection (in VPNKit's memory):                     │ │
│  │     VM (192.168.65.3:45678) ↔ VPNKit                          │ │
│  │                                                                │ │
│  │   Real Connection (on host):                                   │ │
│  │     VPNKit ↔ Google (172.217.164.46:80)                       │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                            ↓                                        │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ Step 5: Proxy Data Between Connections                         │ │
│  │                                                                │ │
│  │   VPNKit creates SYN-ACK response packet to VM:                │ │
│  │     [Ethernet: Gateway MAC → Container MAC]                    │ │
│  │     [IP: 172.217.164.46 → 192.168.65.3]                       │ │
│  │     [TCP: 80 → 45678, SYN-ACK flags]                          │ │
│  │                                                                │ │
│  │   Sends this synthetic packet via AF_VSOCK to VM               │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                            ↓                                        │
└─────────────────────────────────────────────────────────────────────┘
                             ↓
                      AF_VSOCK Channel
                             ↓
┌─────────────────────────────────────────────────────────────────────┐
│ Docker Desktop VM                                                   │
│                            ↑                                        │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ VM receives SYN-ACK                                            │ │
│  │                                                                │ │
│  │ Connection now ESTABLISHED from VM's perspective               │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                            ↑                                        │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ Container Application                                          │ │
│  │                                                                │ │
│  │ connect() returns successfully!                                │ │
│  │                                                                │ │
│  │ Now can send HTTP request:                                     │ │
│  │   send(sock, "GET / HTTP/1.1\r\n...")                         │ │
│  │                                                                │ │
│  │ Data flows:                                                    │ │
│  │   Container → VM → AF_VSOCK → VPNKit → Real connection        │ │
│  │                                                                │ │
│  │ Response flows back:                                           │ │
│  │   Real connection → VPNKit → AF_VSOCK → VM → Container        │ │
│  │                                                                │ │
│  │ ✅ SUCCESS - Internet access works perfectly!                  │ │
│  └───────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘

WHY THIS WORKS:
• Outbound connection (VM initiates to Internet)
• VPNKit designed for exactly this use case
• Destination is Internet address (not peer VM)
• Connection proxying is acceptable for Internet traffic
• VPN sees only host traffic (VPNKit.exe), not VM traffic
• Performance penalty acceptable for development use

THIS IS THE DESIGN GOAL OF DOCKER DESKTOP - IT WORKS PERFECTLY!
```

---

## 4. Port Publishing Flow (Docker Desktop - Works for Single Host) {#port-publishing}

### 4.1 Successful Port Publishing

```
┌─────────────────────────────────────────────────────────────────────┐
│ User executes: docker run -p 8080:80 nginx                         │
└─────────────────────────────────────────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────────────┐
│ Docker Desktop VM                                                   │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ Docker Engine (dockerd)                                        │ │
│  │                                                                │ │
│  │ Starts nginx container: 172.17.0.2:80                          │ │
│  │                                                                │ │
│  │ Creates port forward request via 9P filesystem:                │ │
│  │   Creates: /port/tcp:0.0.0.0:8080:tcp:172.17.0.2:80           │ │
│  │                                                                │ │
│  │ Starts slirp-proxy process:                                    │ │
│  │   /usr/bin/slirp-proxy \                                       │ │
│  │     -proto tcp \                                               │ │
│  │     -host-ip 0.0.0.0 \                                         │ │
│  │     -host-port 8080 \                                          │ │
│  │     -container-ip 172.17.0.2 \                                 │ │
│  │     -container-port 80                                         │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                            ↓                                        │
└─────────────────────────────────────────────────────────────────────┘
                             ↓
                    Port forward request
                    transmitted via AF_VSOCK
                             ↓
┌─────────────────────────────────────────────────────────────────────┐
│ Host OS - com.docker.backend Process                               │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ Receives port forward request                                  │ │
│  │                                                                │ │
│  │ Validates request:                                             │ │
│  │   - Check if port 8080 is available                            │ │
│  │   - If already in use, return error                            │ │
│  │   - If available, proceed                                      │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                            ↓                                        │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ Creates TCP listener on host                                   │ │
│  │                                                                │ │
│  │ Socket API:                                                    │ │
│  │   listen_sock = socket(AF_INET, SOCK_STREAM, 0)              │ │
│  │   setsockopt(listen_sock, SO_REUSEADDR, ...)                  │ │
│  │   bind(listen_sock, "0.0.0.0", 8080)                          │ │
│  │   listen(listen_sock, 128)                                     │ │
│  │                                                                │ │
│  │ ✅ Host now listening on all interfaces:                       │ │
│  │   0.0.0.0:8080 (accessible from anywhere)                      │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                            ↓                                        │
│                      Waiting for connections...                     │
│                            ↓                                        │
└─────────────────────────────────────────────────────────────────────┘
                             ↓
                    External client connects
                             ↓
┌─────────────────────────────────────────────────────────────────────┐
│ External Client (Browser on 192.168.1.200)                         │
│                                                                     │
│  User browses to: http://192.168.1.100:8080                        │
│                                                                     │
│  Browser creates connection:                                        │
│    Source: 192.168.1.200:54321                                     │
│    Dest: 192.168.1.100:8080                                        │
│                                                                     │
│  TCP handshake:                                                     │
│    → SYN sent                                                       │
└─────────────────────────────────────────────────────────────────────┘
                             ↓
                      Physical Network
                             ↓
┌─────────────────────────────────────────────────────────────────────┐
│ Host OS - com.docker.backend Process                               │
│                            ↑                                        │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ accept() returns with new client connection                    │ │
│  │                                                                │ │
│  │   client_sock = accept(listen_sock, ...)                      │ │
│  │   getpeername(client_sock) → 192.168.1.200:54321              │ │
│  │                                                                │ │
│  │ ✅ Backend knows: External client is 192.168.1.200             │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                            ↓                                        │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ Open connection to VM via AF_VSOCK                             │ │
│  │                                                                │ │
│  │ VPNKit-forwarder establishes connection to:                    │ │
│  │   Destination: 172.17.0.2:80 (nginx container)                 │ │
│  │                                                                │ │
│  │ Sends metadata (for logging):                                  │ │
│  │   {                                                            │ │
│  │     "original_source": "192.168.1.200:54321",                  │ │
│  │     "target_container": "172.17.0.2:80"                        │ │
│  │   }                                                            │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                            ↓                                        │
└─────────────────────────────────────────────────────────────────────┘
                             ↓
                      AF_VSOCK Channel
                             ↓
┌─────────────────────────────────────────────────────────────────────┐
│ Docker Desktop VM                                                   │
│                            ↑                                        │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ vpnkit-forwarder receives connection                           │ │
│  │                                                                │ │
│  │ Opens TCP connection to nginx container:                       │ │
│  │   container_sock = socket(AF_INET, SOCK_STREAM, 0)            │ │
│  │   connect(container_sock, 172.17.0.2, 80)                     │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                            ↑                                        │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ nginx container accepts connection                             │ │
│  │                                                                │ │
│  │ From nginx's perspective:                                      │ │
│  │   Connection from: VM's internal IP (not original client)      │ │
│  │   Destination: 172.17.0.2:80                                   │ │
│  │                                                                │ │
│  │ ⚠️  Source IP is lost - nginx sees VM IP, not 192.168.1.200   │ │
│  │    But for web serving, this is usually acceptable             │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                            ↑                                        │
│                            ↓                                        │
│               ┌───────────────────────────┐                         │
│               │   Data Proxying Loop:     │                         │
│               │                           │                         │
│               │   Client ←→ Backend       │                         │
│               │   Backend ←→ Container    │                         │
│               │                           │                         │
│               │   All data copied through │                         │
│               │   these two connections   │                         │
│               └───────────────────────────┘                         │
│                                                                     │
│  Browser sends: GET / HTTP/1.1\r\n...                              │
│    → Host backend → AF_VSOCK → VM → Container                      │
│                                                                     │
│  nginx responds: HTTP/1.1 200 OK\r\n<html>...                      │
│    ← Container → VM → AF_VSOCK → Host backend → Browser            │
│                                                                     │
│  ✅ SUCCESS - Web page loads in browser!                            │
└─────────────────────────────────────────────────────────────────────┘

WHY THIS WORKS:
• Single direction: External → Host → VM → Container
• Port publishing explicitly configured
• Backend can listen on host's real IP
• Client initiates connection (expected pattern)
• Source IP loss acceptable for web servers
• Works for single-host development perfectly!

WHY THIS WON'T WORK FOR SWARM:
• Swarm ports (2377, 7946, 4789) are Docker Engine ports, not container ports
• Bidirectional peer communication needed, not client-server
• Source IP preservation critical for Swarm membership
• UDP port 4789 forwarding would be high-overhead
• No mechanism to publish Engine's ports (only container ports)
```

---

## 5. Architecture Comparison: Standard vs Docker Desktop {#architecture-comparison}

### 5.1 Side-by-Side Comparison

```
┌────────────────────────────────┬────────────────────────────────────┐
│   STANDARD LINUX SETUP         │      DOCKER DESKTOP SETUP          │
│  (Multi-node Swarm WORKS)      │   (Multi-node Swarm FAILS)         │
├────────────────────────────────┼────────────────────────────────────┤
│                                │                                    │
│  ┌──────────────────────────┐  │  ┌──────────────────────────────┐  │
│  │ Physical/Cloud Server    │  │  │ Windows/Mac Workstation      │  │
│  │                          │  │  │                              │  │
│  │ IP: 192.168.1.100        │  │  │ ┌──────────────────────────┐ │  │
│  │ (Real, Routable)         │  │  │ │ Host OS                  │ │  │
│  │                          │  │  │ │ IP: 192.168.1.100        │ │  │
│  │ ┌──────────────────────┐ │  │  │ │ (Real, Routable)         │ │  │
│  │ │ Linux OS (Ubuntu)    │ │  │  │ │                          │ │  │
│  │ │                      │ │  │  │ │ ┌──────────────────────┐ │ │  │
│  │ │ eth0: 192.168.1.100  │ │  │  │ │ │ Hypervisor           │ │ │  │
│  │ │ ↕                    │ │  │  │ │ │                      │ │ │  │
│  │ │ Physical NIC         │ │  │  │ │ │ ┌──────────────────┐ │ │ │  │
│  │ │ ↕                    │ │  │  │ │ │ │ Linux VM         │ │ │ │  │
│  │ │ Network Switch       │ │  │  │ │ │ │                  │ │ │ │  │
│  │ │                      │ │  │  │ │ │ │ eth0: 192.168.65 │ │ │ │  │
│  │ │ ┌──────────────────┐ │ │  │  │ │ │ │ (Non-routable)   │ │ │ │  │
│  │ │ │ Docker Engine    │ │ │  │  │ │ │ │ ↕                │ │ │ │  │
│  │ │ │ (dockerd)        │ │ │  │  │ │ │ │ AF_VSOCK         │ │ │ │  │
│  │ │ │                  │ │ │  │  │ │ │ │ (Shared Memory)  │ │ │ │  │
│  │ │ │ Bound to:        │ │ │  │  │ │ │ │                  │ │ │ │  │
│  │ │ │ 192.168.1.100    │ │ │  │  │ │ │ │ ┌──────────────┐ │ │ │ │  │
│  │ │ │                  │ │ │  │  │ │ │ │ │Docker Engine │ │ │ │  │
│  │ │ │ Swarm Ports:     │ │ │  │  │ │ │ │ │   (dockerd)  │ │ │ │  │
│  │ │ │ :2377 ✅         │ │ │  │  │ │ │ │ │              │ │ │ │  │
│  │ │ │ :7946 ✅         │ │ │  │  │ │ │ │ │ Bound to:    │ │ │ │  │
│  │ │ │ :4789 ✅         │ │ │  │  │ │ │ │ │ 192.168.65.3 │ │ │ │  │
│  │ │ │                  │ │ │  │  │ │ │ │ │              │ │ │ │  │
│  │ │ │ ┌──────────────┐ │ │ │  │  │ │ │ │ │ :2377 ⚠️     │ │ │ │  │
│  │ │ │ │ vxlan0       │ │ │ │  │  │ │ │ │ │ :7946 ⚠️     │ │ │ │  │
│  │ │ │ │ (Kernel)     │ │ │ │  │  │ │ │ │ │ :4789 ⚠️     │ │ │ │  │
│  │ │ │ │              │ │ │ │  │  │ │ │ │ └──────────────┘ │ │ │ │  │
│  │ │ │ │ Direct UDP   │ │ │ │  │  │ │ │ └──────────────────┘ │ │ │  │
│  │ │ │ │ to :4789     │ │ │ │  │  │ │ └────────────────────────┘ │ │  │
│  │ │ │ └──────────────┘ │ │ │  │  │ │                            │ │  │
│  │ │ └──────────────────┘ │ │  │  │ │ ┌────────────────────────┐ │ │  │
│  │ └──────────────────────┘ │  │  │ │ │ VPNKit (Host Process)  │ │ │  │
│  └──────────────────────────┘  │  │ │ │ ↕ AF_VSOCK             │ │ │  │
│                                │  │ │ │ ↕ Internet (Host NIC)  │ │ │  │
│  Network Path for VXLAN:       │  │ │ └────────────────────────┘ │ │  │
│  Container → Kernel VXLAN →    │  │ └──────────────────────────────┘ │  │
│  UDP:4789 → NIC → Network      │  └────────────────────────────────────┘  │
│                                │                                    │
│  ✅ Direct kernel processing   │  ❌ Userspace intermediaries       │
│  ✅ Real IP addresses          │  ❌ Non-routable IP                │
│  ✅ No proxying                │  ❌ AF_VSOCK + VPNKit proxying     │
│  ✅ Source IP preserved        │  ❌ Source IP lost/changed         │
│  ✅ Works across machines      │  ❌ Only works to Internet         │
│                                │                                    │
└────────────────────────────────┴────────────────────────────────────┘

KEY DIFFERENCES:

┌──────────────────┬─────────────────────┬──────────────────────────┐
│   Component      │  Standard Linux     │   Docker Desktop         │
├──────────────────┼─────────────────────┼──────────────────────────┤
│ IP Address       │ 192.168.1.100       │ VM: 192.168.65.3         │
│                  │ (Routable)          │ (Non-routable)           │
├──────────────────┼─────────────────────┼──────────────────────────┤
│ Network Stack    │ Kernel (native)     │ VM Kernel + VPNKit       │
│                  │                     │ (userspace proxy)        │
├──────────────────┼─────────────────────┼──────────────────────────┤
│ VM-Host Comm    │ N/A (no VM)         │ AF_VSOCK                 │
│                  │                     │ (Shared memory)          │
├──────────────────┼─────────────────────┼──────────────────────────┤
│ Internet Access  │ Direct via NIC      │ Via VPNKit proxy         │
├──────────────────┼─────────────────────┼──────────────────────────┤
│ VXLAN Support    │ ✅ Native kernel    │ ❌ Blocked by AF_VSOCK   │
├──────────────────┼─────────────────────┼──────────────────────────┤
│ Peer-to-Peer     │ ✅ Direct UDP       │ ❌ No path to peers      │
├──────────────────┼─────────────────────┼──────────────────────────┤
│ Source IP        │ ✅ Preserved        │ ❌ Lost in proxy         │
├──────────────────┼─────────────────────┼──────────────────────────┤
│ Multi-node Swarm │ ✅ Works            │ ❌ Impossible            │
└──────────────────┴─────────────────────┴──────────────────────────┘
```

---

## 6. Component Interaction Diagrams {#component-interaction}

### 6.1 Complete Packet Flow Comparison

```
╔═════════════════════════════════════════════════════════════════════╗
║            STANDARD LINUX - SWARM COMMUNICATION FLOW                ║
╚═════════════════════════════════════════════════════════════════════╝

Machine A: 192.168.1.100              Machine B: 192.168.1.101
┌─────────────────────┐                ┌─────────────────────┐
│ Container A         │                │ Container B         │
│ 172.18.0.2          │                │ 172.18.0.3          │
└──────────┬──────────┘                └──────────▲──────────┘
           │                                      │
           │ [1] Send data                        │ [9] Receive data
           │                                      │
           ▼                                      │
┌───────────────────────┐                         │
│ Bridge                │                         │
└───────────┬───────────┘                         │
            │                                     │
            │ [2] Route to overlay                │
            │                                     │
            ▼                                     │
┌───────────────────────────────────────┐         │
│ VXLAN Module (Kernel)                 │         │
│ • Lookup FDB                           │         │
│ • Encapsulate with VXLAN/UDP/IP       │         │
│ • Outer IP: .100 → .101                │         │
└────────────┬──────────────────────────┘         │
             │                                     │
             │ [3] Send encapsulated packet        │
             │                                     │
             ▼                                     │
┌────────────────────────┐                         │
│ Physical NIC           │                         │
│ (Direct kernel access) │                         │
└────────────┬───────────┘                         │
             │                                     │
             │ [4] Physical network                │
             │                                     │
             ▼                                     │
        ═════════════════════════════              │
        ║  Network Switch/Router  ║               │
        ═════════════════════════════              │
                       │                           │
                       │ [5] Route packet          │
                       │                           │
                       ▼                           │
          ┌────────────────────────┐               │
          │ Physical NIC           │               │
          │ (Machine B)            │               │
          └────────────┬───────────┘               │
                       │                           │
                       │ [6] Receive at NIC        │
                       │                           │
                       ▼                           │
          ┌────────────────────────────────┐       │
          │ VXLAN Module (Kernel)          │       │
          │ • Receive UDP:4789             │       │
          │ • Decapsulate VXLAN            │       │
          │ • Learn FDB entry              │       │
          │ • Extract inner frame          │       │
          └────────────┬───────────────────┘       │
                       │                           │
                       │ [7] Forward inner frame   │
                       │                           │
                       ▼                           │
          ┌────────────────────────┐               │
          │ Bridge                 │               │
          └────────────┬───────────┘               │
                       │                           │
                       │ [8] Deliver to container  │
                       │                           │
                       └───────────────────────────┘

Processing Points: 6 (all in kernel)
Latency: ~10-20 microseconds
Source IP: Preserved throughout
Result: ✅ SUCCESS


╔═════════════════════════════════════════════════════════════════════╗
║         DOCKER DESKTOP - ATTEMPTED SWARM COMMUNICATION              ║
╚═════════════════════════════════════════════════════════════════════╝

Machine A - Docker Desktop           Machine B - Docker Desktop
┌─────────────────────┐               ┌─────────────────────┐
│ Container A         │               │ Container B         │
│ 172.18.0.2          │               │ 172.18.0.3          │
└──────────┬──────────┘               └──────────┬──────────┘
           │                                     │
           │ [1] Send data                       │ Never
           │                                     │ receives
           ▼                                     ▼
┌───────────────────────┐                        ❌
│ Bridge                │
└───────────┬───────────┘
            │
            │ [2] Route to overlay
            │
            ▼
┌───────────────────────────────────────┐
│ VXLAN Module (Kernel - in VM)        │
│ • Lookup FDB                          │
│ • Encapsulate with VXLAN/UDP/IP      │
│ • Outer IP: 192.168.65.3 → 65.4      │
│   (Both non-routable VM IPs!)        │
└────────────┬──────────────────────────┘
             │
             │ [3] Routing decision
             │
             ▼
┌──────────────────────────┐
│ VM Kernel Routing        │
│ Dest: 192.168.65.4       │
│ Route: via 192.168.65.1  │
└────────────┬─────────────┘
             │
             │ [4] Send to gateway
             │
             ▼
┌──────────────────────────────┐
│ eth0 (NOT a real NIC!)       │
│ Connected to: AF_VSOCK       │
└────────────┬─────────────────┘
             │
             │ [5] Transmit via shared memory
             │
             ▼
        ═════════════════
        ║   AF_VSOCK    ║
        ║ Shared Memory ║
        ═════════════════
             │
             │ [6] Shared memory transfer
             │
             ▼
┌──────────────────────────────────────┐
│ VPNKit Process (Userspace - Host A) │
│                                      │
│ [7] Parse packet                     │
│     Source: 192.168.65.3             │
│     Dest: 192.168.65.4               │
│     Port: 4789                       │
│                                      │
│ [8] Decision Logic:                  │
│     if (internet address):           │
│         proxy to internet ✅         │
│     else if (vpnkit service):        │
│         route internally ✅          │
│     else if (192.168.65.x):          │
│         ❌ NO HANDLER                │
│         ❌ Unknown destination       │
│         ❌ Not internet              │
│         ❌ Not internal service      │
│         ❌ Not on host network       │
│                                      │
│ [9] PACKET DROPPED                   │
└──────────────────────────────────────┘
             │
             ▼
            ❌ FAILURE
         Communication
           impossible

Processing Points: 9 (crosses kernel/user boundary)
Latency: N/A (fails before completion)
Source IP: Would be lost even if succeeded
Result: ❌ FAILURE - No path to remote VM


═══════════════════════════════════════════════════════════════════

THE FUNDAMENTAL DIFFERENCE:

Standard Linux:
  Container → Kernel → Physical NIC → Network → Remote NIC → Kernel → Container
  (Direct path, all in kernel, works across machines)

Docker Desktop:
  Container → VM Kernel → AF_VSOCK → VPNKit → ??? → BLOCKED
  (No path to remote machine's VM)

═══════════════════════════════════════════════════════════════════
```

---

## Summary

These diagrams illustrate the complete technical story:

**What Works in Docker Desktop:**
- Container-to-Internet communication (VPNKit's design purpose)
- Single-host port publishing (for local development)
- All single-machine container operations

**What Fails in Docker Desktop:**
- Multi-node VXLAN overlay networks (no path between VMs)
- Peer-to-peer Docker Swarm communication (AF_VSOCK only reaches host)
- Distributed container orchestration (non-routable VM IPs)

**The Root Causes:**
1. AF_VSOCK creates a VM↔Host channel, not VM↔VM
2. VPNKit proxies VM→Internet, not VM→VM
3. Port forwarding loses source IP information
4. VM IPs are non-routable outside the hypervisor

Each of these design decisions makes perfect sense for Docker Desktop's purpose (single-machine development with VPN compatibility) but creates insurmountable barriers for multi-node orchestration.
