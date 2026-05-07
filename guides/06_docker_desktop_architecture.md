# Docker Desktop Architecture: Complete Guide

## What You'll Learn
- How Docker Desktop works on Windows
- The WSL2 backend architecture
- Why Docker Desktop isolates networking
- vpnkit and network proxying
- Differences from native Docker Engine

**Prerequisites**: Understanding of WSL2 (see [Part 2](02_wsl2_architecture.md)) and Docker basics (see [Part 5](05_docker_swarm_mode.md))

> **Note**: This guide assumes you've read Parts 2 and 5. Key terms like VM, Docker, containers, and images are explained there.

---

## Part 1: What is Docker Desktop?

### The Simple Answer
> Docker Desktop is a **GUI application** that makes Docker easy to use on Windows and Mac by running Docker Engine in a VM.

### What It Includes
- Docker Engine (dockerd)
- Docker CLI (docker command)
- Docker Compose
- Kubernetes (optional)
- GUI for management
- Automatic updates

---

## Part 2: Docker Desktop vs Docker Engine

### Docker Engine (Native Linux)
```
Linux Host
  ├─ dockerd (daemon)
  ├─ containerd
  └─ runc (container runtime)
```

**Direct access** to Linux kernel features (namespaces, cgroups).

### Docker Desktop (Windows/Mac)
```
Windows/Mac Host
  └─ Docker Desktop App
      └─ Virtual Machine
          ├─ Linux Kernel
          ├─ dockerd
          ├─ containerd
          └─ runc
```

**Indirect access** via a VM layer.

---

## Part 3: Docker Desktop on Windows (WSL2 Backend)

### The Architecture
```
┌──────────────────────────────────────────────────┐
│              Windows 10/11 Host                  │
│                                                  │
│  ┌────────────────────────────────────────────┐  │
│  │   Docker Desktop (Windows App)             │  │
│  │   - GUI (systray icon)                     │  │
│  │   - Settings                               │  │
│  │   - com.docker.backend.exe                 │  │
│  │   - com.docker.proxy.exe                   │  │
│  └────────────────┬───────────────────────────┘  │
│                   │                              │
│                   │ Controls                     │
│                   ▼                              │
│  ┌────────────────────────────────────────────┐  │
│  │         Hyper-V / WSL2                     │  │
│  │                                            │  │
│  │  ┌──────────────────────────────────────┐  │  │
│  │  │  docker-desktop (WSL2 distro)        │  │  │
│  │  │  - Alpine Linux                      │  │  │
│  │  │  - dockerd (Docker Engine)           │  │  │
│  │  │  - Kubernetes (optional)             │  │  │
│  │  └──────────────────────────────────────┘  │  │
│  │                                            │  │
│  │  ┌──────────────────────────────────────┐  │  │
│  │  │  docker-desktop-data (WSL2 distro)   │  │  │
│  │  │  - Container images                  │  │  │
│  │  │  - Volumes                           │  │  │
│  │  │  - Build cache                       │  │  │
│  │  └──────────────────────────────────────┘  │  │
│  │                                            │  │
│  │  ┌──────────────────────────────────────┐  │  │
│  │  │  Ubuntu (User's WSL2 distro)         │  │  │
│  │  │  - docker CLI only (client)          │  │  │
│  │  │  - Talks to dockerd via socket       │  │  │
│  │  └──────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────┘
```

### Key Components

#### 1. `docker-desktop` Distro
- **Purpose**: Runs Docker Engine
- **OS**: Alpine Linux (lightweight)
- **Contains**: dockerd, containerd, Kubernetes

#### 2. `docker-desktop-data` Distro
- **Purpose**: Stores Docker data
- **Contains**: Images, volumes, build cache

#### 3. User WSL2 Distros (e.g., Ubuntu)
- **Purpose**: Your development environment
- **Contains**: Only Docker CLI (not the engine!)
- **Connects to**: `docker-desktop` via socket

---

## Part 4: How Communication Works

### The Docker Socket
```
Ubuntu WSL2:
  docker ps
      ↓
  /var/run/docker.sock (Unix socket)
      ↓
  Forwarded to docker-desktop distro
      ↓
  dockerd (in docker-desktop)
      ↓
  Returns container list
```

### Socket Mounting
Docker Desktop **mounts** the socket into your WSL2 distros:
```bash
# In Ubuntu WSL2
ls -la /var/run/docker.sock
# lrwxrwxrwx ... /var/run/docker.sock -> /mnt/wsl/docker-desktop/...
```

It's a **symlink** to the actual socket in `docker-desktop`.

---

## Part 5: Networking Architecture

### The Problem
```
Windows Network:
  - Physical NIC (192.168.1.100)
  - Tailscale (100.66.197.8)

docker-desktop VM:
  - eth0 (172.X.X.Y)
  - No Tailscale interface!
```

How do containers access Windows network?

### The Solution: vpnkit

**vpnkit** is a network proxy that bridges Windows and the VM.

```
Container (172.17.0.2)
      ↓
docker0 bridge
      ↓
docker-desktop VM (172.X.X.Y)
      ↓
vpnkit (proxy)
      ↓
Windows Host (192.168.1.100)
      ↓
Internet
```

### What vpnkit Does
1. **Intercepts** network traffic from containers
2. **Proxies** it through the Windows network stack
3. **Returns** responses to containers

### What vpnkit Does NOT Do
- ❌ Make Windows interfaces visible in the VM
- ❌ Allow binding to Windows IPs from containers
- ❌ Provide bi-directional access (only outbound)

---

## Part 6: Port Forwarding

### Publishing Ports
```bash
docker run -p 8080:80 nginx
```

### What Happens
```
1. Container listens on port 80 (inside VM)
2. Docker Desktop forwards port 8080 on Windows to port 80 in container
3. You can access: http://localhost:8080 from Windows
```

### The Mechanism
```
Windows:
  localhost:8080
      ↓
com.docker.proxy.exe (Windows process)
      ↓
Forwards to docker-desktop VM
      ↓
Container port 80
```

### Limitations
- ✅ Works for **localhost** access
- ✅ Works for **Windows → Container**
- ❌ Doesn't work for **external IPs binding**
- ❌ Doesn't work for **bi-directional cluster communication**

---

## Part 7: Why Docker Swarm Fails with Tailscale

### The Command
```bash
docker swarm init --advertise-addr 100.66.197.8
```

### What Happens
```
1. Docker CLI (Ubuntu WSL2) sends command to dockerd
2. dockerd (in docker-desktop VM) tries to bind to 100.66.197.8
3. Checks available interfaces:
   - lo (127.0.0.1) ✓
   - eth0 (172.X.X.Y) ✓
   - tailscale0 (100.66.197.8) ✗ (doesn't exist!)
4. Error: "cannot assign requested address"
```

### Why It Fails
```
Windows namespace:
  ├─ Tailscale (100.66.197.8) ✓

docker-desktop namespace:
  ├─ eth0 (172.X.X.Y) ✓
  └─ No Tailscale ✗
```

**Different namespaces** = isolated network stacks.

### Why vpnkit Doesn't Help
vpnkit is **one-way** (outbound only):
- ✅ Containers can reach Tailscale network
- ❌ Containers can't **bind** to Tailscale IPs
- ❌ External nodes can't reach containers directly

---

## Part 8: File System Integration

### How It Works
Docker Desktop uses **9P protocol** to share files between Windows and the VM.

```
Windows:
  C:\Users\You\project\
      ↓
  9P file sharing
      ↓
docker-desktop VM:
  /mnt/c/Users/You/project/
      ↓
  Bind mount into container
      ↓
Container:
  /app/
```

### Performance Considerations
- ✅ **Fast**: Files inside WSL2 (`/home/...`)
- ⚠️ **Slow**: Files on Windows (`/mnt/c/...`)

**Recommendation**: Keep project files in WSL2 filesystem.

---

## Part 9: Resource Management

### Settings
Docker Desktop lets you configure:
- **CPUs**: How many cores containers can use
- **Memory**: RAM limit for Docker
- **Disk**: Max size for images/volumes

### Where It's Enforced
```
Windows:
  Docker Desktop Settings (GUI)
      ↓
  Configures WSL2 VM limits
      ↓
docker-desktop VM:
  Limited to configured resources
      ↓
Containers:
  Share the VM's resources
```

### The `.wslconfig` File
You can also configure WSL2 globally:
```ini
# C:\Users\You\.wslconfig
[wsl2]
memory=8GB
processors=4
```

This affects **all** WSL2 distros, including `docker-desktop`.

---

## Part 10: Docker Desktop vs Native Docker in WSL2

### Docker Desktop Approach
```
Ubuntu WSL2:
  - docker CLI only
  - Connects to docker-desktop VM

docker-desktop VM:
  - dockerd runs here
  - Isolated networking
```

**Pros**:
- Easy setup (GUI)
- Automatic updates
- Kubernetes integration

**Cons**:
- Network isolation issues
- Extra layer of abstraction
- Can't bind to Windows IPs

### Native Docker Approach
```
Ubuntu WSL2:
  - docker CLI
  - dockerd (runs directly in Ubuntu)
  - Same network namespace

Tailscale:
  - Also installed in Ubuntu WSL2
  - Same network namespace as dockerd
```

**Pros**:
- ✅ Docker and Tailscale in same namespace
- ✅ Can bind to Tailscale IPs
- ✅ Direct control

**Cons**:
- Manual setup
- Manual updates
- No GUI

---

## Part 11: Hands-On Exploration

### List WSL2 Distros
```powershell
wsl -l -v
```

Output:
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
# See the OS
cat /etc/os-release
# NAME="Alpine Linux"

# See running processes
ps aux | grep dockerd

# See network interfaces
ip addr
# No Tailscale interface!
```

### Check Docker Socket in Ubuntu
```bash
# In Ubuntu WSL2
ls -la /var/run/docker.sock
# Symlink to docker-desktop

# See where it points
readlink -f /var/run/docker.sock
```

### See Docker Desktop Processes on Windows
```powershell
Get-Process | Where-Object {$_.Name -like "*docker*"}
```

You'll see:
- `com.docker.backend`
- `com.docker.proxy`
- `Docker Desktop`

---

## Part 12: Migrating to Native Docker

### Step 1: Uninstall Docker Desktop
```powershell
# Uninstall via Windows Settings
# Or keep it and disable WSL2 integration
```

### Step 2: Install Docker in Ubuntu WSL2
```bash
# In Ubuntu WSL2
curl -fsSL https://get.docker.com | sh

# Add your user to docker group
sudo usermod -aG docker $USER

# Start Docker
sudo service docker start
```

### Step 3: Install Tailscale in Ubuntu WSL2
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

### Step 4: Verify
```bash
# Check Docker
docker ps

# Check Tailscale
ip addr show tailscale0

# Now both are in the same namespace!
```

### Step 5: Initialize Swarm
```bash
docker swarm init --advertise-addr $(tailscale ip -4)
# Should work now!
```

---

## Part 13: Docker Desktop Alternatives

### Option 1: Native Docker in WSL2
**Best for**: Swarm, Tailscale, full control

### Option 2: Rancher Desktop
- Similar to Docker Desktop
- Uses k3s instead of Docker Swarm
- Open source

### Option 3: Podman
- Daemonless (no dockerd)
- Rootless containers
- Docker-compatible CLI

### Option 4: Multipass + Docker
- Ubuntu VMs on Windows
- Full Linux environment
- More overhead

---

## Key Takeaways

1. **Docker Desktop = Docker Engine in a separate WSL2 VM**
2. **`docker-desktop` and `docker-desktop-data` are isolated distros**
3. **vpnkit proxies network traffic** (one-way, outbound only)
4. **Network isolation prevents binding to Windows IPs**
5. **Native Docker in WSL2 solves the Tailscale/Swarm issue**

---

## Further Reading
- [Docker Desktop WSL2 Backend](https://docs.docker.com/desktop/wsl/)
- [vpnkit GitHub](https://github.com/moby/vpnkit)
- [Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
