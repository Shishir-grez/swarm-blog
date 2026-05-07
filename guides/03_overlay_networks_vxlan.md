# Docker Overlay Networks & VXLAN: Complete Guide

## What You'll Learn
- What overlay networks are and why they exist
- How VXLAN works under the hood
- How Docker Swarm uses VXLAN for cross-host communication
- Why all nodes need reachable IPs

**Prerequisites**: None! We'll explain all networking concepts from scratch.

---

## Network Fundamentals

Before diving into overlay networks, let's understand the networking building blocks. If you're already familiar with networking layers, feel free to skip to [Part 1](#part-1-the-problem-overlay-networks-solve).

### The OSI Model (Networking Layers)

The **OSI Model** is a framework that describes how networks work in 7 layers. For our purposes, we'll focus on Layers 2, 3, and 4.

**The 7 Layers (simplified):**
```
Layer 7: Application  (HTTP, FTP, SSH)
Layer 6: Presentation (Encryption, compression)
Layer 5: Session      (Connections)
Layer 4: Transport    (TCP, UDP) ← We care about this
Layer 3: Network      (IP addresses) ← We care about this
Layer 2: Data Link    (MAC addresses, Ethernet) ← We care about this
Layer 1: Physical     (Cables, WiFi signals)
```

**Why layers matter:**
Each layer adds its own header to the data, like wrapping a package multiple times.

### Layer 2: Data Link (Ethernet)

**What it does:** Moves data between devices on the same local network.

**Key concept: MAC Address**
- **MAC** = Media Access Control
- A unique hardware address for each network card
- Format: `00:1A:2B:3C:4D:5E` (6 pairs of hex digits)
- **Permanent**: Burned into the network card at the factory

**Example:**
```
Your laptop's MAC:    00:1A:2B:3C:4D:5E
Your router's MAC:    AA:BB:CC:DD:EE:FF

When sending data on your local network:
From MAC: 00:1A:2B:3C:4D:5E
To MAC:   AA:BB:CC:DD:EE:FF
```

**Ethernet Frame Structure:**
```
┌────────────────────────────────────┐
│ Destination MAC (6 bytes)          │
│ Source MAC (6 bytes)               │
│ Type (2 bytes)                     │
│ Payload (46-1500 bytes)            │
│ CRC Checksum (4 bytes)             │
└────────────────────────────────────┘
```

**Important:** MAC addresses only work on the same local network. To reach the internet, you need Layer 3.

### Layer 3: Network (IP)

**What it does:** Routes data across different networks (the internet).

**Key concept: IP Address**
- Already covered in Part 1, but quick recap:
- Format: `192.168.1.100`
- Can change (unlike MAC addresses)
- Used for routing across networks

**IP Packet Structure:**
```
┌────────────────────────────────────┐
│ Version, Header Length             │
│ Source IP Address                  │
│ Destination IP Address             │
│ TTL (Time to Live)                 │
│ Protocol (TCP=6, UDP=17)           │
│ Payload (your data)                │
└────────────────────────────────────┘
```

**How IP and MAC work together:**

| Property | MAC Address | IP Address |
|----------|-------------|------------|
| **Layer** | Layer 2 (Data Link) | Layer 3 (Network) |
| **Scope** | Same local network only | Across all networks |
| **Changes?** | Never (burned into hardware) | Can change (dynamic) |
| **Analogy** | Next relay runner to hand baton to | Final destination address |

**The Relay Runner Analogy:**
```
You're in New York, sending a package to a friend in Tokyo.

IP Address = "123 Sakura Street, Tokyo, Japan"
             This is the FINAL DESTINATION.
             Every handler along the way reads this to know the goal.

MAC Address = "Hand this to the next person" (changes at every step)

Step 1 (Your house → Post office):   Your hands → Clerk
Step 2 (Post office → Airport):      Clerk → Airport worker
Step 3 (JFK → Narita):               American plane → Japanese ground crew
Step 4 (Courier → Friend):           Delivery person → Friend's hands

At EVERY step, the IP (final destination) stays the same.
But the MAC (immediate next handoff) changes.
```

**In networking:**
```
Your laptop (192.168.1.100) wants to reach google.com (142.250.80.14)

STEP 1: Laptop prepares packet
┌────────────────────────────────────┐
│ IP Header:                         │
│   Source: 192.168.1.100            │
│   Dest:   142.250.80.14 (Google)   │  ← This NEVER changes
│                                    │
│ Ethernet Header:                   │
│   Source MAC: AA:AA:AA:AA:AA:AA    │  ← Your laptop's MAC
│   Dest MAC:   BB:BB:BB:BB:BB:BB    │  ← Your ROUTER's MAC (not Google!)
└────────────────────────────────────┘

"Wait, why router's MAC, not Google's?"
Because your laptop can only "hand off" to devices on the same local network!
Your router is the next hop.

STEP 2: Router receives, creates NEW ethernet header
┌────────────────────────────────────┐
│ IP Header: (unchanged)             │
│   Source: 192.168.1.100            │
│   Dest:   142.250.80.14            │  ← Still the same!
│                                    │
│ NEW Ethernet Header:               │
│   Source MAC: BB:BB:BB:BB:BB:BB    │  ← Router's MAC
│   Dest MAC:   CC:CC:CC:CC:CC:CC    │  ← ISP's MAC (next hop)
└────────────────────────────────────┘

This process repeats at EVERY router hop until Google is reached.
```

> **Key Insight:** MAC addresses are for "who do I hand this to next?" IP addresses are for "where does this ultimately go?"

### Layer 4: Transport (TCP and UDP)

**What it does:** Manages how data is sent between applications.

#### TCP (Transmission Control Protocol)

**Characteristics:**
- **Reliable**: Guarantees delivery
- **Ordered**: Packets arrive in order
- **Connection-based**: Handshake before sending data
- **Slower**: Due to reliability overhead

**Use cases:**
- Web browsing (HTTP/HTTPS)
- Email (SMTP, IMAP)
- File transfers (FTP)
- Anything where you can't lose data

**How it works:**
```
Client                    Server
  │                         │
  ├──── SYN ───────────────>│  "Want to connect?"
  │<──── SYN-ACK ───────────┤  "Sure, ready!"
  ├──── ACK ───────────────>│  "Great, let's go!"
  │                         │
  ├──── Data ──────────────>│
  │<──── ACK ───────────────┤  "Got it!"
  │                         │
```

#### UDP (User Datagram Protocol)

**Characteristics:**
- **Unreliable**: No delivery guarantee
- **Unordered**: Packets may arrive out of order
- **Connectionless**: No handshake
- **Faster**: Minimal overhead

**Use cases:**
- Video streaming (some packet loss is OK)
- Online gaming (speed > reliability)
- DNS queries (small, can retry)
- **VXLAN** (what this guide is about!)

**How it works:**
```
Client                    Server
  │                         │
  ├──── Data ──────────────>│  "Here's data!" (no confirmation)
  ├──── Data ──────────────>│  "More data!"
  │                         │
```

**Why VXLAN uses UDP:**
- Fast (no handshake overhead)
- Stateless (no connection tracking)
- Works through firewalls (port 4789)

### Packets vs Frames

**Frame** = Layer 2 (Ethernet)
```
┌─────────────────────────────────┐
│ MAC Header                      │
│ ┌─────────────────────────────┐ │
│ │ IP Packet                   │ │
│ │ ┌─────────────────────────┐ │ │
│ │ │ TCP/UDP Segment         │ │ │
│ │ │ ┌─────────────────────┐ │ │ │
│ │ │ │ Your Data           │ │ │ │
│ │ │ └─────────────────────┘ │ │ │
│ │ └─────────────────────────┘ │ │
│ └─────────────────────────────┘ │
└─────────────────────────────────┘
```

**Terminology:**
- **Frame**: Layer 2 (includes MAC addresses)
- **Packet**: Layer 3 (includes IP addresses)
- **Segment**: Layer 4 (TCP) or **Datagram**: Layer 4 (UDP)

**In practice:**
People often say "packet" to mean any of these. Context matters!

### What is Encapsulation?

**Encapsulation** is wrapping data in headers as it goes down the network layers.

**Sending an email:**
```
Layer 7: Email message
         ↓ (add SMTP header)
Layer 4: TCP segment [SMTP header | Email]
         ↓ (add IP header)
Layer 3: IP packet [IP header | TCP segment]
         ↓ (add Ethernet header)
Layer 2: Ethernet frame [Ethernet | IP packet]
         ↓
Layer 1: Electrical signals on wire
```

**Receiving the email:**
```
Layer 1: Electrical signals
         ↓ (remove Ethernet header)
Layer 2: IP packet
         ↓ (remove IP header)
Layer 3: TCP segment
         ↓ (remove TCP header)
Layer 4: Email message
```

**Why this matters for VXLAN:**
VXLAN does **double encapsulation**:
```
Original packet (Container A → Container B)
    ↓ Encapsulate in VXLAN
VXLAN packet
    ↓ Encapsulate in UDP/IP/Ethernet
Physical network packet (Host A → Host B)
```

### What is Tunneling?

**Tunneling** is sending one type of network traffic inside another.

#### Why Do We Need Tunnels?

Imagine two apartment buildings in different cities. Each building has apartments numbered 1-10. If someone in Apartment 5 of Building A writes a letter addressed to "Apartment 5", the postal system doesn't know which building's Apartment 5 you mean.

**In networking terms:**
- Buildings = Docker hosts (physical/virtual machines)
- Apartments = Containers
- Apartment numbers = Container IP addresses (like 10.0.0.5)

```
Host A (Chicago)                     Host B (New York)
┌───────────────────┐               ┌───────────────────┐
│ Container: 10.0.0.5 │               │ Container: 10.0.0.5 │
│ Container: 10.0.0.6 │               │ Container: 10.0.0.6 │
└───────────────────┘               └───────────────────┘

Problem: How does a packet from 10.0.0.5 on Host A 
         reach 10.0.0.6 on Host B?

The internet has no idea how to route to 10.0.0.x!
These are PRIVATE addresses that exist inside each host.
```

**Physical network routers only understand:**
- Host A's real IP: 203.0.113.10 ✓
- Host B's real IP: 198.51.100.20 ✓
- Container IP 10.0.0.5: ❌ "What? Never heard of it"

**Container IPs are invisible to the outside world.** They're like internal apartment numbers that only make sense inside each building.

#### The Tunnel Solution

A tunnel is like putting a letter inside another envelope with REAL addresses:

```
WITHOUT TUNNEL (fails):
┌─────────────────────────────┐
│ From: 10.0.0.5              │  ← Internet routers: "Where do I send this?"
│ To:   10.0.0.6              │
│ Data: "Hello"               │
└─────────────────────────────┘

WITH TUNNEL (works):
┌──────────────────────────────────────────────────────────┐
│ OUTER ENVELOPE (for internet)                            │
│ From: 203.0.113.10 (Host A)                              │  ← Routers understand this!
│ To:   198.51.100.20 (Host B)                             │
│                                                          │
│   ┌────────────────────────────────────────────────────┐ │
│   │ INNER ENVELOPE (original container message)        │ │
│   │ From: 10.0.0.5                                     │ │
│   │ To:   10.0.0.6                                     │ │
│   │ Data: "Hello"                                      │ │
│   └────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

**When Host B receives this:**
1. Opens the outer envelope (sees it's from Host A)
2. Opens the inner envelope (sees it's for container 10.0.0.6)
3. Delivers to the correct container

> **Key Insight:** Tunneling = Wrapping private-network packets inside public-network packets. This allows containers on different hosts to communicate as if they're on the same local network, even when they're across the internet.

**Common tunneling protocols:**
- **VXLAN**: Ethernet over UDP (this guide!)
- **GRE**: Generic Routing Encapsulation
- **IPsec**: Encrypted IP tunnels
- **SSH**: Secure Shell tunneling

**Why tunnel?**
- Connect networks across the internet
- Bypass network restrictions
- Add encryption
- Create virtual networks (like Docker overlay networks)

### What is ARP (Address Resolution Protocol)?

**ARP** finds the MAC address for a given IP address on your local network.

**The problem:**
```
You know: "I want to send to 192.168.1.10"
You need: "What's the MAC address of 192.168.1.10?"
```

**How ARP works:**
```
1. Your computer broadcasts: "Who has 192.168.1.10?"
2. Device with that IP responds: "I do! My MAC is AA:BB:CC:DD:EE:FF"
3. Your computer caches this: 192.168.1.10 → AA:BB:CC:DD:EE:FF
4. Now you can send Ethernet frames to that MAC
```

**Why VXLAN doesn't need ARP:**
- VXLAN uses point-to-point tunnels
- No broadcast domain
- MAC addresses are learned via control plane
- More efficient for large networks

### What is Multicast?

**Multicast** sends data to multiple recipients simultaneously.

**Three types of communication:**

**Unicast** (one-to-one):
```
Sender → Receiver
```

**Broadcast** (one-to-all):
```
Sender → Everyone on the network
```

**Multicast** (one-to-many):
```
Sender → Group of interested receivers
```

**Example:**
```
Video streaming:
- Unicast: Netflix sends separate stream to each viewer (wasteful)
- Multicast: One stream, multiple viewers (efficient)
```

**In VXLAN context:**
- Early VXLAN used multicast for discovery
- Modern VXLAN uses control plane (no multicast needed)
- Simpler, works better across networks

---

## Part 1: The Problem Overlay Networks Solve

### Scenario: Multi-Host Container Communication

Imagine you have:
- **Host A** (192.168.1.10): Running Container 1
- **Host B** (192.168.1.20): Running Container 2

You want Container 1 and Container 2 to talk to each other as if they're on the same local network.

### The Challenge
```
Container 1 (172.17.0.2) on Host A
    ↓
    How do I reach Container 2 (172.17.0.3) on Host B?
    ↓
Container 2 (172.17.0.3) on Host B
```

**Problem**: Container IPs are private to each host. The physical network doesn't know how to route `172.17.0.0/16` traffic.

---

## Part 2: What is an Overlay Network?

### Simple Definition
> An overlay network is a **virtual network** built **on top of** an existing physical network.

### The Layers
```
┌─────────────────────────────────────┐
│   Overlay Network (Virtual)         │  ← Containers see this
│   10.0.0.0/24                       │
└─────────────────────────────────────┘
              ▲
              │ Encapsulation
              ▼
┌─────────────────────────────────────┐
│   Underlay Network (Physical)       │  ← Actual network infrastructure
│   192.168.1.0/24                    │
└─────────────────────────────────────┘
```

### Key Concept: Encapsulation
Overlay networks work by **wrapping** (encapsulating) the container's network packet inside another packet that can travel over the physical network.

---

## Part 3: Enter VXLAN

### What is VXLAN?
**VXLAN** = Virtual Extensible LAN

It's a **tunneling protocol** that:
- Encapsulates Layer 2 (Ethernet) frames
- Inside Layer 4 (UDP) packets
- Allowing them to travel over Layer 3 (IP) networks

### Why VXLAN?
Traditional VLANs have a limit of **4096** networks (12-bit VLAN ID).

VXLAN uses a **24-bit VNI** (VXLAN Network Identifier), allowing **16 million** virtual networks!

---

## Part 4: How VXLAN Works

### The Packet Structure
```
Original Container Packet:
┌──────────────────────────────────────┐
│ Ethernet Header (Container MAC)     │
│ IP Header (Container IP: 10.0.0.2)  │
│ TCP/UDP Header                       │
│ Application Data                     │
└──────────────────────────────────────┘

After VXLAN Encapsulation:
┌──────────────────────────────────────┐
│ Outer Ethernet Header (Host MAC)    │  ← Physical network
│ Outer IP Header (Host IP: 192.168.1.10) │
│ UDP Header (Dest Port: 4789)        │  ← VXLAN uses port 4789
│ VXLAN Header (VNI: 42)              │  ← Network ID
│ ┌────────────────────────────────┐  │
│ │ Original Container Packet      │  │  ← Encapsulated
│ └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

### The VXLAN Header
```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|R|R|R|R|I|R|R|R|            Reserved                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                VXLAN Network Identifier (VNI) |   Reserved    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**VNI** = The "network ID" that isolates different overlay networks.

### VNI: Network Isolation Deep Dive

Think of VNI as an apartment building number:

```
Without VNI (single overlay network):
┌─────────────────────────────────────────────────────────────┐
│ ALL containers on the SAME virtual network                   │
│                                                              │
│ Container A (web)      can see    Container X (db)          │
│ Container B (payments) can see    Container Y (cache)       │
│                                                              │
│ NO ISOLATION! Security nightmare!                            │
└─────────────────────────────────────────────────────────────┘

With VNI (multiple isolated overlay networks):
┌─────────────────────────────────────────────────────────────┐
│ VNI 100: Web Frontend Network                                │
│   Container A, Container B                                   │
├─────────────────────────────────────────────────────────────┤
│ VNI 200: Database Network                                    │
│   Container X, Container Y                                   │
├─────────────────────────────────────────────────────────────┤
│ VNI 300: Payment Processing Network                          │
│   Container P, Container Q                                   │
└─────────────────────────────────────────────────────────────┘

Containers in VNI 100 CANNOT see containers in VNI 200!
Complete isolation at the network level.
```

**Why 24 bits for VNI?**
- Traditional VLAN: 12 bits = 4,096 networks
- VNI: 24 bits = **16,777,216** networks
- This is why it's called "Virtual **Extensible** LAN"!

**VNI Isolation Example:**
```
Host A                                    Host B
┌───────────────────┐                    ┌───────────────────┐
│ Container 1       │                    │ Container 3       │
│ frontend-net      │                    │ frontend-net      │
│ VNI: 4097         │                    │ VNI: 4097         │
├───────────────────┤                    ├───────────────────┤
│ Container 2       │                    │ Container 4       │
│ backend-net       │                    │ backend-net       │
│ VNI: 4098         │                    │ VNI: 4098         │
└─────────┬─────────┘                    └─────────┬─────────┘
          │                                        │
          │        Physical Network                │
          └────────────────────────────────────────┘

Container 1 → Container 3: ✓ (same VNI 4097)
Container 2 → Container 4: ✓ (same VNI 4098)
Container 1 → Container 4: ✗ (different VNIs!)
Container 2 → Container 3: ✗ (different VNIs!)
```

---

## Part 5: VXLAN Tunnel Endpoints (VTEPs)

### What is a VTEP?
**VTEP** = VXLAN Tunnel Endpoint

The VTEP is responsible for:
1. **Encapsulating** packets leaving the overlay network
2. **Decapsulating** packets entering the overlay network

### In Docker Swarm
Each Docker host acts as a VTEP.

```
┌─────────────────────────────────────────────────┐
│              Host A (192.168.1.10)              │
│                                                 │
│  ┌──────────────┐                               │
│  │ Container 1  │                               │
│  │ 10.0.0.2     │                               │
│  └──────┬───────┘                               │
│         │                                       │
│    ┌────▼─────┐                                 │
│    │  VTEP    │ ← Encapsulates packets          │
│    │  (vxlan) │                                 │
│    └────┬─────┘                                 │
│         │                                       │
│    ┌────▼─────────────────────────────┐         │
│    │  Physical NIC (192.168.1.10)    │         │
│    └─────────────────────────────────┘         │
└─────────────────────────────────────────────────┘
                    │
                    │ UDP packet (port 4789)
                    ▼
┌─────────────────────────────────────────────────┐
│              Host B (192.168.1.20)              │
│                                                 │
│    ┌─────────────────────────────────┐         │
│    │  Physical NIC (192.168.1.20)    │         │
│    └────┬─────────────────────────────┘         │
│         │                                       │
│    ┌────▼─────┐                                 │
│    │  VTEP    │ ← Decapsulates packets          │
│    │  (vxlan) │                                 │
│    └────┬─────┘                                 │
│         │                                       │
│  ┌──────▼───────┐                               │
│  │ Container 2  │                               │
│  │ 10.0.0.3     │                               │
│  └──────────────┘                               │
└─────────────────────────────────────────────────┘
```

### Decapsulation: How Packets Are "Unloaded" at the Receiver

In VXLAN, there are TWO complete packet layers. Decapsulation happens in two phases:

```
What Host B receives (the outer packet):
┌─────────────────────────────────────────────────────────────┐
│ Outer Ethernet: [Host A MAC → Host B MAC]                   │
│ Outer IP:       [Host A IP → Host B IP]                     │
│ Outer UDP:      [Port 4789]                                 │
│ VXLAN Header:   [VNI: 256]                                  │
│                                                              │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ Inner Ethernet: [Container 1 MAC → Container 2 MAC] │   │
│   │ Inner IP:       [10.0.0.5 → 10.0.0.6]               │   │
│   │ Inner TCP:      [Port 80]                            │   │
│   │ Data:           "GET /api"                           │   │
│   └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**Phase 1: Regular Network Stack Processing**
```
1. NIC receives packet
   "Dest MAC is mine? Yes, accept"
   → Strips outer Ethernet header
   
2. IP stack receives
   "Dest IP is mine? Yes"
   "Protocol is UDP, port 4789"
   → Passes to UDP handler
   
3. UDP handler 
   "Port 4789 = VXLAN"
   → Passes to VXLAN kernel module (VTEP)
```

**Phase 2: VTEP Processing**
```
4. VTEP receives the VXLAN payload
   Reads VXLAN header: VNI = 256
   "I know network 256!"
   → Strips VXLAN header
   
5. Now we have the INNER packet!
   [Container 1 MAC → Container 2 MAC]
   [10.0.0.5 → 10.0.0.6 port 80]
   
   VTEP looks at inner dest MAC
   "Container 2's MAC... that container is on bridge br-xxx"
   → Injects packet into bridge network
   
6. Container 2 receives packet
   Standard decapsulation of inner packet
   Container app gets "GET /api"
```

> **The Beautiful Illusion:** From Container 2's perspective, it received a packet from 10.0.0.5 (Container 1). It has NO IDEA the packet traveled across the internet. It thinks Container 1 is on the same local network!

---

## Part 6: Docker Swarm Overlay Networks

### How Docker Uses VXLAN

When you create an overlay network in Docker Swarm:
```bash
docker network create --driver overlay my-network
```

Docker:
1. Assigns a **VNI** (e.g., VNI 256)
2. Creates a **VTEP** on each Swarm node
3. Distributes network info via the **control plane** (Raft consensus)

### Understanding the Control Plane

**Control Plane vs Data Plane:**

| Aspect | Control Plane | Data Plane |
|--------|--------------|------------|
| **What** | Decisions about WHERE data goes | Actually MOVING the data |
| **Analogy** | GPS navigation system | The car's engine/wheels |
| **Frequency** | Occasional (when routes change) | Constant (every packet) |
| **Ports** | 2377, 7946 | 4789 |

```
CONTROL PLANE (Port 2377, 7946):
┌─────────────────────────────────────────────────────────────┐
│ "Container X is on Host B at IP 100.66.197.9"               │
│ "Service web now has 3 replicas"                            │
│ "Network my-overlay uses VNI 256"                           │
│ "Container X's MAC is AA:BB:CC:DD:EE:FF"                    │
│                                                              │
│ This information flows between managers and workers          │
│ so everyone KNOWS where everything is.                       │
└─────────────────────────────────────────────────────────────┘
                         ↓ informs
DATA PLANE (Port 4789 - VXLAN):
┌─────────────────────────────────────────────────────────────┐
│ *Actual HTTP requests between containers*                    │
│ *Database queries*                                           │
│ *File transfers*                                             │
│                                                              │
│ This is the real work – using the routes that control plane │
│ established.                                                 │
└─────────────────────────────────────────────────────────────┘
```

**Why control plane matters for VXLAN:**

Traditional networks use ARP broadcasts to discover MAC→IP mappings:
```
Traditional LAN:
  "Who has IP 10.0.0.5?" → Broadcast to everyone
  Device responds: "I do! MAC is AA:BB:CC"
```

But in VXLAN, hosts might be across the internet—broadcasting won't work!

```
VXLAN with Control Plane:
  Manager already told everyone:
    "Container service.web.1 (10.0.0.5) is on Host B"
    "Its MAC is AA:BB:CC:DD:EE:FF"
    "Host B's VTEP IP is 100.66.197.9"
  
  No broadcasting needed! Everyone already knows.
```

### The Flow

#### Step 1: Container 1 sends packet to Container 2
```
Container 1 (10.0.0.2) → Container 2 (10.0.0.3)
```

#### Step 2: Packet reaches VTEP on Host A
VTEP checks:
- Destination MAC: Container 2's MAC
- Looks up: "Which host has Container 2?"
- Answer: Host B (192.168.1.20)

#### Step 3: Encapsulation
```
Original packet wrapped in:
- Outer IP: 192.168.1.10 → 192.168.1.20
- UDP port: 4789
- VXLAN header: VNI 256
```

#### Step 4: Transmission
Packet travels over physical network to Host B.

#### Step 5: Decapsulation on Host B
VTEP on Host B:
- Receives UDP packet on port 4789
- Checks VNI: 256 (matches my-network)
- Extracts original packet
- Delivers to Container 2

---

## Part 7: Why All Nodes Need Reachable IPs

### The Critical Requirement
For VXLAN to work, **VTEPs must be able to reach each other** over the underlay network.

### Your Tailscale Scenario
```
Host A (Manager):
- Tailscale IP: 100.66.197.8 ✓
- VTEP can bind to this IP ✓

Host B (Worker):
- Advertises IP: 172.X.X.Y (WSL2 internal)
- Manager tries to send VXLAN packet to 172.X.X.Y
- ❌ Can't reach it! (NAT, not routable)
```

**Result**: Overlay network breaks.

### The Fix
Both nodes must advertise **routable IPs**:
```
Host A: 100.66.197.8 (Tailscale)
Host B: 100.66.197.9 (Tailscale)

Now:
- Manager sends VXLAN to 100.66.197.9 ✓
- Worker sends VXLAN to 100.66.197.8 ✓
- Overlay network works! ✓
```

---

## Part 8: Hands-On: Inspecting VXLAN

### Create an Overlay Network
```bash
# On Swarm manager
docker network create --driver overlay --attachable test-overlay
```

### Inspect the Network
```bash
docker network inspect test-overlay
```

Look for:
```json
"Driver": "overlay",
"Options": {
    "com.docker.network.driver.overlay.vxlanid_list": "4097"
}
```

**VNI = 4097** for this network.

### See the VTEP Interface
```bash
# On a Swarm node
ip -d link show

# Look for something like:
# vxlan: vxlan id 4097 ...
```

### Capture VXLAN Traffic
```bash
# Install tcpdump
sudo tcpdump -i eth0 'udp port 4789' -vv

# Run a container and ping another container
# You'll see VXLAN encapsulated packets!
```

---

## Part 9: Docker Swarm Ports (Deep Dive)

### Required Ports for Overlay Networks
| Port | Protocol | Purpose | Official Standard? |
|------|----------|---------|-------------------|
| 2377 | TCP | Cluster management (Raft) | No (Docker choice) |
| 7946 | TCP/UDP | Node discovery (gossip) | No (Docker choice) |
| 4789 | UDP | VXLAN overlay traffic | **Yes** (IANA/RFC 7348) |

### Port 2377: Cluster Management (TCP)

**What it's for:** The "command center" of Docker Swarm

```
┌─────────────────────────────────────────────────────────────┐
│ Manager Node - Port 2377 handles:                            │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ • Raft Consensus (leader election)                    │   │
│  │ • Join tokens (new nodes joining)                     │   │
│  │ • Cluster state changes                               │   │
│  │ • Service deployment commands                         │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**Why TCP?**
- "Create service web-server with 3 replicas" — you can't lose this command!
- Raft consensus requires ordered, reliable delivery
- TCP guarantees messages arrive in order and without loss

### Port 7946: Node Discovery (TCP + UDP)

**What it's for:** Nodes gossiping about who's alive and who's dead

```
Node A                    Node B                    Node C
  │                         │                         │
  ├── "I'm alive!" ────────>│                         │
  │<── "So am I!" ──────────┤                         │
  │                         ├── "Node A is alive" ───>│
  │                         │<── "Confirmed" ─────────┤
  │<── "Node C is alive" ───┤                         │
  
This is "gossip protocol" – nodes constantly share status info.
```

**Why both TCP and UDP?**
- **UDP:** Fast heartbeats ("ping, I'm alive") — losing one isn't critical
- **TCP:** Important state sync (membership changes) — must be reliable

### Port 4789: VXLAN Overlay Traffic (UDP)

**What it's for:** The actual data traveling between containers

```
Container A → "GET /api/user" → Container B

This request travels as:
[Outer UDP packet on port 4789]
  └── [VXLAN header]
       └── [Original HTTP request]
```

**Why UDP (not TCP)?**
- **Speed:** No connection handshake overhead
- **Avoid TCP-over-TCP:** The encapsulated data might already be TCP. TCP tunneling TCP causes terrible "retransmit storm" performance
- **Stateless:** Each packet is independent, no connection state to track

**Why 4789 specifically?**
- This IS an official IANA standard (RFC 7348)
- Universal agreement means firewalls can have pre-built rules
- Any VXLAN implementation uses this port

### Why These Ports Matter
If **any** of these ports are blocked between nodes:
- 2377 blocked → Can't join cluster
- 7946 blocked → Nodes can't discover each other
- 4789 blocked → **Overlay network fails**

---

## Part 10: Advanced: Encrypted Overlay Networks

### Enable Encryption
```bash
docker network create \
  --driver overlay \
  --opt encrypted \
  secure-network
```

### What Changes
Docker adds **IPsec encryption** to VXLAN traffic:
```
VXLAN packet:
┌────────────────────────────┐
│ Outer IP Header            │
│ UDP Header (4789)          │
│ VXLAN Header               │
│ ┌────────────────────────┐ │
│ │ IPsec Encrypted Data   │ │  ← Encrypted!
│ └────────────────────────┘ │
└────────────────────────────┘
```

**Trade-off**: ~10-15% performance overhead.

---

## Part 11: Troubleshooting Overlay Networks

### Symptom: Containers can't ping each other across hosts

#### Check 1: Are nodes in the Swarm?
```bash
docker node ls
```

#### Check 2: Is port 4789 open?
```bash
# On each node
sudo netstat -tulpn | grep 4789
```

#### Check 3: Can VTEPs reach each other?
```bash
# From Host A, ping Host B's advertise IP
ping <host-b-ip>
```

#### Check 4: Check VXLAN interface
```bash
ip -d link show | grep vxlan
```

If no VXLAN interface exists, the overlay network isn't set up.

---

## Key Takeaways

1. **Overlay networks = Virtual networks on top of physical networks**
2. **VXLAN = Tunneling protocol** (encapsulates L2 in UDP)
3. **VTEPs = Endpoints** that encapsulate/decapsulate packets
4. **Docker Swarm uses VXLAN** for cross-host container communication
5. **All nodes need reachable IPs** for VTEPs to communicate
6. **Port 4789 must be open** between all Swarm nodes

---

## Further Reading
- [VXLAN RFC 7348](https://datatracker.ietf.org/doc/html/rfc7348)
- [Docker Overlay Network Driver](https://docs.docker.com/network/drivers/overlay/)
- [Understanding VXLAN (Medium)](https://medium.com/@ODOWDAIBM/vxlan-introduction-d4b4f3ae0e6f)
