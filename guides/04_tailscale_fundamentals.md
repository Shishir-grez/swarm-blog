# Tailscale Fundamentals: Complete Guide

## What You'll Learn
- What Tailscale is and how it works
- WireGuard protocol basics
- Why Tailscale uses `/32` subnet masks
- Userspace vs kernel networking
- How NAT traversal works (STUN, DERP)

**Prerequisites**: None! We'll explain VPNs and networking from the ground up.

---

## VPN and Security Fundamentals

Before diving into Tailscale, let's understand VPNs, encryption, and mesh networking. If you're already familiar with these concepts, skip to [Part 1](#part-1-what-is-tailscale).

### What is a VPN (Virtual Private Network)?

A **VPN** creates a secure, encrypted connection over the internet, making it appear as if you're on a different network.

**Without VPN:**
```
Your Computer → Internet (unencrypted) → Website
     ↓
Anyone can see:
- What websites you visit
- Your data (if not HTTPS)
- Your real IP address
```

**With VPN:**
```
Your Computer → VPN Tunnel (encrypted) → VPN Server → Website
     ↓
Others see:
- Encrypted gibberish
- VPN server's IP (not yours)
- Can't tell what you're doing
```

**Common VPN use cases:**
1. **Privacy**: Hide your internet activity from ISP
2. **Security**: Protect data on public WiFi
3. **Access**: Bypass geographic restrictions
4. **Remote work**: Access company network from home

**Traditional VPN architecture (hub-and-spoke):**
```
┌─────────────────────────────────────┐
│         VPN Server (Hub)            │
│         (Central point)             │
└─────────────────────────────────────┘
       ↗         ↑         ↖
      /          |          \
Device A     Device B     Device C
(spoke)      (spoke)      (spoke)

Problem: All traffic goes through the hub (bottleneck)
```

### What is a Mesh Network?

A **mesh network** allows devices to connect directly to each other, without a central hub.

**Mesh VPN (like Tailscale):**
```
Device A ←──────→ Device B
    ↖              ↗
      ↘          ↙
       Device C

Each device can connect directly to any other device
No central bottleneck!
```

**Benefits:**
- **Faster**: Direct connections (no middle server)
- **More reliable**: No single point of failure
- **Scalable**: Adding devices doesn't slow down the network

**Tailscale is a mesh VPN** - this is what makes it special!

### What is Encryption?

**Encryption** scrambles data so only authorized parties can read it.

**Simple analogy:**
```
Original message: "Hello"
Encryption key:   "SECRET123"
Encrypted:        "X7$mK2@pL"

Without the key, it's gibberish.
With the key, you can decrypt: "Hello"
```

**Two types of encryption:**

#### Symmetric Encryption
**Same key** for encryption and decryption.

```
Alice                          Bob
  │                             │
  │  Shared Key: "SECRET123"    │
  │                             │
  ├─ Encrypt with "SECRET123" ──┤
  │  Send: "X7$mK2@pL"          │
  ├────────────────────────────>│
  │                             │
  │  Decrypt with "SECRET123" ──┤
  │  Read: "Hello"              │
```

**Problem**: How do Alice and Bob share the key securely?

#### Asymmetric Encryption (Public/Private Keys)
**Two keys**: Public key (share with everyone) and Private key (keep secret).

```
Bob's Keys:
├─ Public Key:  "BOB_PUBLIC"  (anyone can have this)
└─ Private Key: "BOB_SECRET"  (only Bob knows this)

Alice wants to send to Bob:
1. Alice encrypts with Bob's PUBLIC key
2. Only Bob's PRIVATE key can decrypt it
3. Even Alice can't decrypt it after encrypting!
```

**How it works:**
```
Alice                          Bob
  │                             │
  │  Gets Bob's public key      │
  │<────────────────────────────┤
  │                             │
  ├─ Encrypt with BOB_PUBLIC ───┤
  │  Send encrypted message     │
  ├────────────────────────────>│
  │                             │
  │  Decrypt with BOB_SECRET ───┤
  │  (only Bob can do this!)    │
```

**Real-world example:**
- **HTTPS**: Your browser uses the website's public key
- **SSH**: You use public/private key pairs
- **WireGuard/Tailscale**: Each device has a key pair

### What is End-to-End Encryption?

**End-to-end encryption** means only the sender and receiver can read the data - not even the service provider.

**Without end-to-end encryption:**
```
You → Email Provider (can read your email) → Recipient
```

**With end-to-end encryption:**
```
You → Email Provider (sees gibberish) → Recipient
      ↑
      Only you and recipient have the keys
```

**Tailscale uses end-to-end encryption:**
```
Your Device → Tailscale Relay (can't read data) → Other Device
              ↑
              Even if using a relay, data is encrypted
```

### What is NAT (Revisited for VPNs)?

NAT was covered in Part 2, but here's why it matters for VPNs:

**The problem:**
```
Your home network:
├─ Router: Public IP 203.0.113.50
└─ Your PC: Private IP 192.168.1.10

Friend's network:
├─ Router: Public IP 198.51.100.25
└─ Friend's PC: Private IP 192.168.1.15

How can your PC directly connect to friend's PC?
Both are behind NAT!
```

**Traditional VPN solution:**
```
Both connect to VPN server (public IP)
VPN server relays traffic between you
```

**Tailscale's solution:**
```
Uses NAT traversal techniques (STUN, DERP)
Tries to establish direct connection
Falls back to relay only if necessary
```

### What is CGNAT? (Carrier-Grade NAT)

**CGNAT** is when your **ISP** puts you behind their own NAT, in addition to your home router's NAT.

```
Without CGNAT (traditional setup):
You (192.168.1.x) → Your Router → Public IP (203.0.113.50) → Internet
                                   ↑
                                   This is YOUR IP

With CGNAT:
You (192.168.1.x) → Your Router → ISP's NAT → ISP's Public IP → Internet
                    ↑              ↑
                    Private IP     Another private IP!
                    (your home)    (ISP's network, often 100.64.x.x)
```

**The nightmare scenario:**
- You think you have a "public IP" but it's actually another private IP (often starting with `100.64.x.x`)
- You're behind **double NAT** - your router's NAT AND the ISP's NAT
- Port forwarding on your router does NOTHING because there's another NAT above it you don't control

**Why CGNAT exists:**
IPv4 has only ~4 billion addresses. With billions of devices, ISPs ran out. Instead of giving each customer a public IP, they share one public IP among many customers using NAT at the ISP level.

```
ISP's Public IP: 203.0.113.50

Customer A (behind 100.64.0.1) ─┐
Customer B (behind 100.64.0.2) ─┼── All share → 203.0.113.50
Customer C (behind 100.64.0.3) ─┘
```

**CGNAT = You MUST Use Relay Servers**

With CGNAT, **direct P2P connections become nearly impossible** because:

1. **You can't port forward** - The ISP's NAT ignores your router's settings
2. **STUN doesn't help much** - Even if you discover your "public" IP, it's behind another NAT you can't traverse
3. **Symmetric NAT** - CGNAT often uses symmetric NAT (the hardest type to penetrate)

> **Bottom Line**: If you're behind CGNAT, you'll almost always end up using DERP relays. Direct connections might occasionally work, but don't count on them.

### What is DNS (Domain Name System)?

**DNS** translates human-readable names to IP addresses.

**How it works:**
```
You type: google.com
         ↓
DNS Server: "google.com = 142.250.185.46"
         ↓
Your browser connects to: 142.250.185.46
```

**DNS hierarchy:**
```
You: "What's google.com?"
  ↓
Local DNS (ISP): "Let me check..."
  ↓
Root DNS: "Ask the .com server"
  ↓
.com DNS: "Ask Google's DNS"
  ↓
Google DNS: "It's 142.250.185.46"
```

**Tailscale's MagicDNS:**
- Runs a local DNS server (100.100.100.100)
- Automatically creates names for your devices
- `laptop.tailnet.ts.net` → `100.64.0.1`
- No manual configuration needed!

### What is a Daemon?

A **daemon** is a background program that runs continuously.

**Analogy:**
```
Regular program: You open it, use it, close it
Daemon: Starts when computer boots, runs in background
```

**Common daemons:**
- `sshd`: SSH server daemon
- `httpd`: Web server daemon
- `dockerd`: Docker daemon
- `tailscaled`: Tailscale daemon

**Tailscale daemon:**
```
┌─────────────────────────────────────┐
│         Your Computer               │
│                                     │
│  Applications (browser, etc.)       │
│         ↓                           │
│  tailscaled (daemon)                │
│  - Manages VPN connection           │
│  - Handles encryption               │
│  - Routes traffic                   │
│         ↓                           │
│  Network Interface                  │
└─────────────────────────────────────┘
```

**How to interact with daemons:**
```bash
# Start daemon
sudo systemctl start tailscaled

# Check status
sudo systemctl status tailscaled

# Stop daemon
sudo systemctl stop tailscaled

# Control daemon
tailscale up    # Tell daemon to connect
tailscale down  # Tell daemon to disconnect
```

### What is a Relay?

A **relay** is a server that forwards traffic between two parties who can't connect directly.

**Direct connection (ideal):**
```
Device A ←──────────────→ Device B
         (fast, direct)
```

**Relayed connection (fallback):**
```
Device A ───→ Relay ───→ Device B
         (slower, but works when direct fails)
```

**When relays are needed:**
- Both devices behind strict NAT
- Firewalls block direct connections
- Network policies prevent P2P

**Tailscale's DERP relays:**
- Distributed globally (low latency)
- End-to-end encrypted (relay can't see data)
- Automatic fallback (tries direct first)

### Peer-to-Peer (P2P) Connections

**Peer-to-peer** means devices connect directly without a middleman.

**Client-Server (traditional):**
```
Client A → Server ← Client B
           (all traffic through server)
```

**Peer-to-Peer:**
```
Peer A ←──────────→ Peer B
       (direct connection)
```

**Benefits of P2P:**
- Lower latency (no middle hop)
- Higher bandwidth (no server bottleneck)
- More private (no server sees traffic)

**Challenges:**
- NAT traversal (both behind routers)
- Firewall configuration
- Discovery (how do peers find each other?)

**Tailscale solves these challenges:**
- Coordination server helps peers find each other
- STUN helps with NAT traversal
- DERP relays as fallback

### What is a Firewall?

A **firewall** controls what network traffic is allowed in or out.

**Analogy:**
```
Firewall = Security guard at a building
- Checks everyone entering/leaving
- Follows a list of rules
- Blocks unauthorized access
```

**Types of rules:**
```
Allow: Incoming traffic on port 22 (SSH)
Block: All traffic from 192.168.1.100
Allow: Outgoing traffic to 8.8.8.8 (Google DNS)
```

**How firewalls affect VPNs:**
```
Without VPN:
Firewall blocks: Incoming connections to your PC

With VPN:
You initiate connection to VPN server (allowed)
VPN tunnel established
Now you can receive traffic through the tunnel
```

**Tailscale and firewalls:**
- Works through most firewalls (outbound connections)
- No port forwarding needed
- Uses standard ports (443, 41641)

---

## Part 1: What is Tailscale?

### The Simple Answer
> Tailscale is a **zero-config VPN** that creates a secure mesh network between your devices, no matter where they are.

### What Makes It Different?
Traditional VPNs:
```
Your Device → VPN Server → Other Device
              (bottleneck)
```

Tailscale (mesh network):
```
Your Device ←→ Other Device (direct connection!)
```

### Key Features
- **Zero configuration**: No port forwarding, no firewall rules
- **Peer-to-peer**: Direct connections when possible
- **NAT traversal**: Works behind routers and firewalls
- **WireGuard-based**: Fast, modern encryption

---

## Part 2: WireGuard Under the Hood

### What is WireGuard?
WireGuard is a **modern VPN protocol** that's:
- **Fast**: Runs in the Linux kernel (or userspace)
- **Secure**: Uses state-of-the-art cryptography
- **Simple**: ~4,000 lines of code (vs OpenVPN's 100,000+)

### How WireGuard Works
```
┌─────────────────────────────────────────┐
│         Your Device                     │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │  Application (e.g., browser)      │  │
│  └───────────────┬───────────────────┘  │
│                  │                      │
│                  ▼                      │
│  ┌───────────────────────────────────┐  │
│  │  WireGuard Interface (wg0)        │  │
│  │  - Encrypts packets               │  │
│  │  - Adds WireGuard header          │  │
│  └───────────────┬───────────────────┘  │
│                  │                      │
│                  ▼                      │
│  ┌───────────────────────────────────┐  │
│  │  Physical Network Interface       │  │
│  │  - Sends encrypted UDP packets    │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

### WireGuard Packet Structure
```
Original Packet:
┌────────────────────────────────┐
│ IP Header (10.0.0.1 → 10.0.0.2)│
│ TCP/UDP Header                 │
│ Application Data               │
└────────────────────────────────┘

After WireGuard Encryption:
┌────────────────────────────────┐
│ Outer IP (Public IPs)          │  ← Routable on Internet
│ UDP Header (Port 51820)        │  ← WireGuard default port
│ WireGuard Header               │
│ ┌────────────────────────────┐ │
│ │ Encrypted Original Packet  │ │
│ └────────────────────────────┘ │
└────────────────────────────────┘
```

---

## Part 3: Tailscale's Architecture

### The Tailnet
When you install Tailscale, your devices join a **tailnet** (Tailscale network).

```
Your Tailnet (example.ts.net):
┌─────────────────────────────────────────┐
│                                         │
│  Laptop (100.64.0.1)                    │
│     ↕                                   │
│  Phone (100.64.0.2)                     │
│     ↕                                   │
│  Server (100.64.0.3)                    │
│                                         │
└─────────────────────────────────────────┘
```

### The Control Plane
Tailscale uses a **coordination server** (control plane) to:
1. Authenticate devices
2. Distribute public keys
3. Coordinate NAT traversal
4. Manage ACLs (access control lists)

**Important**: The control plane does NOT route your traffic!

### The Data Plane
Actual traffic flows **peer-to-peer** when possible:
```
Device A ←──────────────────────→ Device B
         (direct encrypted tunnel)
```

---

## Part 4: NAT Traversal Magic

### The Problem
Most devices are behind NAT (Network Address Translation):
```
Your Device (192.168.1.100) → Router (Public IP) → Internet
```

How does another device reach you?

### Solution 1: STUN (Session Traversal Utilities for NAT)
STUN helps devices discover their **public IP and port**.

```
1. Your device asks STUN server: "What's my public IP?"
2. STUN responds: "You're 203.0.113.50:12345"
3. Tailscale shares this info with other devices
4. Other devices can now connect directly!
```

### Solution 2: DERP Relays
When direct connection fails (strict NAT, firewall), Tailscale uses **DERP relays**.

**DERP** = Designated Encrypted Relay for Packets

```
Device A → DERP Relay → Device B
           (encrypted end-to-end)
```

**Key Point**: Even when relayed, traffic is **encrypted end-to-end**. The relay can't see your data.

### STUN vs TURN vs DERP (What's the Difference?)

These are all NAT traversal techniques, but they serve different purposes:

**STUN (Session Traversal Utilities for NAT)**
- **Purpose:** Discover your public IP and port mapping
- **Does NOT relay traffic** - it just helps you discover your public endpoint
- If NAT is too strict, STUN alone isn't enough

**TURN (Traversal Using Relays around NAT)**
- **Purpose:** Actually relay traffic when direct connection fails
- **Expensive** - all traffic goes through it, requiring bandwidth
- Standard protocol used by WebRTC

**DERP (Tailscale's Relay System)**
- **Purpose:** Tailscale's custom relay solution (similar to TURN)
- Your traffic is **already encrypted** with WireGuard before reaching DERP
- DERP relays just forward opaque encrypted blobs - they can't see your data
- Tailscale runs DERP servers globally, so you get low latency

```
┌─────────────────────────────────────────────────────────────────┐
│ Feature          │ TURN               │ DERP                    │
├──────────────────┼────────────────────┼─────────────────────────┤
│ Protocol         │ Standard (WebRTC)  │ Tailscale proprietary   │
│ Encryption       │ Handled separately │ Already WireGuard       │
│ Fallback         │ WebRTC decides     │ Automatic by Tailscale  │
│ Global network   │ You set up servers │ Tailscale runs them     │
└─────────────────────────────────────────────────────────────────┘
```

### The Decision Tree
```
Can devices connect directly?
├─ Yes → Use direct P2P connection (fastest!)
└─ No  → Use DERP relay (still secure, slightly slower)
```

### How a Connection is Made (Step by Step)

```
You want to connect to peer "server" (100.64.0.2):

1. CHECK CACHE
   Do we already have a working path to 100.64.0.2?
   ├─ Yes → Use it
   └─ No  → Discovery phase

2. DISCOVERY (if needed)
   a. Ask coordination server for peer's known endpoints
   b. Ask STUN for your own public endpoint
   c. Try connecting directly to each known endpoint
   d. Peer does the same simultaneously ("hole punching")

3. ESTABLISH TUNNEL
   ├─ Direct connection works → Use P2P WireGuard tunnel
   └─ Direct fails → Fall back to DERP relay

4. COMMUNICATE
   All traffic encrypted with WireGuard (peer's public key)
```

**The Actual Traffic Flow:**
```
You send data to 100.64.0.2:

Your App → Tailscale interface → WireGuard encrypts → 
    → Internet (direct or via DERP) →
        → Peer's WireGuard decrypts → Peer's App
```

---

## Part 5: The `/32` Subnet Mystery

### What is `/32`?
In CIDR notation:
- `/24` = 256 addresses (e.g., `192.168.1.0/24` → `192.168.1.0` to `192.168.1.255`)
- `/32` = **1 address** (e.g., `100.64.0.1/32` → only `100.64.0.1`)

### Why Tailscale Uses `/32`
Tailscale assigns each device a **point-to-point** interface:
```bash
$ ip addr show tailscale0
tailscale0: <POINTOPOINT,MULTICAST,NOARP,UP>
    inet 100.64.0.1/32 scope global tailscale0
```

**Point-to-point** means:
- No broadcast
- No ARP (Address Resolution Protocol)
- Each peer is a separate route

### The Routing Table
```bash
$ ip route
100.64.0.2 dev tailscale0  # Route to peer 1
100.64.0.3 dev tailscale0  # Route to peer 2
100.64.0.4 dev tailscale0  # Route to peer 3
```

Each peer gets its own route, not a subnet.

### Why This Matters for Docker Swarm
Docker Swarm expects to **bind** to an IP that exists on a local interface.

```
Problem:
- Tailscale IP: 100.64.0.1/32 (point-to-point)
- Docker tries: bind(100.64.0.1)
- Some systems reject binding to /32 interfaces
```

**This is one reason for the "cannot assign requested address" error.**

### Won't Tailscale Run Out of IP Addresses?

> "Tailscale uses 100.64.0.0/10 (about 4 million IPs). With /32, each device gets exactly 1 IP. Won't they run out?"

**Why It's Not a Problem:**

```
100.64.0.0/10 = 4,194,304 addresses (about 4 million)

But wait - this is PER TAILNET!
```

**Each tailnet (organization/account) has its own virtual address space:**
```
Alice's tailnet:
- laptop: 100.64.0.1
- phone: 100.64.0.2

Bob's tailnet:
- laptop: 100.64.0.1  ← SAME IP, different tailnet!
- server: 100.64.0.2

Devices in different tailnets can't see each other by default.
IPs are only unique WITHIN a tailnet.
```

**4 million per org is plenty:**
- Most organizations have < 1,000 devices
- Even enterprise companies rarely exceed 100,000
- Tailscale could support 40,000+ companies each with 100 devices

### Why /32 Specifically? (The Routing Perspective)

The `/32` is about **routing**, not address allocation:

```
Traditional subnet (e.g., /24):
- "All 256 IPs in this range are on this local network"
- Router sends ARP: "Who has 192.168.1.5?"
- Machine responds: "That's me!"
- Broadcast/multicast works within subnet

/32 (point-to-point):
- "This SINGLE IP is reachable via this interface"
- No ARP, no broadcast
- Each peer is individually tracked
```

**Why Tailscale chose /32:**
1. **No broadcast storms** - Tailnets can be huge without broadcast overhead
2. **Precise routing** - Each peer is a separate route entry
3. **Security** - Devices can't see each other's traffic (no broadcast snooping)
4. **Scalability** - Works the same with 3 devices or 3,000

### Point-to-Point Interfaces Deep Dive

A **point-to-point** interface connects exactly two endpoints with no intermediate nodes.

```
Regular Ethernet (broadcast capable):
┌─────────────────────────────────────────┐
│     Local Network (192.168.1.0/24)      │
│                                         │
│  PC─┬─Laptop─┬─Phone─┬─Server─┬─Router  │
│     │        │       │        │         │
│     └────────┴───────┴────────┘         │
│     (all devices can "hear" each other) │
└─────────────────────────────────────────┘

Point-to-Point (Tailscale):
┌──────────────┐          ┌──────────────┐
│   Device A   │──tunnel──│   Device B   │
└──────────────┘          └──────────────┘
      ↑                          ↑
      Each tunnel is independent
      No shared medium
```

**Interface Flags Explained:**

When you run `ip addr show tailscale0`, you see:
```
tailscale0: <POINTOPOINT,MULTICAST,NOARP,UP>
            ↑           ↑        ↑
            │           │        └─ No ARP (Address Resolution Protocol)
            │           └─ Can do multicast (but usually doesn't)
            └─ Point-to-point interface
```

**POINTOPOINT flag means:**
- No ARP needed (we know exactly who's on the other end)
- No subnet assumptions (each peer is explicitly routed)
- No broadcast (can't yell at everyone on the network)

**NOARP means:**
- On Ethernet, you'd use ARP: "Who has 192.168.1.5?"
- On point-to-point, you already know: "100.64.0.5 is reachable via this tunnel"

---

## Part 6: Userspace vs Kernel Networking

### Kernel Networking (Traditional)
```
Application
    ↓
Kernel Network Stack
    ↓
Network Interface (eth0)
    ↓
Physical Hardware
```

**Fast** but requires kernel modules.

### Userspace Networking (Tailscale on some platforms)
```
Application
    ↓
Tailscale Daemon (userspace)
    ↓
TUN/TAP Device
    ↓
Kernel Network Stack
    ↓
Physical Hardware
```

**Slower** but works without kernel modules.

### Tailscale on WSL2
On WSL2, Tailscale uses **userspace networking**:
- Creates a TUN device (`tailscale0`)
- Tailscale daemon handles packet routing
- Slightly higher overhead

**This can cause issues** with software that expects kernel-level networking.

### Detailed Kernel vs Userspace Comparison

**Kernel networking:**
```
┌─────────────────────────────────────────┐
│ Application                             │
├─────────────────────────────────────────┤
│                                         │
│     KERNEL SPACE (privileged mode)      │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │ TCP/IP Stack                      │  │
│  │ Network Drivers                   │  │
│  │ WireGuard (if kernel module)      │  │
│  └───────────────────────────────────┘  │
│                   ↓                     │
│  Physical NIC (eth0, wlan0)             │
└─────────────────────────────────────────┘

Everything happens in kernel = FAST
```

**Userspace networking:**
```
┌─────────────────────────────────────────┐
│ Application                             │
├─────────────────────────────────────────┤
│                                         │
│     USER SPACE (normal programs)        │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │ tailscaled (WireGuard impl)       │  │
│  │ - Reads from TUN device           │  │
│  │ - Encrypts/decrypts               │  │
│  │ - Implements WireGuard protocol   │  │
│  └──────────────┬────────────────────┘  │
│                 │ (context switch!)     │
├─────────────────┼───────────────────────┤
│     KERNEL      │                       │
│  ┌──────────────┴────────────────────┐  │
│  │ TUN device (dumb pipe)            │  │
│  │ TCP/IP Stack                      │  │
│  │ Physical NIC                      │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘

Packets bounce between user and kernel = slower
```

### Why Userspace Networking Exists

**Sometimes you can't use kernel modules:**

| Situation | Why Kernel Module Fails | Userspace Solution |
|-----------|------------------------|-------------------|
| WSL2 | Microsoft's kernel doesn't include WireGuard | tailscaled implements WireGuard |
| Containers | Can't load modules in containers | Run WireGuard in the container |
| Restricted OS | macOS sandbox restrictions | Use userspace implementation |
| Old kernels | WireGuard wasn't in kernel before 5.6 | wireguard-go (userspace) |

### Performance Impact

```
Kernel WireGuard:
- < 1ms per packet overhead
- ~3-5 Gbps throughput on good hardware
- CPU efficient (no context switches)

Userspace WireGuard (tailscaled):
- ~0.5-2ms additional latency
- ~500-1500 Mbps typical throughput
- More CPU usage (context switches between user/kernel)
```

**For most uses, you won't notice** - the internet is usually the bottleneck, not local processing.

### Tailscale Platform Modes

| Platform | Mode | Why |
|----------|------|-----|
| Linux (native) | Kernel WireGuard | Best performance |
| Linux (old kernel) | Userspace | WireGuard module not available |
| WSL2 | Userspace | Microsoft kernel restrictions |
| macOS | Userspace | Sandboxing requirements |
| Windows | Userspace | No kernel module support |
| iOS/Android | Userspace | Mobile OS restrictions |

---

## Part 7: MagicDNS

### What is MagicDNS?
Tailscale automatically assigns DNS names to your devices:
```
laptop.example.ts.net    → 100.64.0.1
phone.example.ts.net     → 100.64.0.2
server.example.ts.net    → 100.64.0.3
```

### How It Works
Tailscale runs a **local DNS server** (100.100.100.100):
```bash
$ nslookup server.example.ts.net
Server:  100.100.100.100
Address: 100.100.100.100#53

Name:    server.example.ts.net
Address: 100.64.0.3
```

### Why It's Useful
Instead of remembering IPs:
```bash
# Without MagicDNS
ssh 100.64.0.3

# With MagicDNS
ssh server.example.ts.net
```

---

## Part 8: Tailscale on Different Platforms

### Linux (Native)
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Creates `tailscale0` interface in the **kernel**.

### Windows (Native)
Tailscale runs as a Windows service.
Creates a virtual network adapter.

### WSL2 (Inside Linux)
```bash
# Inside Ubuntu WSL2
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Creates `tailscale0` inside the **WSL2 namespace** (separate from Windows).

### Docker Container
```bash
docker run -d \
  --name=tailscale \
  --cap-add=NET_ADMIN \
  --device=/dev/net/tun \
  tailscale/tailscale
```

Tailscale runs **inside the container's namespace**.

---

## Part 9: Why Your Docker Swarm Setup Failed

### The Setup
```
Windows Host:
- Tailscale installed (100.66.197.8)

Docker Desktop:
- Runs in docker-desktop WSL2 VM
- Separate network namespace
- No Tailscale interface
```

### The Command
```bash
docker swarm init --advertise-addr 100.66.197.8
```

### What Happens
```
1. Docker Engine (in docker-desktop namespace) tries to bind to 100.66.197.8
2. Checks local interfaces:
   - lo (127.0.0.1) ✓
   - eth0 (172.X.X.Y) ✓
   - tailscale0 (100.66.197.8) ✗ (doesn't exist in this namespace!)
3. Error: "cannot assign requested address"
```

### The Fix
Put Tailscale in the **same namespace** as Docker:

**Option A**: Install Tailscale in Ubuntu WSL2 (where you'll also install Docker)
```bash
# In Ubuntu WSL2
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up

# Install Docker Engine (not Desktop)
curl -fsSL https://get.docker.com | sh

# Now both are in the same namespace!
docker swarm init --advertise-addr $(tailscale ip -4)
```

**Option B**: Use Windows 11 mirrored networking (experimental)

---

## Part 10: Hands-On Exploration

### Check Your Tailscale IP
```bash
tailscale ip -4  # IPv4 address
tailscale ip -6  # IPv6 address
```

### See Tailscale Interface
```bash
ip addr show tailscale0
```

Look for:
```
inet 100.64.0.1/32  ← Notice the /32!
```

### Check Routing Table
```bash
ip route | grep tailscale
```

You'll see individual routes for each peer.

### Test Connectivity
```bash
# Ping another device on your tailnet
ping 100.64.0.2

# Check which path is used (direct or DERP)
tailscale status
```

Look for:
```
100.64.0.2   server   direct  ← Direct connection!
100.64.0.3   phone    relay   ← Using DERP relay
```

### See Active Connections
```bash
tailscale netcheck
```

Shows:
- Your public IP
- Available DERP relays
- NAT type

---

## Part 11: Common Tailscale Commands

### Basic Operations
```bash
# Start Tailscale
sudo tailscale up

# Stop Tailscale
sudo tailscale down

# Check status
tailscale status

# Get your IP
tailscale ip
```

### Advanced
```bash
# Enable subnet routing (share local network)
sudo tailscale up --advertise-routes=192.168.1.0/24

# Accept routes from other devices
sudo tailscale up --accept-routes

# Use specific DERP region
sudo tailscale up --advertise-exit-node

# SSH to another tailnet device
ssh user@device-name.tailnet-name.ts.net
```

---

## Part 12: Security Considerations

### What Tailscale Encrypts
- ✅ All traffic between devices (WireGuard encryption)
- ✅ Even when using DERP relays

### What Tailscale Knows
- ✅ Your device list
- ✅ Your public keys
- ✅ Connection metadata (who connects to whom)
- ❌ Your actual traffic (end-to-end encrypted)

### Self-Hosting (Headscale)
Don't trust Tailscale's control plane? Use **Headscale**:
```bash
# Open-source Tailscale control server
# You run it yourself
```

### How Encryption Works at Tailscale's Scale

**The Key Insight: Decentralized Encryption**

Tailscale doesn't encrypt your traffic - **WireGuard on each device does**. This is what makes it scalable.

```
Traditional centralized encryption:
Device A → Central Server (encrypts/decrypts) → Device B
           ↑
           This server must handle ALL traffic = bottleneck

Tailscale's approach:
Device A ←────── encrypted ──────→ Device B
    ↑                                    ↑
    Each device encrypts/decrypts        locally
```

**Key Distribution:**

The coordination server's job is just **distributing public keys**, not encrypting:

```
┌─────────────────────────────────────────────────┐
│ Tailscale Coordination Server                   │
│                                                 │
│ Device A: Public Key = ABC123...                │
│ Device B: Public Key = XYZ789...                │
│ Device C: Public Key = DEF456...                │
│                                                 │
│ "Here's everyone's public keys. Now talk        │
│  directly to each other, encrypted."            │
└─────────────────────────────────────────────────┘
```

**Why this scales infinitely:**
1. **No central bottleneck** - Traffic goes peer-to-peer
2. **No key escrow** - Tailscale never sees private keys
3. **Coordination is lightweight** - Just metadata, not traffic

### WireGuard Cryptography (The Actual Primitives)

WireGuard uses **Noise Protocol Framework** with these primitives:

```
Curve25519      → Key exchange (ECDH)
ChaCha20        → Symmetric encryption (fast on CPUs without AES hardware)
Poly1305        → Message authentication (detect tampering)
BLAKE2s         → Hashing
```

**Each peer-to-peer tunnel has unique keys:**
```
You + Peer A → Unique session key
You + Peer B → Different unique session key
You + Peer C → Yet another unique session key

If one tunnel is compromised, others are safe.
```

### Perfect Forward Secrecy

WireGuard renegotiates keys every 2 minutes (or 2GB of data):

```
Minute 0:   Key_v1 in use
Minute 2:   Key_v2 generated, Key_v1 discarded
Minute 4:   Key_v3 generated, Key_v2 discarded

If attacker captures encrypted traffic AND steals Key_v3:
- They can only decrypt traffic from the last 2 minutes
- All previous traffic remains secure
```

---

## Key Takeaways

1. **Tailscale = Zero-config mesh VPN** using WireGuard
2. **NAT traversal** via STUN and DERP relays
3. **`/32` subnet** = point-to-point interface (one IP per peer)
4. **Userspace networking** on some platforms (like WSL2)
5. **MagicDNS** provides automatic DNS names
6. **Namespace isolation** is why Docker Desktop can't see Windows Tailscale

---

## Further Reading
- [How Tailscale Works](https://tailscale.com/blog/how-tailscale-works/)
- [WireGuard Whitepaper](https://www.wireguard.com/papers/wireguard.pdf)
- [Tailscale on WSL2](https://tailscale.com/kb/1167/wsl/)
- [NAT Traversal Explained](https://tailscale.com/blog/how-nat-traversal-works/)
