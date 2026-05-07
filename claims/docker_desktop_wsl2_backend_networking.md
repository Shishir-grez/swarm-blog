# Docker Desktop WSL2 Backend: Same Architecture, Different VM

## Quick Answer

**Yes, the WSL2 backend uses the SAME networking architecture** (shared memory + vpnkit) as the Hyper-V backend. The only difference is which VM technology hosts the Docker Engine.

## The Two Backend Options

Docker Desktop on Windows offers two backend choices:

### Option 1: Hyper-V Backend
- Docker Desktop creates its own dedicated Hyper-V VM called `DockerDesktopVM`
- You can see this VM in PowerShell: `Get-VM`
- This VM is completely separate from WSL2
- Requires Windows Pro/Enterprise (Hyper-V is not available on Windows Home)

### Option 2: WSL2 Backend
- Docker Desktop runs inside the WSL2 VM managed by Windows
- Uses WSL2's lightweight Hyper-V-based virtualization
- Works on Windows Home (WSL2 is available on Home edition)
- You can see Docker's WSL distributions: `wsl -l -v`

## What's the Same: The Networking Architecture

**Both backends use identical networking:**

Docker Desktop uses the same VM processes for both WSL 2 (in the docker-desktop distribution) and Hyper-V (in DockerDesktopVM). Host/VM communication uses AF_VSOCK hypervisor sockets (shared memory) rather than network switches or interfaces.

This means:

### 1. Shared Memory Communication (AF_VSOCK)
- Both backends use AF_VSOCK for host-VM communication
- No traditional network interfaces between host and VM
- Fast, memory-based data transfer

### 2. vpnkit for Outbound Traffic
The WSL 2 backend leverages our efforts in this area, using vpnkit to ensure a VPN-friendly networking stack

Both backends use:
- vpnkit as a user-space TCP/IP proxy
- Same VPN-friendly design
- Same HTTP proxy support
- Same network isolation

### 3. Port Publishing
- Both use the same forwarding mechanism
- Docker Desktop backend processes (`com.docker.backend.exe`, `com.docker.vpnkit.exe`) listen on host ports
- Traffic forwarded through shared memory to containers

### 4. Container IP Addresses
- Containers get private IPs (like `172.17.0.x`)
- These IPs exist only within the VM
- Not routable from external networks

## What's Different: The VM Itself

### Hyper-V Backend

**Architecture:**
```
Windows Host
  └─ Hyper-V
       └─ DockerDesktopVM (single VM)
            ├─ Docker Engine (dockerd)
            └─ Containers
```

**Characteristics:**
- Single dedicated VM named `DockerDesktopVM`
- Built with LinuxKit (Docker's custom minimal Linux)
- You manually configure CPU/RAM in Docker Desktop settings
- Completely isolated from any other VMs

### WSL2 Backend

**Architecture:**
```
Windows Host
  └─ WSL2 (Hyper-V under the hood)
       ├─ docker-desktop (distribution)
       │    ├─ Docker Engine (dockerd)
       │    └─ Kubernetes (if enabled)
       ├─ docker-desktop-data (distribution)
       │    └─ Storage for images/volumes
       └─ Ubuntu/Debian/etc (optional user distributions)
            └─ Can integrate with Docker Desktop
```

**Two Special Distributions:**

1. **docker-desktop**: Runs the Docker Engine
   - This is where `dockerd` actually runs
   - Built with LinuxKit (same as Hyper-V backend)
   - Isolated in its own namespace

2. **docker-desktop-data**: Stores container data
   - Holds images, volumes, and container filesystems
   - Separated to make backup/migration easier

**Why two distributions?** The docker-desktop-data distro exists as a storage for images and configs, as well as the Kubernetes data store. Those are used by the docker-desktop distro, the same result is achieved when docker is run under Hyper-V by mounting a VHD (Virtual Hard Disk) in the Hyper-V image but mounting a VHD isn't possible with WSL2 yet.

**Characteristics:**
- Uses WSL2's shared VM infrastructure
- Dynamic CPU/RAM allocation (WSL2 manages this automatically)
- Can integrate with user WSL distributions (Ubuntu, Debian, etc.)

## WSL2 Integration Feature

This is an extra feature ONLY available with WSL2 backend:

When enabled, you can run Docker commands from inside your WSL2 distributions (like Ubuntu):

```bash
# From inside WSL2 Ubuntu
docker run -it ubuntu bash
```

**How it works:**
- Docker CLI in your WSL distribution connects to the Docker Engine in `docker-desktop`
- Uses Unix socket forwarding
- The containers still run in the `docker-desktop` distribution, not in your Ubuntu distribution

**Important:** Even with integration enabled, the Docker Engine runs in the isolated `docker-desktop` distribution. Your Ubuntu distribution just gets access to control it.

## Network Architecture Details (Same for Both Backends)

### Inbound Traffic (Port Publishing)

```
Browser on Windows
       ↓
  localhost:8080
       ↓
[com.docker.backend.exe on Host]
       ↓
  AF_VSOCK Shared Memory
       ↓
[Docker Engine in VM] (either DockerDesktopVM or docker-desktop)
       ↓
[Container]
```

### Outbound Traffic (Internet Access)

```
[Container in VM]
       ↓
[VM's Linux Network Stack]
       ↓
  AF_VSOCK Shared Memory
       ↓
[vpnkit on Host]
       ↓
[Host's Network Stack]
       ↓
[Internet]
```

This flow is **identical** whether you're using Hyper-V or WSL2 backend!

## WSL2 Mirrored Networking Mode (New Feature)

Starting with Docker Desktop 4.26.0 and WSL 2.0.4+, there's a NEW experimental mode called "mirrored networking":

**What it does:**
- Makes WSL2 VM's network interfaces mirror the Windows host's network interfaces
- Allows direct LAN access from WSL2 applications
- Simplifies networking between WSL and Windows

**How to enable:**
Edit `%UserProfile%\.wslconfig`:
```ini
[wsl2]
networkingMode=mirrored
```

**Important notes:**
- This is for WSL2 generally, not just Docker Desktop
- Still experimental and may have bugs
- Docker Desktop itself still uses vpnkit for container networking
- This primarily helps WSL applications (not running in containers) access your network

## Why Both Backends Can't Do Multi-Node Swarm

The networking architecture is the same, so the limitations are identical:

### Problem 1: Non-Routable VM IP
- Hyper-V backend: `DockerDesktopVM` has internal IP (like `192.168.65.3`)
- WSL2 backend: `docker-desktop` distribution has internal IP (like `172.28.161.30`)
- Both IPs are not routable from other computers

### Problem 2: Shared Memory Only Works Locally
- AF_VSOCK works between VM and host on same computer
- Cannot work between VMs on different computers
- Both backends use AF_VSOCK

### Problem 3: vpnkit Proxying
- Both backends proxy all traffic through vpnkit
- No direct network path for VXLAN overlay networks
- Swarm's UDP port 4789 traffic can't be proxied correctly

### Problem 4: Single Docker Engine
- Hyper-V backend: One Docker Engine in `DockerDesktopVM`
- WSL2 backend: One Docker Engine in `docker-desktop` distribution
- Both have only one Docker Engine instance

## Memory and Resource Management

This is one area where the backends differ:

### Hyper-V Backend
- You manually set CPU and RAM in Docker Desktop settings
- Resources are fixed and reserved
- VM always uses those resources even if idle

### WSL2 Backend
Because docker is now running in a WSL distro it gets access to all the CPU cores and by default 80% of the RAM. This is the shared across all distros running under WSL 2 and the allocation of resources will be handled dynamically for you by WSL.

- Dynamic allocation by WSL2
- Uses only what's needed
- Shares resources with other WSL2 distributions
- Can configure limits in `.wslconfig` file

## Summary Table

| Aspect | Hyper-V Backend | WSL2 Backend |
|--------|----------------|--------------|
| **VM Technology** | Dedicated Hyper-V VM | WSL2 (Hyper-V under hood) |
| **Host-VM Communication** | AF_VSOCK shared memory | AF_VSOCK shared memory |
| **Outbound Networking** | vpnkit proxy | vpnkit proxy |
| **Port Publishing** | Backend process forwarding | Backend process forwarding |
| **Container IPs** | Private, non-routable | Private, non-routable |
| **Resource Allocation** | Manual, fixed | Dynamic, automatic |
| **Windows Requirement** | Pro/Enterprise | Home/Pro/Enterprise |
| **WSL Integration** | Not available | Can integrate with WSL distros |
| **Multi-Node Swarm** | ❌ No | ❌ No |
| **Networking Architecture** | ✅ Same | ✅ Same |

## The Bottom Line

**The networking architecture is fundamentally the same.** Whether you use Hyper-V or WSL2 backend:
- Shared memory (AF_VSOCK) for host-VM communication
- vpnkit for internet access
- Port forwarding for inbound traffic
- No routable IP addresses
- Cannot support multi-node Docker Swarm

The WSL2 backend is just a more efficient VM hosting mechanism with better resource management and Windows Home compatibility. The network isolation problems remain identical.
