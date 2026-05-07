# IP Address Binding & Socket Programming: Complete Guide

## What You'll Learn
- What the `bind()` syscall does
- Why "cannot assign requested address" happens
- Binding to specific IPs vs `0.0.0.0`
- The `ip_nonlocal_bind` sysctl
- How Docker Swarm uses binding

**Prerequisites**: None! We'll explain sockets and networking from scratch.

---

## Socket Programming Basics

Before diving into socket binding, let's understand what sockets are. If you're familiar with sockets, skip to [Part 1](#part-1-what-is-a-socket).

### What is a Socket?

A **socket** is an endpoint for sending or receiving data across a network.

**Analogy:**
```
Socket = Phone jack on the wall
- You plug your phone into it
- Now you can make/receive calls
- Each jack has an address (phone number)
```

**Network socket:**
```
Socket = IP address + Port number
Example: 192.168.1.100:8080
         └─ IP address ─┘ └port┘
```

**Why sockets exist:**
- Allow programs to communicate over networks
- Provide standard interface for network programming
- Work with different protocols (TCP, UDP)

### What Does "Bind" Mean?

To **bind** a socket means to assign it a specific IP address and port number.

**Before binding:**
```
Program creates socket
Socket has no address yet
Can't receive connections
```

**After binding:**
```
Program binds socket to 192.168.1.100:8080
Socket now has an address
Can receive connections on that address
```

**Code example (Python):**
```python
import socket

# Create socket
s = socket.socket()

# Bind to IP and port
s.bind(('192.168.1.100', 8080))

# Now listening on 192.168.1.100:8080
```

**Why binding matters:**
- Only one program can bind to a specific IP:port combination
- If another program tries: "Address already in use" error
- This is the core issue with Docker Swarm and Tailscale!

### What Does "Listen" Mean?

After binding, a socket must **listen** for incoming connections.

**The process:**
```
1. Create socket
2. Bind to address (192.168.1.100:8080)
3. Listen for connections
4. Accept connections when they arrive
```

**Code example:**
```python
# After binding...
s.listen(5)  # Allow up to 5 pending connections

# Wait for a client to connect
client, address = s.accept()
print(f"Connection from {address}")
```

### TCP vs UDP (Revisited)

Sockets can use different protocols (covered in Part 3, but quick recap):

**TCP Sockets:**
- Connection-oriented
- Reliable delivery
- Used for: Web servers, databases, SSH

**UDP Sockets:**
- Connectionless
- No delivery guarantee
- Used for: DNS, video streaming, VXLAN

### What is a File Descriptor?

In Unix/Linux, **everything is a file** - including network sockets.

A **file descriptor** is a number that represents an open file or socket.

**Example:**
```
File descriptor 0: Standard input (keyboard)
File descriptor 1: Standard output (screen)
File descriptor 2: Standard error
File descriptor 3: Your socket
File descriptor 4: Another socket
File descriptor 5: An open file
```

**Why this matters:**
```python
# When you create a socket, you get a file descriptor
sock = socket.socket()  # Returns file descriptor (e.g., 3)

# You can read/write to it like a file
sock.send(b"Hello")     # Write to file descriptor
data = sock.recv(1024)  # Read from file descriptor
```

### What is a System Call?

A **system call** (syscall) is how programs ask the operating system to do something.

**Socket-related syscalls:**
```c
socket()   // Create a socket
bind()     // Bind to an address
listen()   // Listen for connections
accept()   // Accept a connection
connect()  // Connect to a server
send()     // Send data
recv()     // Receive data
close()    // Close socket
```

**How it works:**
```
Your Program
    ↓
    Calls bind(socket, "192.168.1.100:8080")
    ↓
Operating System Kernel
    ↓
    Checks: Is this IP on a local interface?
    Checks: Is this port already in use?
    ↓
    If yes: Bind succeeds
    If no: Returns error "Cannot assign requested address"
```

**This is exactly what happens with Docker Swarm:**
```
Docker calls: bind(socket, "100.66.197.8:2377")
              ↓
Kernel checks: Do I have interface with 100.66.197.8?
              ↓
In docker-desktop namespace: NO
              ↓
Error: "Cannot assign requested address"
```

---

## Part 1: What is Socket Binding?

### The Basics
When a program wants to listen for network connections, it must:
1. Create a **socket** (communication endpoint)
2. **Bind** the socket to an IP address and port
3. **Listen** for incoming connections

### Example: Web Server
```
Web server wants to listen on port 80:
1. Create socket
2. Bind to 0.0.0.0:80 (all interfaces)
3. Listen for HTTP requests
```

---

## Part 2: The `bind()` System Call

### What It Does
```c
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

**Purpose**: Associates a socket with a specific IP address and port.

### Simple Example (C)
```c
#include <sys/socket.h>
#include <netinet/in.h>

int main() {
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(8080);  // Port 8080
    addr.sin_addr.s_addr = inet_addr("192.168.1.10");  // Specific IP
    
    if (bind(sock, (struct sockaddr*)&addr, sizeof(addr)) < 0) {
        perror("bind failed");
        return 1;
    }
    
    listen(sock, 5);
    // Now listening on 192.168.1.10:8080
}
```

---

## Part 3: Binding to Different Addresses

### Option 1: Bind to Specific IP
```c
addr.sin_addr.s_addr = inet_addr("192.168.1.10");
```

**Result**: Only listens on `192.168.1.10:8080`

```
Connections to 192.168.1.10:8080  ✓ Accepted
Connections to 127.0.0.1:8080     ✗ Refused
Connections to 10.0.0.1:8080      ✗ Refused
```

### Option 2: Bind to `0.0.0.0` (All Interfaces)
```c
addr.sin_addr.s_addr = INADDR_ANY;  // 0.0.0.0
```

**Result**: Listens on **all** available interfaces

```
Connections to 192.168.1.10:8080  ✓ Accepted
Connections to 127.0.0.1:8080     ✓ Accepted
Connections to 10.0.0.1:8080      ✓ Accepted (if interface exists)
```

### Option 3: Bind to Localhost
```c
addr.sin_addr.s_addr = inet_addr("127.0.0.1");
```

**Result**: Only local connections accepted

```
Connections from same machine     ✓ Accepted
Connections from other machines   ✗ Refused
```

---

## Part 4: The "Cannot Assign Requested Address" Error

### Error Code
```
EADDRNOTAVAIL (99): Cannot assign requested address
```

### When It Happens
```c
addr.sin_addr.s_addr = inet_addr("100.66.197.8");
bind(sock, ...);
// Error: EADDRNOTAVAIL
```

### Why It Happens
The kernel checks:
```
1. Does the IP 100.66.197.8 exist on any local interface?
2. Check all interfaces:
   - lo: 127.0.0.1 ✗
   - eth0: 192.168.1.10 ✗
   - wlan0: 192.168.0.50 ✗
3. IP not found!
4. Return error: EADDRNOTAVAIL
```

**Rule**: You can only bind to IPs that exist on local interfaces.

---

## Part 5: Checking Available IPs

### Linux: `ip addr`
```bash
ip addr
```

Output:
```
1: lo: <LOOPBACK,UP,LOWER_UP>
    inet 127.0.0.1/8 scope host lo

2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP>
    inet 192.168.1.10/24 scope global eth0

3: tailscale0: <POINTOPOINT,MULTICAST,NOARP,UP>
    inet 100.66.197.8/32 scope global tailscale0
```

**You can bind to**:
- ✅ `127.0.0.1`
- ✅ `192.168.1.10`
- ✅ `100.66.197.8`
- ❌ `10.0.0.1` (doesn't exist)

### Testing Binding
```bash
# Try to bind to an IP
python3 -c "
import socket
s = socket.socket()
s.bind(('100.66.197.8', 8080))
print('Success!')
"
```

If the IP doesn't exist:
```
OSError: [Errno 99] Cannot assign requested address
```

---

## Part 6: The `ip_nonlocal_bind` Sysctl

### What It Does
Allows binding to IPs that **don't exist** on local interfaces.

### Enable It
```bash
sudo sysctl -w net.ipv4.ip_nonlocal_bind=1

# Make it permanent
echo "net.ipv4.ip_nonlocal_bind=1" | sudo tee -a /etc/sysctl.conf
```

### Use Case
**Load balancers** and **high-availability** setups:
```
Server 1: Bind to 10.0.0.100 (VIP)
Server 2: Also bind to 10.0.0.100 (VIP)

Only one has the IP at a time (via failover)
But both can bind to it in advance
```

### Why It Doesn't Help with Docker Swarm
Even with `ip_nonlocal_bind`:
- ✅ You can **bind** to the IP
- ❌ Incoming traffic still can't reach it (no route!)

**Swarm needs bi-directional communication**, so this doesn't solve the problem.

---

## Part 7: How Docker Swarm Uses Binding

### The `--advertise-addr` Flag
```bash
docker swarm init --advertise-addr 100.66.197.8
```

### What Docker Does
```
1. Bind to 100.66.197.8:2377 (cluster management)
2. Bind to 100.66.197.8:7946 (node discovery)
3. Bind to 100.66.197.8:4789 (VXLAN overlay)
```

### Why It Needs to Bind
Other nodes need to connect **to** this IP:
```
Worker Node:
  "I want to join the swarm at 100.66.197.8:2377"
      ↓
  Connects to manager
      ↓
Manager:
  Must be listening on 100.66.197.8:2377
```

If the manager can't bind to that IP, workers can't connect.

---

## Part 8: Your Specific Error

### The Setup
```
Windows:
  Tailscale interface: 100.66.197.8

docker-desktop WSL2 VM:
  eth0: 172.X.X.Y
  No Tailscale interface
```

### The Command
```bash
docker swarm init --advertise-addr 100.66.197.8
```

### What Happens
```
1. Docker Engine (in docker-desktop VM) calls bind()
2. Kernel checks: "Does 100.66.197.8 exist locally?"
3. Checks interfaces:
   - lo: 127.0.0.1 ✗
   - eth0: 172.X.X.Y ✗
4. Not found!
5. Returns: EADDRNOTAVAIL
6. Docker reports: "cannot assign requested address"
```

### The Fix
Put Tailscale in the **same namespace** as Docker:
```bash
# Install Tailscale in Ubuntu WSL2
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up

# Now check interfaces
ip addr show tailscale0
# inet 100.66.197.8/32 ✓

# Now Docker can bind to it!
docker swarm init --advertise-addr 100.66.197.8
```

---

## Part 9: The Deeper Issue - Explicit Binding and /32 Ownership

### Important: Docker Engine vs Containers

The Docker Engine is **not** a container. It's a system process that manages containers. When you run `docker swarm init --advertise-addr 100.66.197.8`, the **Docker Engine itself** tries to bind to that IP.

### What is Explicit Binding?

**Explicit binding** means a program requests a **specific IP address**, not all interfaces.

```
Two ways to bind:

1. Bind to all interfaces (0.0.0.0):
   bind(0.0.0.0:8080)
   → "Accept connections on ANY of my IP addresses"
   → Works on 127.0.0.1:8080, 192.168.1.100:8080, 100.66.197.8:8080

2. Explicit binding (specific IP):
   bind(100.66.197.8:8080)
   → "ONLY accept connections on 100.66.197.8:8080"
   → Requires ownership of that specific IP
```

### Why Docker Swarm Uses Explicit Binding

```bash
docker swarm init --advertise-addr 100.66.197.8

# Docker Engine says to the kernel:
# "I am the Swarm node with IP 100.66.197.8"
# "Give me exclusive control of 100.66.197.8:2377"
# "I need to listen ONLY on this IP, not others"
```

**Why not bind to 0.0.0.0?**
- Swarm needs to advertise a specific IP to other nodes
- Other nodes connect to that exact IP
- Binding to 0.0.0.0 would accept on all IPs (ambiguous)

### The /32 Subnet Kernel Restriction

Tailscale uses `/32` subnet masks, creating **point-to-point** connections:

```
Regular network interface:
eth0: 192.168.1.100/24
      └─ Part of a network with 256 addresses
      └─ Any program can bind to this IP

Tailscale interface:
tailscale0: 100.66.197.8/32
            └─ ONLY this one address (point-to-point)
            └─ Special kernel restriction!
```

### The Kernel's Strict Rule for /32 Addresses

> **Only the program that created the /32 interface can bind to it**

**Why this rule exists:**
```
/32 interfaces are special:
- They represent point-to-point connections
- The creating program (Tailscale) manages the connection
- Allowing other programs to bind could break the connection
- Kernel enforces ownership to prevent conflicts
```

### Step-by-Step: What Happens with Docker Swarm

```
1. Tailscale daemon starts:
   - Creates tailscale0 interface
   - Assigns IP: 100.66.197.8/32
   - Kernel marks: "Owned by Tailscale process (PID 1234)"

2. Docker Engine tries to bind:
   docker swarm init --advertise-addr 100.66.197.8
   
3. Docker Engine calls:
   bind(socket, 100.66.197.8:2377)
   
4. Kernel checks:
   - Does 100.66.197.8 exist? YES ✓
   - Is it a /32 interface? YES ✓
   - Is Docker the owner? NO ❌
   - Owner is: Tailscale daemon (PID 1234)
   
5. Kernel responds:
   "Access Denied. You don't own this address."
   Error: EADDRNOTAVAIL (Cannot assign requested address)
```

### Visual Representation

```
┌─────────────────────────────────────────────────────────┐
│                  Linux Kernel                           │
│                                                         │
│  Interface Ownership Table:                             │
│  ┌───────────────────────────────────────────────────┐ │
│  │ Interface    │ IP              │ Owner Process    │ │
│  ├───────────────────────────────────────────────────┤ │
│  │ lo           │ 127.0.0.1/8     │ (anyone)         │ │
│  │ eth0         │ 192.168.1.100/24│ (anyone)         │ │
│  │ tailscale0   │ 100.66.197.8/32 │ tailscaled (PID) │ │
│  │              │                 │ EXCLUSIVE! 🔒    │ │
│  └───────────────────────────────────────────────────┘ │
│                                                         │
│  Docker Engine tries to bind to 100.66.197.8:2377      │
│  ❌ Rejected: "Not the owner"                           │
└─────────────────────────────────────────────────────────┘
```

### Why This is Different from Namespace Isolation

Even if Docker Engine and Tailscale are in the **same namespace**, Docker still can't bind because of the `/32` ownership rule.

```
Same namespace scenario:
┌─────────────────────────────────────┐
│  Default Namespace                  │
│                                     │
│  ├─ Tailscale daemon (running)      │
│  │  └─ Owns 100.66.197.8/32 🔒      │
│  │                                  │
│  └─ Docker Engine (running)         │
│     └─ Tries to bind to             │
│        100.66.197.8:2377            │
│        ❌ Still fails!               │
│        (Not the owner)              │
└─────────────────────────────────────┘
```

### Two Separate Problems

1. **Namespace isolation** (containers):
   - Container can't see Tailscale interface
   - Solution: Use `--network host`

2. **Explicit binding + /32 ownership** (Docker Engine):
   - Docker Engine can see the interface
   - But kernel won't let it bind (not the owner)
   - Solution: Different approaches (covered in later parts)

### Summary Table

| Scenario | Can see interface? | Can bind to IP? | Why? |
|----------|-------------------|-----------------|------|
| Container (different namespace) | ❌ No | ❌ No | Not in namespace |
| Container (--network host) | ✓ Yes | ❌ No | Not owner of /32 |
| Docker Engine (same namespace) | ✓ Yes | ❌ No | Not owner of /32 |
| Tailscale daemon | ✓ Yes | ✓ Yes | Is the owner |

### The Key Insight

> Being in the same namespace is **necessary but not sufficient**. For /32 interfaces, you must also be the owning process.

This is why the solutions in later parts involve different approaches beyond just namespace management.

---

## Part 10: Binding in Different Languages

### Python
```python
import socket

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.bind(('192.168.1.10', 8080))
s.listen(5)
print("Listening on 192.168.1.10:8080")
```

### Go
```go
package main

import (
    "fmt"
    "net"
)

func main() {
    ln, err := net.Listen("tcp", "192.168.1.10:8080")
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    defer ln.Close()
    fmt.Println("Listening on 192.168.1.10:8080")
}
```

### Node.js
```javascript
const net = require('net');

const server = net.createServer();
server.listen(8080, '192.168.1.10', () => {
    console.log('Listening on 192.168.1.10:8080');
});
```

All will fail with `EADDRNOTAVAIL` if the IP doesn't exist!

---

## Part 10: Debugging Binding Issues

### Step 1: Check Available IPs
```bash
ip addr
```

### Step 2: Try Binding Manually
```bash
# Python one-liner
python3 -c "import socket; s=socket.socket(); s.bind(('100.66.197.8', 9999)); s.listen(1); print('OK')"
```

### Step 3: Check for Port Conflicts
```bash
# See what's already using the port
sudo netstat -tulpn | grep :8080
```

### Step 4: Check Firewall
```bash
# Ubuntu/Debian
sudo ufw status

# CentOS/RHEL
sudo firewall-cmd --list-all
```

### Step 5: Check SELinux (if applicable)
```bash
sestatus
```

---

## Part 11: Special Cases

### IPv6 Binding
```python
# Bind to IPv6 address
s = socket.socket(socket.AF_INET6, socket.SOCK_STREAM)
s.bind(('::1', 8080))  # IPv6 localhost
```

### Dual-Stack (IPv4 + IPv6)
```python
s = socket.socket(socket.AF_INET6, socket.SOCK_STREAM)
s.setsockopt(socket.IPPROTO_IPV6, socket.IPV6_V6ONLY, 0)
s.bind(('::', 8080))  # Accepts both IPv4 and IPv6
```

### Unix Domain Sockets
```python
# Bind to a file path instead of IP:port
s = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
s.bind('/tmp/my.sock')
```

---

## Part 12: Port Reuse

### The `SO_REUSEADDR` Option
```python
s = socket.socket()
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
s.bind(('0.0.0.0', 8080))
```

**Allows**:
- Reusing a port immediately after a process dies
- Multiple processes binding to the same port (with multicast)

**Doesn't help with**:
- Binding to non-existent IPs

---

## Part 13: Practical Examples

### Example 1: Simple HTTP Server
```python
import socket

s = socket.socket()
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
s.bind(('0.0.0.0', 8080))
s.listen(5)

print("Server listening on all interfaces, port 8080")

while True:
    conn, addr = s.accept()
    print(f"Connection from {addr}")
    conn.send(b"HTTP/1.1 200 OK\r\n\r\nHello!\r\n")
    conn.close()
```

### Example 2: Bind to Specific Interface
```python
import socket

# Only listen on Tailscale interface
s = socket.socket()
s.bind(('100.66.197.8', 8080))
s.listen(5)

print("Server listening on Tailscale IP only")
```

---

## Key Takeaways

1. **`bind()` associates a socket with an IP and port**
2. **You can only bind to IPs that exist on local interfaces**
3. **`EADDRNOTAVAIL` = IP doesn't exist locally**
4. **`0.0.0.0` = bind to all interfaces**
5. **Docker Swarm needs to bind to the advertised IP**
6. **Namespace isolation causes binding failures**

---

## Further Reading
- [bind(2) man page](https://man7.org/linux/man-pages/man2/bind.2.html)
- [Socket Programming Guide](https://beej.us/guide/bgnet/)
- [ip_nonlocal_bind Documentation](https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt)
