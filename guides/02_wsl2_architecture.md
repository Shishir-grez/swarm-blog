# WSL2 Architecture: Complete Guide

## What You'll Learn
- What WSL2 actually is (spoiler: it's a VM!)
- How it differs from WSL1
- Why WSL2 has network isolation
- How this affects Docker Desktop and Tailscale

**Prerequisites**: None!

---

## Foundational Concepts

Before diving into WSL2, let's understand some key terms. If you're already familiar with virtualization, feel free to skip to [Part 1](#part-1-what-is-wsl2).

### What is a Virtual Machine (VM)?

A **virtual machine** is a software-based computer running inside your physical computer.



**How it works:**
```
┌─────────────────────────────────────┐
│      Your Physical Computer         │
│                                     │
│  ┌──────────┐  ┌──────────┐       │
│  │   VM 1   │  │   VM 2   │       │
│  │ (Ubuntu) │  │ (Windows)│       │
│  │          │  │          │       │
│  └──────────┘  └──────────┘       │
│                                     │
│         Hypervisor                  │
│  ────────────────────────────      │
│         Hardware (CPU, RAM)         │
└─────────────────────────────────────┘
```

**Why use VMs?**
- Run multiple operating systems on one computer
- Isolate applications (if one VM crashes, others are fine)
- Test software in different environments
- Security (malware in VM can't easily escape)

### What is a Hypervisor?

A **hypervisor** is software that creates and manages virtual machines.

**Two types:**

**Type 1 Hypervisor** (Bare Metal):
```
┌─────────────────────────────────────┐
│  VM 1  │  VM 2  │  VM 3             │
├─────────────────────────────────────┤
│      Hypervisor (VMware ESXi)       │
├─────────────────────────────────────┤
│      Hardware (CPU, RAM, Disk)      │
└─────────────────────────────────────┘
```
- Runs directly on hardware
- No host operating system
- Better performance
- Examples: VMware ESXi, Hyper-V, KVM

**Type 2 Hypervisor** (Hosted):
```
┌─────────────────────────────────────┐
│  VM 1  │  VM 2                      │
├─────────────────────────────────────┤
│  Hypervisor (VirtualBox)            │
├─────────────────────────────────────┤
│  Host OS (Windows/macOS/Linux)      │
├─────────────────────────────────────┤
│  Hardware                           │
└─────────────────────────────────────┘
```
- Runs as an application on a host OS
- Easier to use
- Slightly slower
- Examples: VirtualBox, VMware Workstation

**WSL2 uses Hyper-V** (Type 1 hypervisor built into Windows).

### What is NAT (Network Address Translation)?

**NAT** is a technique that lets multiple devices share one public IP address.

**Your home network example:**
```
Inside your home:
├─ Laptop:     192.168.1.10
├─ Phone:      192.168.1.11
└─ Desktop:    192.168.1.12

Your router:
├─ Internal IP: 192.168.1.1 (gateway)
└─ External IP: 203.0.113.50 (public internet)

When laptop visits google.com:
1. Laptop sends: 192.168.1.10 → google.com
2. Router translates: 203.0.113.50 → google.com
3. Google responds to: 203.0.113.50
4. Router translates back: → 192.168.1.10
```

**Why NAT exists:**
- Not enough public IPv4 addresses for every device
- Security (hides internal network structure)
- Allows private IP reuse (every home can use 192.168.1.x)

**In WSL2 context:**
- Windows acts as the NAT router
- WSL2 VM gets a private IP (172.x.x.x)
- Windows translates traffic to/from the internet

### What is a Gateway?

A **gateway** is the "door" between your network and another network.

**Simple explanation:**
```
Your network:     192.168.1.0/24
├─ Your computer: 192.168.1.10
├─ Printer:       192.168.1.20
└─ Gateway:       192.168.1.1 ← This is your router

Internet:         Everything else

To reach the internet:
Your computer → Gateway (192.168.1.1) → Internet
```

**How your computer knows to use the gateway:**
```bash
# Check your routing table
ip route

# You'll see something like:
default via 192.168.1.1  ← This is the gateway
```

**In WSL2:**
- Windows acts as the gateway for WSL2
- WSL2 sends internet traffic to Windows
- Windows forwards it to your actual router

### What is a Syscall (System Call)?

A **syscall** is how programs ask the operating system to do something.

**Analogy:**
```
Program = Customer at a restaurant
Syscall = Placing an order
OS Kernel = Kitchen

Customer can't go into the kitchen directly.
They must place an order (syscall).
Kitchen prepares it and returns the result.
```

**Common syscalls:**
```c
open()    // Open a file
read()    // Read from file
write()   // Write to file
socket()  // Create network connection
fork()    // Create new process
```

**Why this matters for WSL:**
- Linux programs make Linux syscalls
- Windows kernel doesn't understand Linux syscalls
- WSL1: Translated Linux syscalls to Windows syscalls (slow, incomplete)
- WSL2: Runs real Linux kernel (native Linux syscalls, fast!)

### What is a Distro (Distribution)?

A **distro** is a packaged version of Linux with specific software and configurations.

**Linux = Kernel + Software:**
```
Linux Kernel (core)
    +
Software (shell, package manager, desktop, etc.)
    =
Linux Distribution
```

**Popular distros:**
- **Ubuntu**: Beginner-friendly, lots of software
- **Debian**: Stable, conservative updates
- **Fedora**: Cutting-edge features
- **Arch**: Minimal, DIY approach

**In WSL2:**
- You can install multiple distros simultaneously
- Each distro is a separate environment
- Examples: Ubuntu, Debian, Kali Linux, Alpine

**Why multiple distros?**
```
WSL2 on your PC:
├─ Ubuntu (for web development)
├─ Kali Linux (for security testing)
└─ Alpine (for lightweight containers)
```

Each runs independently with its own files and software.

### What is the 9P Protocol?

**9P** is a network protocol for sharing files.

**In WSL2 context:**
- Windows files are stored on the Windows filesystem (NTFS)
- WSL2 needs to access these files
- 9P protocol bridges the gap

**How it works:**
```
WSL2 (Linux)                Windows
    │                          │
    │  "Give me file.txt"      │
    ├──────────────────────────>│
    │                          │
    │  (9P protocol)           │
    │                          │
    │<──────────────────────────┤
    │  "Here's the file"       │
```

**Practical example:**
```bash
# In WSL2, access Windows files:
cd /mnt/c/Users/YourName/Documents
# This uses 9P to read Windows files
```

**Trade-off:**
- Convenient (access Windows files from Linux)
- Slower than native Linux filesystem
- Best practice: Keep Linux files in Linux filesystem (`/home/`)

### What is a TUN/TAP Device?

**TUN/TAP** devices are virtual network interfaces used by VPNs and virtualization.

**TUN (Layer 3 - IP packets):**
```
Application → TUN device → VPN software → Encrypted → Internet
```

**TAP (Layer 2 - Ethernet frames):**
```
Application → TAP device → Bridge → Virtual network
```

**In WSL2/Tailscale context:**
- Tailscale creates a TUN device (`tailscale0`)
- Your applications send packets to this virtual interface
- Tailscale encrypts and routes them through the VPN

**Why "virtual"?**
- No physical hardware
- Created by software
- Acts like a real network card to applications

---

## Part 1: What is WSL2?

### The Simple Answer
> WSL2 = A **real Linux kernel** running in a **lightweight virtual machine** on Windows.

### Why Does It Exist?
Microsoft wanted to let Windows users run Linux programs without:
- Dual-booting
- Installing a full VM (like VirtualBox)
- Using slow compatibility layers

---

## Part 2: WSL1 vs WSL2

### WSL1 (The Translation Layer)
```
┌─────────────────────────────┐
│       Windows Kernel        │
│                             │
│  ┌───────────────────────┐  │
│  │  Translation Layer    │  │ ← Converts Linux syscalls
│  │  (NT Kernel)          │  │    to Windows syscalls
│  └───────────────────────┘  │
│           ▲                 │
│           │                 │
│  ┌────────┴──────────────┐  │
│  │  Linux Program        │  │
│  │  (thinks it's Linux)  │  │
│  └───────────────────────┘  │
└─────────────────────────────┘
```

**Problems with WSL1**:
- Not all Linux syscalls could be translated
- Docker didn't work properly
- File I/O was slow

### WSL2 (The Real VM)
```
┌──────────────────────────────────────┐
│         Windows 10/11                │
│                                      │
│  ┌────────────────────────────────┐  │
│  │    Hyper-V (Type 1 Hypervisor)│  │
│  │                                │  │
│  │  ┌──────────┐  ┌────────────┐ │  │
│  │  │ Windows  │  │ WSL2 VM    │ │  │
│  │  │ Kernel   │  │            │ │  │
│  │  │          │  │ Real Linux │ │  │
│  │  │          │  │ Kernel     │ │  │
│  │  └──────────┘  └────────────┘ │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

**Advantages**:
- ✅ Full Linux kernel (100% compatibility)
- ✅ Docker works natively
- ✅ Fast file I/O (inside the VM)

**Trade-offs**:
- ⚠️ Network isolation (separate IP space)
- ⚠️ Slightly higher memory usage

### Networking Architecture: Deep Dive

Understanding how WSL2 handles networking is critical for diagnosing connectivity issues.

#### Virtual Switch Architecture

WSL2 introduces a complete virtualized network layer:

```
┌──────────────────────────────────────────────────────────────────┐
│                        Windows Host                               │
│                                                                   │
│  Physical/WiFi Adapter                   Tailscale Adapter        │
│  ┌──────────────┐                        ┌──────────────┐         │
│  │ 192.168.1.100│                        │ 100.66.197.8 │         │
│  └──────┬───────┘                        └──────┬───────┘         │
│         │                                       │                 │
│         └───────────────┬───────────────────────┘                 │
│                         ▼                                         │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │                 Windows TCP/IP Stack                        │   │
│  │              (with routing + NAT rules)                     │   │
│  └────────────────────────────┬───────────────────────────────┘   │
│                               │                                   │
│                               │ Internal vSwitch                  │
│                               ▼                                   │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │              vEthernet (WSL) - Virtual Switch               │   │
│  │                   172.28.224.1/20                           │   │
│  │                                                             │   │
│  │  Functions as:                                              │   │
│  │  • DHCP Server (assigns IPs to WSL2 VM)                    │   │
│  │  • Default Gateway (routes traffic)                         │   │
│  │  • NAT Device (translates private ↔ public)                │   │
│  └────────────────────────────┬───────────────────────────────┘   │
│                               │                                   │
│  ┌────────────────────────────┴───────────────────────────────┐   │
│  │                        WSL2 VM                              │   │
│  │                                                             │   │
│  │    eth0: 172.28.230.45/20                                   │   │
│  │    Gateway: 172.28.224.1                                    │   │
│  │    DNS: 172.28.224.1 (proxied to Windows DNS)              │   │
│  │                                                             │   │
│  │  ┌──────────────┐    ┌──────────────┐                       │   │
│  │  │   Ubuntu     │    │    Debian    │                       │   │
│  │  │  (distro 1)  │    │  (distro 2)  │                       │   │
│  │  └──────────────┘    └──────────────┘                       │   │
│  └─────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────┘
```

#### The vEthernet Adapter Explained

When WSL2 starts, Windows creates a **Hyper-V Virtual Ethernet Adapter**:

```powershell
# View the WSL virtual adapter
Get-NetAdapter | Where-Object {$_.Name -like "*WSL*"}

# Output:
# Name           InterfaceDescription                    Status
# ----           --------------------                    ------
# vEthernet (WSL) Hyper-V Virtual Ethernet Adapter       Up
```

**What this adapter does:**
1. **Acts as the NAT gateway** for all WSL2 traffic
2. **Runs a DHCP-like service** to assign IPs to the WSL2 VM
3. **Handles DNS proxying** (forwards DNS queries to Windows resolver)

#### IP Assignment Process

Every time WSL2 boots, it gets a **new dynamic IP**:

```bash
# Inside WSL2 - check your current IP
ip addr show eth0

# Typical output:
# inet 172.28.230.45/20 brd 172.28.239.255 scope global eth0

# Check your gateway (Windows host)
ip route show default
# default via 172.28.224.1 dev eth0
```

**Why does the IP change?**
- The `172.x.x.x/20` range is dynamically allocated
- IP is assigned via internal DHCP at each WSL boot
- **This causes issues for services expecting stable IPs**

#### Outbound Traffic Flow (WSL2 → Internet)

```
Step-by-step: WSL2 app connects to google.com:443

┌───────────────────────────────────────────────────────────────┐
│ 1. App creates socket and connects                            │
│    WSL2 App → socket() → connect(google.com:443)             │
└───────────────────────────┬───────────────────────────────────┘
                            ▼
┌───────────────────────────────────────────────────────────────┐
│ 2. Linux kernel routes through eth0                           │
│    src: 172.28.230.45:54321 → dst: 142.250.80.14:443         │
└───────────────────────────┬───────────────────────────────────┘
                            ▼
┌───────────────────────────────────────────────────────────────┐
│ 3. Packet hits vEthernet (WSL) - NAT translation              │
│    src: 192.168.1.100:54321 → dst: 142.250.80.14:443         │
│         ↑                                                     │
│    (Windows IP replaces WSL2 private IP)                      │
└───────────────────────────┬───────────────────────────────────┘
                            ▼
┌───────────────────────────────────────────────────────────────┐
│ 4. Windows routes to physical adapter → Internet              │
│    Physical NIC → Router → ISP → Google                       │
└───────────────────────────────────────────────────────────────┘
```

#### Inbound Traffic Flow (Internet → WSL2)

This is where problems arise:

```
Scenario: Remote machine tries to connect to WSL2 service on port 8080

┌───────────────────────────────────────────────────────────────┐
│ 1. Remote machine sends packet                                 │
│    Remote: 203.0.113.50:12345 → Your IP: 192.168.1.100:8080   │
└───────────────────────────┬───────────────────────────────────┘
                            ▼
┌───────────────────────────────────────────────────────────────┐
│ 2. Packet arrives at Windows                                   │
│    Windows sees: "Is anything listening on port 8080?"        │
│                                                                │
│    ❌ If no Windows app is listening → Packet dropped         │
│    ✅ If portproxy rule exists → Forward to WSL2              │
└───────────────────────────┬───────────────────────────────────┘
                            ▼
┌───────────────────────────────────────────────────────────────┐
│ 3. Port forwarding required!                                   │
│                                                                │
│    # Manual port forward command:                              │
│    netsh interface portproxy add v4tov4 \                     │
│        listenaddress=0.0.0.0 listenport=8080 \                │
│        connectaddress=172.28.230.45 connectport=8080          │
└───────────────────────────────────────────────────────────────┘
```

**The portproxy problem:**
- WSL2's IP changes on every boot
- You must update the `connectaddress` each time
- This is why people use scripts to automate port forwarding

#### localhost Forwarding (Automatic)

Windows does provide **automatic localhost forwarding**:

```bash
# Start a server in WSL2
python3 -m http.server 8000
```

```powershell
# Access from Windows - works automatically!
curl http://localhost:8000
```

**How it works:**
- Windows intercepts `localhost:8000` connections
- Automatically forwards to WSL2's corresponding port
- **Only works for localhost, not external IPs**

#### DNS Resolution Chain

```
┌─────────────────────────────────────────────────────────────┐
│ WSL2 DNS Resolution Flow                                     │
│                                                              │
│  1. App calls: getaddrinfo("example.com")                   │
│                      ▼                                       │
│  2. /etc/resolv.conf points to: nameserver 172.28.224.1     │
│                      ▼                                       │
│  3. DNS query sent to Windows (vEthernet gateway)           │
│                      ▼                                       │
│  4. Windows DNS Client resolves (using system DNS settings) │
│                      ▼                                       │
│  5. Response returned to WSL2 app                           │
└─────────────────────────────────────────────────────────────┘
```

**Common DNS issues:**
```bash
# Check WSL2's DNS configuration
cat /etc/resolv.conf

# If auto-generated, you'll see:
# nameserver 172.28.224.1  ← Points to Windows

# If DNS isn't working:
# 1. Check if Windows DNS is working
# 2. Check if vEthernet adapter is up
# 3. Try: wsl --shutdown && wsl
```


---

## Part 3: The Hyper-V Magic

### What is Hyper-V?
Hyper-V is Microsoft's **Type 1 hypervisor** (like VMware ESXi or KVM).

**Type 1 Hypervisor** = Runs directly on hardware, not inside an OS.

### How WSL2 Uses It
WSL2 uses a special "lightweight" Hyper-V VM:
- Boots in ~1 second
- Dynamic memory (only uses what it needs)
- Shared filesystem with Windows (via 9P protocol)

---

## Part 4: WSL2 Networking Architecture

### The Default Setup (NAT Mode)

```
┌────────────────────────────────────────────────┐
│                Windows Host                    │
│                                                │
│  Physical NIC: 192.168.1.100                   │
│  Tailscale:    100.66.197.8                    │
│                                                │
│  ┌──────────────────────────────────────────┐  │
│  │         vEthernet (WSL)                  │  │
│  │         172.X.X.1 (NAT gateway)          │  │
│  └──────────────────────────────────────────┘  │
│                     │                          │
│                     │ NAT                      │
│                     ▼                          │
│  ┌──────────────────────────────────────────┐  │
│  │         WSL2 VM                          │  │
│  │                                          │  │
│  │  eth0: 172.X.X.Y (dynamic IP)            │  │
│  │                                          │  │
│  │  ┌────────────┐    ┌────────────┐       │  │
│  │  │ Ubuntu     │    │ Debian     │       │  │
│  │  │ distro     │    │ distro     │       │  │
│  │  └────────────┘    └────────────┘       │  │
│  └──────────────────────────────────────────┘  │
└────────────────────────────────────────────────┘
```

### Key Points
1. **WSL2 gets a private IP** (like `172.X.X.Y`)
2. **Windows acts as NAT gateway** (like a home router)
3. **Outbound traffic works** (WSL2 → Internet)
4. **Inbound traffic requires port forwarding**

### The Isolation Problem
```
Windows:  Tailscale interface (100.66.197.8)
          ↕ (different network namespace)
WSL2:     No Tailscale interface
          Only sees: eth0 (172.X.X.Y)
```

**Result**: Programs in WSL2 can't bind to `100.66.197.8` because they can't "see" it.

---

## Part 5: Mirrored Networking Mode (Windows 11)

### The New Option
Windows 11 introduced **mirrored networking mode** to solve isolation issues.

### How to Enable
Edit `C:\Users\<YourName>\.wslconfig`:
```ini
[wsl2]
networkingMode=mirrored
```

Then restart WSL:
```powershell
wsl --shutdown
```

### What Changes
```
Before (NAT):
Windows: eth0 (192.168.1.100), tailscale0 (100.66.197.8)
WSL2:    eth0 (172.X.X.Y)  ← Different IPs

After (Mirrored):
Windows: eth0 (192.168.1.100), tailscale0 (100.66.197.8)
WSL2:    eth0 (192.168.1.100), tailscale0 (100.66.197.8)  ← Same IPs!
```

**Now** WSL2 can bind to the Tailscale IP!

### Limitations
- Windows 11 only
- Still experimental
- Docker Desktop doesn't fully leverage it yet

---

## Part 6: How Docker Desktop Fits In

### Docker Desktop's Architecture
```
┌────────────────────────────────────────────────┐
│                Windows Host                    │
│                                                │
│  ┌──────────────────────────────────────────┐  │
│  │    Docker Desktop (Windows App)          │  │
│  │    - GUI                                 │  │
│  │    - Settings                            │  │
│  │    - com.docker.backend.exe              │  │
│  └──────────────────────────────────────────┘  │
│                     │                          │
│                     │ Controls                 │
│                     ▼                          │
│  ┌──────────────────────────────────────────┐  │
│  │         WSL2 VM                          │  │
│  │                                          │  │
│  │  ┌────────────────────────────────────┐  │  │
│  │  │  docker-desktop (distro)           │  │  │
│  │  │  - Docker Engine (dockerd)         │  │  │
│  │  │  - Kubernetes (optional)           │  │  │
│  │  └────────────────────────────────────┘  │  │
│  │                                          │  │
│  │  ┌────────────────────────────────────┐  │  │
│  │  │  docker-desktop-data (distro)      │  │  │
│  │  │  - Container images                │  │  │
│  │  │  - Volumes                         │  │  │
│  │  └────────────────────────────────────┘  │  │
│  │                                          │  │
│  │  ┌────────────────────────────────────┐  │  │
│  │  │  Ubuntu (your distro)              │  │  │
│  │  │  - Your files                      │  │  │
│  │  │  - docker CLI (client only)        │  │  │
│  │  └────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────┘  │
└────────────────────────────────────────────────┘
```

### Important Facts
1. **Docker Engine runs in `docker-desktop` distro**, not your Ubuntu
2. **Your Ubuntu only has the Docker CLI** (talks to the engine remotely)
3. **`docker-desktop` is a separate WSL2 instance** with its own network namespace

---

## Part 7: Why Your Docker Swarm Command Failed

### The Command
```bash
docker swarm init --advertise-addr 100.66.197.8
```

### What Happens
```
1. You run command in Ubuntu WSL2
   ↓
2. Docker CLI sends request to Docker Engine
   ↓
3. Docker Engine (in docker-desktop distro) tries to bind to 100.66.197.8
   ↓
4. Checks available interfaces in docker-desktop namespace
   ↓
5. Finds: eth0 (172.X.X.Z), lo (127.0.0.1)
   ↓
6. Does NOT find: 100.66.197.8 (that's in Windows namespace!)
   ↓
7. Error: "cannot assign requested address"
```

### The Root Cause
```
Windows namespace:        Tailscale (100.66.197.8) ✓
                          ↕ (isolated)
docker-desktop namespace: No Tailscale ✗
```

---

## Part 8: Solutions

### Option 1: Install Tailscale in WSL2 Ubuntu
```bash
# In Ubuntu WSL2
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

**Result**: Ubuntu WSL2 gets its own Tailscale interface (e.g., `100.66.197.9`)

### Option 2: Use Mirrored Networking (Windows 11)
Edit `.wslconfig`, restart WSL. Now WSL2 sees Windows interfaces.

### Option 3: Don't Use Docker Desktop
Install Docker Engine directly in Ubuntu WSL2:
```bash
# Uninstall Docker Desktop
# Then in Ubuntu:
curl -fsSL https://get.docker.com | sh
```

Now Docker and Tailscale are in the same namespace.

---

## Part 9: Hands-On Exploration

### Check Your WSL Version
```powershell
wsl --version
```

### List All WSL Distros
```powershell
wsl -l -v
```

You should see:
```
  NAME                   STATE           VERSION
* Ubuntu                 Running         2
  docker-desktop         Running         2
  docker-desktop-data    Stopped         2
```

### Enter docker-desktop Distro
```powershell
wsl -d docker-desktop
```

Then:
```bash
ip addr  # See its network interfaces
ps aux   # See running processes (dockerd!)
```

### Compare with Your Ubuntu
```powershell
wsl -d Ubuntu
```

```bash
ip addr  # Different interfaces!
which docker  # Only the CLI, not dockerd
```

---

## Part 10: Common Misconceptions

### ❌ "WSL2 is just a compatibility layer"
**Reality**: It's a full Linux kernel in a VM.

### ❌ "WSL2 and Windows share the same network"
**Reality**: They're isolated by default (NAT mode).

### ❌ "Docker Desktop installs Docker in my Ubuntu"
**Reality**: It installs Docker in its own `docker-desktop` distro.

### ❌ "I can just bind to any Windows IP from WSL2"
**Reality**: Only if using mirrored mode (Windows 11) or if the service is in the same namespace.

---

## Key Takeaways

1. **WSL2 = Real Linux VM** using Hyper-V
2. **Default networking = NAT** (isolated from Windows)
3. **Docker Desktop = Separate WSL2 distro** (`docker-desktop`)
4. **Network namespaces = Why Tailscale IP is invisible** to Docker
5. **Solutions exist**: Tailscale in WSL2, mirrored mode, or native Docker

---

## Further Reading
- [WSL2 Architecture (Microsoft Docs)](https://learn.microsoft.com/en-us/windows/wsl/compare-versions)
- [WSL2 Networking (Microsoft Docs)](https://learn.microsoft.com/en-us/windows/wsl/networking)
- [Docker Desktop WSL2 Backend](https://docs.docker.com/desktop/wsl/)
