# Docker Desktop Architecture: Visual Diagrams

This document contains mermaid diagrams illustrating all the networking concepts, data flows, and limitations we've discussed.

## Diagram 1: Traditional Docker on Linux (How It Should Work)

```mermaid
graph TB
    subgraph "Physical Machine - Linux Host"
        subgraph "Docker Engine"
            C1[Container A<br/>172.17.0.2]
            C2[Container B<br/>172.17.0.3]
            Bridge[docker0 Bridge<br/>172.17.0.1]
            Engine[Docker Engine<br/>dockerd]
        end
        
        Network[Linux Network Stack<br/>iptables/routing]
        NIC[Physical Network Card<br/>eth0: 192.168.1.100]
    end
    
    Internet[Internet]
    LocalNetwork[Other Devices on LAN<br/>192.168.1.x]
    
    C1 --> Bridge
    C2 --> Bridge
    Bridge --> Engine
    Engine --> Network
    Network --> NIC
    NIC --> Internet
    NIC --> LocalNetwork
    
    style C1 fill:#e1f5ff
    style C2 fill:#e1f5ff
    style Engine fill:#fff4e6
    style NIC fill:#f0f0f0
```

**Key Points:**
- Docker Engine runs directly on Linux
- Containers connect via docker0 bridge
- Network stack is in the kernel (fast)
- Physical NIC has real IP (192.168.1.100) accessible on LAN
- Direct path from containers to internet

---

## Diagram 2: Docker Desktop Architecture Overview

```mermaid
graph TB
    subgraph "Physical Machine - Windows/Mac Host"
        Desktop[Docker Desktop Application]
        Backend[Backend Process<br/>com.docker.backend.exe]
        VPNKit[vpnkit Process<br/>User-Space TCP/IP Stack]
        HostNetwork[Host Network Stack]
        HostNIC[Host Network Card<br/>192.168.1.50]
        
        subgraph "Hypervisor"
            subgraph "Linux VM"
                subgraph "Inside VM"
                    C1[Container A<br/>172.17.0.2]
                    C2[Container B<br/>172.17.0.3]
                    Bridge[docker0 Bridge<br/>172.17.0.1]
                    Engine[Docker Engine<br/>dockerd]
                    VMNetwork[VM Network Stack]
                    VirtNIC[Virtual NIC<br/>192.168.65.3]
                end
            end
        end
        
        SharedMem[Shared Memory<br/>AF_VSOCK]
    end
    
    Internet[Internet]
    LAN[Other Devices on LAN]
    
    C1 --> Bridge
    C2 --> Bridge
    Bridge --> Engine
    Engine --> VMNetwork
    VMNetwork --> VirtNIC
    VirtNIC -.->|No real network| SharedMem
    SharedMem -.->|Memory operations| VPNKit
    SharedMem -.->|Memory operations| Backend
    VPNKit --> HostNetwork
    Backend --> HostNetwork
    HostNetwork --> HostNIC
    HostNIC --> Internet
    HostNIC --> LAN
    
    style C1 fill:#e1f5ff
    style C2 fill:#e1f5ff
    style Engine fill:#fff4e6
    style SharedMem fill:#ffe6e6
    style VPNKit fill:#fff0cc
    style Backend fill:#fff0cc
    style VirtNIC fill:#f0f0f0,stroke-dasharray: 5 5
```

**Key Points:**
- Docker Engine runs inside isolated VM
- VM has private IP (192.168.65.3) not routable outside
- No traditional network between VM and host
- Shared memory used for all host-VM communication
- vpnkit proxies all internet traffic

---

## Diagram 3: Container-to-Container Communication (Same Host)

```mermaid
sequenceDiagram
    participant C1 as Container A<br/>172.17.0.2
    participant Bridge as docker0 Bridge<br/>172.17.0.1
    participant C2 as Container B<br/>172.17.0.3
    
    Note over C1,C2: Containers on same Docker host
    
    C1->>Bridge: TCP packet to 172.17.0.3:8080
    Note over Bridge: VM's Linux kernel routes packet
    Bridge->>C2: Packet delivered
    C2->>Bridge: Response packet
    Bridge->>C1: Response delivered
    
    Note over C1,C2: Fast, direct communication<br/>All within VM's network namespace
```

**Key Points:**
- Happens entirely inside the VM
- Uses VM's Linux kernel networking
- No shared memory or vpnkit involved
- Fast and efficient
- Works identically in Docker Desktop and native Linux Docker

---

## Diagram 4: Published Port - Inbound Traffic Flow

```mermaid
sequenceDiagram
    participant Browser as Browser on Host<br/>Windows/Mac
    participant Backend as Docker Desktop Backend<br/>Listening on :8080
    participant SharedMem as Shared Memory<br/>AF_VSOCK
    participant VMBridge as Process in VM
    participant Container as nginx Container<br/>172.17.0.2:80
    
    Note over Browser,Container: User runs: docker run -p 8080:80 nginx
    
    Browser->>Backend: Connect to localhost:8080
    Note over Backend: Backend accepts TCP connection
    Backend->>SharedMem: Write connection data to memory
    Note over SharedMem: Memory operation (not networking!)
    SharedMem->>VMBridge: Read from shared memory
    VMBridge->>Container: Connect to 172.17.0.2:80
    Container->>VMBridge: HTTP response
    VMBridge->>SharedMem: Write response to memory
    SharedMem->>Backend: Read from shared memory
    Backend->>Browser: Send response
    
    Note over Browser,Container: Port forwarding via shared memory<br/>No real network between host and VM
```

**Key Points:**
- Backend process on host listens on port 8080
- Connection forwarded through shared memory (not network)
- VM process makes separate connection to container
- Bidirectional data copying
- Works for accessing containers from host

---

## Diagram 5: Container Outbound Traffic (Accessing Internet)

```mermaid
sequenceDiagram
    participant Container as Container<br/>172.17.0.2
    participant VMStack as VM Network Stack
    participant VirtNIC as Virtual NIC
    participant SharedMem as Shared Memory
    participant VPNKit as vpnkit<br/>User-Space Proxy
    participant HostStack as Host Network Stack
    participant Internet as Google.com
    
    Note over Container,Internet: Container runs: curl https://google.com
    
    Container->>VMStack: TCP SYN to google.com:443
    VMStack->>VirtNIC: Send Ethernet frame
    VirtNIC->>SharedMem: Write frame to memory
    Note over SharedMem: No network card - just memory!
    SharedMem->>VPNKit: Read Ethernet frame
    Note over VPNKit: Parse frame<br/>Extract IP packet<br/>Parse TCP packet
    VPNKit->>HostStack: connect() syscall to google.com
    Note over VPNKit: Connection appears to come from HOST
    HostStack->>Internet: Real TCP connection from host
    Internet->>HostStack: SYN-ACK response
    HostStack->>VPNKit: Deliver to vpnkit process
    VPNKit->>SharedMem: Construct fake SYN-ACK frame
    SharedMem->>VirtNIC: Read from memory
    VirtNIC->>VMStack: Deliver frame
    VMStack->>Container: TCP handshake complete!
    
    Note over Container,Internet: vpnkit proxies ALL traffic<br/>Makes it appear to come from host
```

**Key Points:**
- Container thinks it's using normal networking
- All traffic goes through shared memory to vpnkit
- vpnkit makes real connections from host
- VPN/firewall sees traffic from host, not VM
- Response path is reversed

---

## Diagram 6: Multi-Node Swarm - What It Needs

```mermaid
graph TB
    subgraph "Physical Machine 1 - 192.168.1.10"
        Engine1[Docker Engine 1<br/>Advertise: 192.168.1.10]
        subgraph "Overlay Network"
            C1[Container A<br/>10.0.0.2]
            C2[Container B<br/>10.0.0.3]
        end
        NIC1[NIC: 192.168.1.10]
    end
    
    subgraph "Physical Machine 2 - 192.168.1.11"
        Engine2[Docker Engine 2<br/>Advertise: 192.168.1.11]
        subgraph "Overlay Network 2"
            C3[Container C<br/>10.0.0.4]
            C4[Container D<br/>10.0.0.5]
        end
        NIC2[NIC: 192.168.1.11]
    end
    
    subgraph "Network Communication"
        Port2377[Port 2377 TCP<br/>Cluster Management]
        Port7946[Port 7946 TCP/UDP<br/>Node Discovery]
        Port4789[Port 4789 UDP<br/>VXLAN Overlay]
    end
    
    C1 --> Engine1
    C2 --> Engine1
    Engine1 --> NIC1
    NIC1 -.->|2377 TCP| Port2377
    NIC1 -.->|7946 TCP/UDP| Port7946
    NIC1 -.->|4789 UDP VXLAN| Port4789
    
    C3 --> Engine2
    C4 --> Engine2
    Engine2 --> NIC2
    NIC2 -.->|2377 TCP| Port2377
    NIC2 -.->|7946 TCP/UDP| Port7946
    NIC2 -.->|4789 UDP VXLAN| Port4789
    
    Port2377 -.->|Direct connection| Port2377
    Port7946 -.->|Direct connection| Port7946
    Port4789 -.->|Direct connection| Port4789
    
    style Engine1 fill:#c8e6c9
    style Engine2 fill:#c8e6c9
    style NIC1 fill:#bbdefb
    style NIC2 fill:#bbdefb
    style Port2377 fill:#fff9c4
    style Port7946 fill:#fff9c4
    style Port4789 fill:#fff9c4
```

**Requirements:**
- Multiple separate Docker Engine instances
- Each engine has routable IP (192.168.1.10, 192.168.1.11)
- Direct network connectivity between machines
- Open ports 2377, 7946, 4789
- VXLAN encapsulation for overlay networks

---

## Diagram 7: Docker Desktop - Why Multi-Node Swarm Fails (Problem 1)

```mermaid
graph TB
    subgraph "Computer 1 - Windows/Mac"
        subgraph "Docker Desktop VM"
            Engine1[Docker Engine<br/>IP: 192.168.65.3<br/>NOT ROUTABLE]
        end
        Host1[Host: 192.168.1.50]
        SharedMem1[Shared Memory]
    end
    
    subgraph "Computer 2 - Another Machine"
        Engine2[Docker Engine<br/>IP: 192.168.1.51<br/>ROUTABLE]
        NIC2[NIC: 192.168.1.51]
    end
    
    Engine1 -.-> SharedMem1
    SharedMem1 -.-> Host1
    Engine2 --> NIC2
    
    Engine2 -.->|"❌ Cannot reach 192.168.65.3<br/>Not on network!"| Engine1
    
    style Engine1 fill:#ffcdd2
    style Engine2 fill:#c8e6c9
    style SharedMem1 fill:#ffe6e6
```

**Problem:** Docker Desktop VM's IP (192.168.65.3) is not routable. Other machines cannot send packets to it.

---

## Diagram 8: Docker Desktop - Why Multi-Node Swarm Fails (Problem 2)

```mermaid
graph LR
    subgraph "Computer 1 - Docker Desktop"
        VM1[VM with Docker Engine<br/>192.168.65.3]
        SharedMem1[Shared Memory<br/>AF_VSOCK]
        Host1[Host]
    end
    
    subgraph "Computer 2"
        VM2[VM with Docker Engine<br/>192.168.65.3]
        SharedMem2[Shared Memory<br/>AF_VSOCK]
        Host2[Host]
    end
    
    VM1 <-->|✓ Works| SharedMem1
    SharedMem1 <-->|✓ Works| Host1
    VM2 <-->|✓ Works| SharedMem2
    SharedMem2 <-->|✓ Works| Host2
    
    VM1 -.->|"❌ AF_VSOCK only works locally<br/>Cannot communicate across machines"| VM2
    SharedMem1 -.->|"❌ Shared memory is per-host<br/>Not a network protocol"| SharedMem2
    
    style VM1 fill:#ffcdd2
    style VM2 fill:#ffcdd2
    style SharedMem1 fill:#ffe6e6
    style SharedMem2 fill:#ffe6e6
```

**Problem:** AF_VSOCK (shared memory) only works between VM and its host. Two VMs on different computers cannot use shared memory.

---

## Diagram 9: Docker Desktop - Why Multi-Node Swarm Fails (Problem 3)

```mermaid
sequenceDiagram
    participant C1 as Container on Node 1<br/>10.0.0.2
    participant Engine1 as Docker Engine 1<br/>in VM
    participant VPNKit as vpnkit
    participant Network as Physical Network
    participant Engine2 as Docker Engine 2<br/>on Node 2
    participant C2 as Container on Node 2<br/>10.0.0.5
    
    Note over C1,C2: Swarm overlay network needs VXLAN
    
    C1->>Engine1: Send data to 10.0.0.5
    Note over Engine1: Should encapsulate in VXLAN<br/>Send UDP to Engine2:4789
    Engine1->>VPNKit: Packet goes to shared memory
    Note over VPNKit: ❌ vpnkit doesn't understand VXLAN<br/>Can't proxy this correctly
    VPNKit--xNetwork: Cannot send VXLAN traffic
    
    Note over C1,C2: ❌ VXLAN overlay networks require<br/>direct network access between engines<br/>vpnkit proxying breaks this
```

**Problem:** Swarm overlay networks use VXLAN (UDP port 4789). vpnkit cannot proxy VXLAN correctly - it needs direct network access.

---

## Diagram 10: Docker Desktop - Why Multi-Node Swarm Fails (Problem 4)

```mermaid
graph TB
    subgraph "What Swarm Expects"
        N1[Node 1<br/>192.168.1.10:2377]
        N2[Node 2<br/>192.168.1.11:2377]
        N3[Node 3<br/>192.168.1.12:2377]
        
        N1 <-->|"Direct TCP<br/>Cluster management"| N2
        N2 <-->|"Direct TCP<br/>Cluster management"| N3
        N3 <-->|"Direct TCP<br/>Cluster management"| N1
    end
    
    subgraph "What Docker Desktop Provides"
        subgraph "VM - Isolated"
            Engine[Docker Engine<br/>192.168.65.3:2377]
        end
        Backend[Backend Process<br/>localhost:2377]
        SharedMem[Shared Memory]
        
        Backend -->|Port forwarding| SharedMem
        SharedMem --> Engine
    end
    
    N2 -.->|"❌ Cannot connect directly<br/>Only sees localhost:2377<br/>which is port forwarding"| Backend
    
    style N1 fill:#c8e6c9
    style N2 fill:#c8e6c9
    style N3 fill:#c8e6c9
    style Engine fill:#ffcdd2
```

**Problem:** Even with port publishing, other nodes can't connect properly. Swarm expects direct connection to Docker Engine, not forwarding through a proxy.

---

## Diagram 11: Single-Node Swarm on Docker Desktop (What Works)

```mermaid
graph TB
    subgraph "Docker Desktop - Single Machine"
        subgraph "VM - One Docker Engine"
            Manager[Manager Node<br/>Docker Engine<br/>192.168.65.3]
            
            subgraph "Swarm Services"
                subgraph "Service: web (3 replicas)"
                    R1[Replica 1]
                    R2[Replica 2]
                    R3[Replica 3]
                end
                
                subgraph "Service: db (1 replica)"
                    DB[Database]
                end
            end
            
            Overlay[Overlay Network<br/>10.0.0.0/24]
        end
        
        User[User: docker service create]
    end
    
    User -->|Commands| Manager
    Manager -->|Schedules| R1
    Manager -->|Schedules| R2
    Manager -->|Schedules| R3
    Manager -->|Schedules| DB
    
    R1 --> Overlay
    R2 --> Overlay
    R3 --> Overlay
    DB --> Overlay
    
    style Manager fill:#c8e6c9
    style R1 fill:#e1f5ff
    style R2 fill:#e1f5ff
    style R3 fill:#e1f5ff
    style DB fill:#e1f5ff
    style Overlay fill:#fff9c4
```

**What Works:**
- Initialize swarm: `docker swarm init`
- Create services with multiple replicas
- Overlay networks work (within single VM)
- Service discovery works
- Rolling updates work
- Everything runs on ONE node

**What Doesn't Work:**
- Cannot add worker nodes
- No high availability (if VM crashes, everything stops)
- Cannot distribute load across machines

---

## Diagram 12: Comparison - Native Linux vs Docker Desktop

```mermaid
graph TB
    subgraph "Native Linux Docker"
        direction TB
        L_App[Application]
        L_Container[Container]
        L_Engine[Docker Engine]
        L_Kernel[Linux Kernel Network Stack]
        L_NIC[Network Card<br/>Real IP: 192.168.1.100]
        L_Network[Physical Network]
        
        L_App --> L_Container
        L_Container --> L_Engine
        L_Engine --> L_Kernel
        L_Kernel --> L_NIC
        L_NIC <--> L_Network
        
        style L_NIC fill:#c8e6c9
    end
    
    subgraph "Docker Desktop"
        direction TB
        D_App[Application]
        D_Container[Container]
        D_Engine[Docker Engine]
        D_VMKernel[VM Linux Kernel]
        D_VirtNIC[Virtual NIC<br/>Private IP: 192.168.65.3]
        D_SharedMem[Shared Memory AF_VSOCK]
        D_VPNKit[vpnkit User-Space Stack]
        D_HostKernel[Host OS Network Stack]
        D_NIC[Network Card<br/>Real IP: 192.168.1.50]
        D_Network[Physical Network]
        
        D_App --> D_Container
        D_Container --> D_Engine
        D_Engine --> D_VMKernel
        D_VMKernel --> D_VirtNIC
        D_VirtNIC -.->|No network| D_SharedMem
        D_SharedMem --> D_VPNKit
        D_VPNKit --> D_HostKernel
        D_HostKernel --> D_NIC
        D_NIC <--> D_Network
        
        style D_VirtNIC fill:#ffcdd2,stroke-dasharray: 5 5
        style D_SharedMem fill:#ffe6e6
        style D_VPNKit fill:#fff0cc
    end
```

**Key Differences:**
- **Linux**: Direct kernel path, real network card with routable IP
- **Docker Desktop**: VM isolation, shared memory, vpnkit proxy, no routable VM IP

---

## Diagram 13: WSL2 Backend vs Hyper-V Backend (Same Networking!)

```mermaid
graph TB
    subgraph "Hyper-V Backend"
        HV_Host[Windows Host]
        subgraph "Hyper-V"
            HV_VM[DockerDesktopVM<br/>Single VM]
            HV_Engine[Docker Engine]
        end
        HV_SharedMem[AF_VSOCK<br/>Shared Memory]
        HV_VPNKit[vpnkit]
        
        HV_Engine --> HV_VM
        HV_VM <--> HV_SharedMem
        HV_SharedMem <--> HV_VPNKit
        HV_VPNKit --> HV_Host
    end
    
    subgraph "WSL2 Backend"
        WSL_Host[Windows Host]
        subgraph "WSL2"
            WSL_Docker[docker-desktop<br/>Distribution]
            WSL_Data[docker-desktop-data<br/>Distribution]
            WSL_Engine[Docker Engine]
        end
        WSL_SharedMem[AF_VSOCK<br/>Shared Memory]
        WSL_VPNKit[vpnkit]
        
        WSL_Engine --> WSL_Docker
        WSL_Data -.->|Storage| WSL_Docker
        WSL_Docker <--> WSL_SharedMem
        WSL_SharedMem <--> WSL_VPNKit
        WSL_VPNKit --> WSL_Host
    end
    
    Note1[Different VM Technology]
    Note2[SAME Networking Architecture]
    
    style HV_SharedMem fill:#ffe6e6
    style WSL_SharedMem fill:#ffe6e6
    style HV_VPNKit fill:#fff0cc
    style WSL_VPNKit fill:#fff0cc
    style Note2 fill:#c8e6c9
```

**Key Points:**
- Different VM hosting (Hyper-V vs WSL2)
- Same AF_VSOCK shared memory
- Same vpnkit proxying
- Same networking limitations
- Same inability to run multi-node swarm

---

## Diagram 14: The Fundamental Architectural Mismatch

```mermaid
graph LR
    subgraph "Swarm's Expectation"
        direction TB
        SE1[Independent<br/>Docker Engine 1]
        SE2[Independent<br/>Docker Engine 2]
        SE3[Independent<br/>Docker Engine 3]
        SNetwork[Real Network<br/>Direct communication<br/>Routable IPs]
        
        SE1 <--> SNetwork
        SE2 <--> SNetwork
        SE3 <--> SNetwork
        
        style SE1 fill:#c8e6c9
        style SE2 fill:#c8e6c9
        style SE3 fill:#c8e6c9
        style SNetwork fill:#bbdefb
    end
    
    subgraph "Docker Desktop's Reality"
        direction TB
        DE[Single Docker Engine<br/>in Isolated VM]
        DMem[Shared Memory<br/>Host-Local Only]
        DProxy[vpnkit Proxy<br/>User-Space]
        
        DE --> DMem
        DMem --> DProxy
        
        style DE fill:#ffcdd2
        style DMem fill:#ffe6e6
        style DProxy fill:#fff0cc
    end
    
    Mismatch[❌ FUNDAMENTAL MISMATCH<br/>Distributed system vs<br/>Isolated single-node system]
    
    Swarm's Expectation -.-> Mismatch
    Docker Desktop's Reality -.-> Mismatch
    
    style Mismatch fill:#f44336,color:#fff
```

**Summary:**
- Swarm needs: Multiple engines, real networking, direct communication
- Docker Desktop provides: Single engine, memory-based communication, proxy for external access
- These architectures are fundamentally incompatible

---

## Quick Reference: What Each Component Does

| Component | Purpose | Works Across Machines? |
|-----------|---------|----------------------|
| **AF_VSOCK** | Shared memory communication between VM and host | ❌ No - host-local only |
| **vpnkit** | User-space proxy for internet access | ❌ No - proxies outbound only |
| **Port Publishing** | Forward host ports to containers | ❌ No - forwarding, not real network |
| **docker0 Bridge** | Container networking within VM | ❌ No - exists only in VM |
| **Overlay Network** | Multi-host container networking | ❌ Not in Docker Desktop |
| **VXLAN (port 4789)** | Encapsulation for overlay networks | ❌ Requires direct network access |
| **Swarm Management (port 2377)** | Cluster orchestration | ❌ Needs routable Docker Engine IP |
| **Node Discovery (port 7946)** | Cluster membership | ❌ Needs direct node-to-node communication |

---

## The Complete Picture

```mermaid
graph TB
    Start[Want to run Docker Swarm?]
    
    Start --> Q1{How many physical<br/>machines/VMs?}
    
    Q1 -->|One| Single[Single-Node Swarm]
    Q1 -->|Multiple| Multi[Multi-Node Swarm]
    
    Single --> Q2{Using Docker Desktop?}
    Q2 -->|Yes| Works1[✅ Will work<br/>Limited to single node<br/>Good for testing]
    Q2 -->|No - Linux| Works2[✅ Will work<br/>Can add nodes later]
    
    Multi --> Q3{Using Docker Desktop?}
    Q3 -->|Yes| Fails[❌ Won't work<br/>Architecture incompatible<br/>VM isolation prevents it]
    Q3 -->|No - Real VMs/Machines| Q4{Do they have routable IPs?}
    
    Q4 -->|Yes| Q5{Are ports 2377, 7946, 4789 open?}
    Q4 -->|No| FixIP[Configure bridged networking<br/>or use cloud VMs]
    
    Q5 -->|Yes| Success[✅ Will work<br/>True multi-node swarm]
    Q5 -->|No| FixPorts[Open required ports<br/>in firewall]
    
    FixIP --> Q5
    FixPorts --> Success
    
    style Works1 fill:#c8e6c9
    style Works2 fill:#c8e6c9
    style Success fill:#c8e6c9
    style Fails fill:#ffcdd2
```

---

## Summary

These diagrams show:

1. **How traditional Docker works** - direct kernel networking
2. **Docker Desktop's architecture** - VM + shared memory + vpnkit
3. **Container-to-container communication** - works within VM
4. **Port publishing** - forwarding through shared memory
5. **Outbound traffic** - proxied through vpnkit
6. **What multi-node swarm needs** - direct network access
7. **Why Docker Desktop can't do it** - four fundamental problems
8. **What works** - single-node swarm for testing
9. **WSL2 vs Hyper-V** - same networking architecture
10. **The fundamental mismatch** - distributed vs isolated architecture

The core issue: Docker Swarm needs a distributed system with real networking. Docker Desktop provides an isolated single-node system with memory-based communication.
