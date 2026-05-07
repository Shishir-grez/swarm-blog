# Docker Swarm Mode: Complete Guide

## What You'll Learn
- Docker Swarm architecture (managers vs workers)
- How Raft consensus works
- Service, task, and container relationships
- Networking requirements and ports
- Why `--advertise-addr` matters

**Prerequisites**: None! We'll explain Docker and orchestration from scratch.

---

## Docker and Orchestration Fundamentals

Before diving into Docker Swarm, let's understand containers, Docker, and orchestration concepts. If you're already familiar with Docker, skip to [Part 1](#part-1-what-is-docker-swarm).

### What is Docker?

**Docker** is a platform for running applications in isolated containers.

**Traditional deployment:**
```
Server
├─ Operating System
├─ App A (needs Python 3.8)
├─ App B (needs Python 3.10) ← Conflict!
└─ App C (needs specific libraries)
```

**With Docker:**
```
Server
├─ Operating System
├─ Docker Engine
    ├─ Container A (has Python 3.8)
    ├─ Container B (has Python 3.10) ← No conflict!
    └─ Container C (has its libraries)
```

**Benefits:**
- **Isolation**: Each app has its own environment
- **Portability**: "Works on my machine" → works everywhere
- **Efficiency**: Lighter than virtual machines
- **Consistency**: Same environment in dev, test, and production

### What is a Container?

A **container** is a lightweight, isolated environment for running applications.

**Container vs VM:**
```
Virtual Machines:
┌─────────────────────────────────┐
│  VM 1    │  VM 2    │  VM 3    │
│  App A   │  App B   │  App C   │
│  Guest OS│  Guest OS│  Guest OS│ ← Each has full OS
├─────────────────────────────────┤
│        Hypervisor               │
├─────────────────────────────────┤
│        Host OS                  │
├─────────────────────────────────┤
│        Hardware                 │
└─────────────────────────────────┘

Containers:
┌─────────────────────────────────┐
│Container1│Container2│Container3│
│  App A   │  App B   │  App C   │ ← Share host OS kernel
├─────────────────────────────────┤
│        Docker Engine            │
├─────────────────────────────────┤
│        Host OS                  │
├─────────────────────────────────┤
│        Hardware                 │
└─────────────────────────────────┘
```

**Key differences:**
- **VMs**: Include full OS (GBs), slower to start (minutes)
- **Containers**: Share host OS (MBs), fast to start (seconds)

**What's inside a container:**
```
Container:
├─ Your application code
├─ Dependencies (libraries, packages)
├─ Configuration files
└─ Runtime environment
```

### What is a Docker Image?

An **image** is a template for creating containers.

**Analogy:**
```
Image = Cookie cutter
Container = Cookie

One image can create many identical containers
```

**Image layers:**
```
┌─────────────────────────────┐
│ Your App Code               │ ← Layer 3
├─────────────────────────────┤
│ Python + Dependencies       │ ← Layer 2
├─────────────────────────────┤
│ Base OS (Ubuntu)            │ ← Layer 1
└─────────────────────────────┘
```

**Why layers matter:**
- Reusable (share common layers)
- Efficient (only download changes)
- Fast builds (cache unchanged layers)

**Example:**
```bash
# Pull an image from Docker Hub
docker pull nginx:latest

# Run a container from that image
docker run -d nginx:latest

# One image, multiple containers
docker run -d nginx:latest  # Container 1
docker run -d nginx:latest  # Container 2
```

### What is Container Orchestration?

**Orchestration** is automating the deployment, scaling, and management of containers across multiple servers.

**Without orchestration:**
```
You manually:
- Start containers on each server
- Monitor if they crash
- Restart failed containers
- Distribute traffic
- Scale up/down
```

**With orchestration (Docker Swarm, Kubernetes):**
```
You declare:
"I want 5 replicas of nginx"

Orchestrator automatically:
- Distributes containers across servers
- Monitors health
- Restarts failures
- Load balances traffic
- Scales as needed
```

**Popular orchestrators:**
- **Docker Swarm**: Simple, built into Docker
- **Kubernetes**: Complex, feature-rich, industry standard
- **Nomad**: Flexible, multi-workload

### What is High Availability (HA)?

**High availability** means your application stays running even when things fail.

**Without HA:**
```
Single Server:
├─ App running
└─ Server crashes → App is down ❌
```

**With HA:**
```
Multiple Servers:
├─ Server 1: App running
├─ Server 2: App running
└─ Server 1 crashes → Server 2 still serves traffic ✓
```

**How orchestrators provide HA:**
```
Desired state: 3 replicas of app

Current state:
Server A: 1 replica ✓
Server B: 1 replica ✓
Server C: 1 replica ✓

Server B fails:
Server A: 1 replica ✓
Server B: 0 replicas ❌
Server C: 1 replica ✓

Orchestrator detects and fixes:
Server A: 2 replicas ✓ ← Added one here
Server B: 0 replicas (down)
Server C: 1 replica ✓
```

### What is Load Balancing?

**Load balancing** distributes traffic across multiple servers/containers.

**Without load balancing:**
```
All requests → Server 1 (overloaded)
              Server 2 (idle)
              Server 3 (idle)
```

**With load balancing:**
```
Requests → Load Balancer
           ├─→ Server 1 (33% traffic)
           ├─→ Server 2 (33% traffic)
           └─→ Server 3 (33% traffic)
```

**Load balancing algorithms:**
- **Round-robin**: Request 1 → Server A, Request 2 → Server B, Request 3 → Server C, repeat
- **Least connections**: Send to server with fewest active connections
- **IP hash**: Same client always goes to same server

**Docker Swarm's load balancing:**
```
User request to any node:80
         ↓
   Ingress Network
         ↓
Automatically routes to one of:
├─ Container 1 on Node A
├─ Container 2 on Node B
└─ Container 3 on Node C
```

### What is an API (Application Programming Interface)?

An **API** is a way for programs to talk to each other.

**Analogy:**
```
Restaurant:
- Menu = API (what you can order)
- Waiter = API endpoint (takes your order)
- Kitchen = Backend (prepares food)

You don't go into the kitchen directly.
You use the menu and waiter (API).
```

**Docker API example:**
```
You: "docker run nginx"
     ↓
Docker CLI: Sends API request to Docker Engine
     ↓
Docker Engine API: Receives request
     ↓
Docker Engine: Creates container
     ↓
Docker CLI: Shows result to you
```

**Why APIs matter for Swarm:**
- Manager nodes expose Swarm API
- You interact via `docker` CLI
- CLI sends API requests to managers
- Managers coordinate the cluster

### What is a Replica?

A **replica** is a copy of your application running in a container.

**Single instance:**
```
1 container running nginx
If it crashes → service is down
```

**Multiple replicas:**
```
3 containers running nginx
If one crashes → 2 still running
```

**Declaring replicas:**
```bash
docker service create --replicas 5 nginx

Swarm creates:
├─ nginx.1 on Node A
├─ nginx.2 on Node B
├─ nginx.3 on Node C
├─ nginx.4 on Node A
└─ nginx.5 on Node B
```

### What is a Rolling Update?

A **rolling update** updates your application without downtime.

**Bad way (downtime):**
```
1. Stop all old containers
2. Start all new containers
   ↓
   During this time: Service is down ❌
```

**Rolling update (no downtime):**
```
Start: 3 containers running v1.0

Step 1: Start 1 container with v2.0
        2 containers v1.0 ✓
        1 container v2.0 ✓

Step 2: Stop 1 container v1.0
        1 container v1.0 ✓
        1 container v2.0 ✓

Step 3: Start another v2.0
        1 container v1.0 ✓
        2 containers v2.0 ✓

Step 4: Stop last v1.0
        3 containers v2.0 ✓

Service never went down!
```

**Docker Swarm rolling update:**
```bash
docker service update --image nginx:1.21 my-service

# Swarm automatically does rolling update
# Configurable: update parallelism, delay, failure action
```

### What is Declarative vs Imperative?

**Imperative** (step-by-step commands):
```bash
# You tell HOW to do it
docker run -d nginx
docker run -d nginx
docker run -d nginx
# If one dies, you must manually restart it
```

**Declarative** (desired state):
```bash
# You tell WHAT you want
docker service create --replicas 3 nginx

# Swarm maintains this state:
# - If container dies → automatically restarts
# - If node fails → moves containers to healthy nodes
# - You declare the goal, Swarm figures out how
```

**Benefits of declarative:**
- Self-healing (automatic recovery)
- Simpler (describe goal, not steps)
- Consistent (same result every time)

### What is Consensus?

**Consensus** is how multiple computers agree on a single truth.

**The problem:**
```
3 managers need to agree:
- Manager 1 thinks: "5 containers running"
- Manager 2 thinks: "4 containers running"
- Manager 3 thinks: "6 containers running"

Which is correct?
```

**Consensus algorithm (Raft):**
```
1. Managers elect a leader
2. Leader makes decisions
3. Leader tells followers
4. Followers confirm receipt
5. Once majority confirms → decision is final
```

**Why consensus matters:**
- Prevents split-brain (different managers disagree)
- Ensures cluster has single source of truth
- Allows cluster to survive failures

**Quorum:**
```
3 managers: Need 2 to agree (majority)
5 managers: Need 3 to agree (majority)

If you lose majority → cluster can't make new decisions
But existing containers keep running
```

---

## Part 1: What is Docker Swarm?

### The Simple Answer
> Docker Swarm turns multiple Docker hosts into a **single virtual Docker host** for running containers at scale.

### Why Use It?
**Without Swarm** (single host):
```
docker run -d nginx  # Runs on this machine only
```

**With Swarm** (cluster):
```
docker service create --replicas 5 nginx
# Runs 5 copies across multiple machines!
```

### Key Benefits
- **High availability**: If one host dies, containers move to others
- **Load balancing**: Traffic distributed across replicas
- **Declarative**: "I want 5 replicas" → Swarm maintains this
- **Rolling updates**: Update without downtime

---

## Part 2: Swarm Architecture

### Node Types

```
┌────────────────────────────────────────────────┐
│              Docker Swarm Cluster              │
│                                                │
│  ┌──────────────────────────────────────────┐  │
│  │         Manager Nodes                    │  │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  │  │
│  │  │Manager 1│  │Manager 2│  │Manager 3│  │  │
│  │  │ (Leader)│  │         │  │         │  │  │
│  │  └─────────┘  └─────────┘  └─────────┘  │  │
│  │       ↓ Raft Consensus ↓                 │  │
│  └──────────────────────────────────────────┘  │
│                     │                          │
│                     │ Schedules tasks          │
│                     ▼                          │
│  ┌──────────────────────────────────────────┐  │
│  │         Worker Nodes                     │  │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  │  │
│  │  │Worker 1 │  │Worker 2 │  │Worker 3 │  │  │
│  │  │         │  │         │  │         │  │  │
│  │  └─────────┘  └─────────┘  └─────────┘  │  │
│  └──────────────────────────────────────────┘  │
└────────────────────────────────────────────────┘
```

### Manager Nodes
**Responsibilities**:
- Maintain cluster state (Raft database)
- Schedule services to workers
- Serve Swarm API
- Elect a leader

**Can they run containers?** Yes! By default, managers also act as workers.

### Worker Nodes
**Responsibilities**:
- Execute tasks (containers) assigned by managers
- Report status back to managers

**Can they manage the cluster?** No, they're execution-only.

---

## Part 3: Raft Consensus Algorithm

### What is Raft?
Raft is a **consensus algorithm** that ensures all manager nodes agree on the cluster state.

### Why Do We Need It?
Imagine 3 managers:
- User creates a service on Manager 1
- Manager 2 needs to know about it
- Manager 3 also needs to know

**Raft ensures** all managers have the same view of reality.

### How Raft Works

#### 1. Leader Election
```
Initial State:
Manager 1: Follower
Manager 2: Follower
Manager 3: Follower

After Election:
Manager 1: Leader  ← Handles all writes
Manager 2: Follower
Manager 3: Follower
```

#### 2. Log Replication
```
User: docker service create nginx
         ↓
Manager 1 (Leader):
  1. Writes to local log: "Create nginx service"
  2. Sends to followers: "Replicate this entry"
  3. Waits for majority (2 out of 3) to confirm
  4. Commits the entry
  5. Responds to user: "Service created"
```

#### 3. Fault Tolerance
```
3 Managers: Can tolerate 1 failure (need 2 for quorum)
5 Managers: Can tolerate 2 failures (need 3 for quorum)
7 Managers: Can tolerate 3 failures (need 4 for quorum)
```

**Formula**: Can tolerate `(N-1)/2` failures, where N = total managers.

### Quorum
**Quorum** = Majority of managers must agree.

```
3 managers: Quorum = 2
5 managers: Quorum = 3
```

**If quorum is lost**:
- ❌ Can't create new services
- ❌ Can't update existing services
- ✅ Existing containers keep running

---

## Part 4: Services, Tasks, and Containers

### The Hierarchy
```
Service (nginx-web)
  ├─ Task 1 (nginx-web.1) → Container on Worker 1
  ├─ Task 2 (nginx-web.2) → Container on Worker 2
  └─ Task 3 (nginx-web.3) → Container on Worker 3
```

### Service
A **service** is a definition:
```yaml
Name: nginx-web
Image: nginx:latest
Replicas: 3
Ports: 80:80
```

### Task
A **task** is a slot for a container:
- `nginx-web.1` = First replica
- `nginx-web.2` = Second replica
- etc.

### Container
The **container** is the actual running instance.

### Example Flow
```bash
# Create service
docker service create --name nginx-web --replicas 3 nginx

# Swarm creates 3 tasks
# Scheduler assigns tasks to nodes
# Docker Engine on each node starts containers
```

---

## Part 5: Swarm Networking

### Required Ports
| Port | Protocol | Purpose |
|------|----------|---------|
| **2377** | TCP | Cluster management (Raft, API) |
| **7946** | TCP/UDP | Node discovery (Serf gossip) |
| **4789** | UDP | Overlay network traffic (VXLAN) |

### Port 2377: Cluster Management
```
Manager ←──────────────→ Manager (Raft consensus)
         Port 2377 TCP

Worker  ────────────────→ Manager (Join cluster)
         Port 2377 TCP
```

### Port 7946: Node Discovery
```
Node A ←────────────────→ Node B (Gossip protocol)
        Port 7946 TCP/UDP
```

Nodes exchange:
- Health status
- Network topology
- Service updates

### Port 4789: Overlay Traffic
```
Container A ────────────→ Container B (VXLAN tunnel)
            Port 4789 UDP
```

This is the **data plane** for container-to-container communication.

---

## Part 6: The `--advertise-addr` Flag

### What Does It Do?
Tells Swarm **which IP address** this node should use for cluster communication.

### Syntax
```bash
docker swarm init --advertise-addr <IP>
docker swarm join --advertise-addr <IP> <manager-ip>:2377
```

### Why It Matters
A host might have multiple network interfaces:
```
eth0: 192.168.1.10 (LAN)
wlan0: 192.168.0.50 (WiFi)
tailscale0: 100.64.0.1 (VPN)
```

**Without `--advertise-addr`**:
Docker picks one (might be the wrong one!).

**With `--advertise-addr`**:
You explicitly choose which interface to use.

### Your Tailscale Scenario
```bash
# You want Swarm to use Tailscale IP
docker swarm init --advertise-addr 100.64.0.1

# Docker tries to bind to 100.64.0.1
# But if that IP doesn't exist in Docker's namespace...
# Error: "cannot assign requested address"
```

---

## Part 7: Initializing a Swarm

### Single Manager (Development)
```bash
# On the manager node
docker swarm init --advertise-addr 192.168.1.10
```

Output:
```
Swarm initialized: current node (abc123) is now a manager.

To add a worker to this swarm, run:
    docker swarm join --token SWMTKN-1-xxx 192.168.1.10:2377

To add a manager to this swarm, run:
    docker swarm join-token manager
```

### Adding Workers
```bash
# On worker nodes
docker swarm join --token SWMTKN-1-xxx 192.168.1.10:2377
```

### Adding More Managers (Production)
```bash
# On the first manager, get the manager token
docker swarm join-token manager

# On new manager nodes
docker swarm join --token SWMTKN-1-yyy 192.168.1.10:2377
```

---

## Part 8: Creating Services

### Basic Service
```bash
docker service create \
  --name web \
  --replicas 3 \
  --publish 80:80 \
  nginx
```

### Check Service Status
```bash
docker service ls
docker service ps web
```

Output:
```
ID            NAME    IMAGE   NODE      DESIRED STATE  CURRENT STATE
abc123        web.1   nginx   worker1   Running        Running 1 min
def456        web.2   nginx   worker2   Running        Running 1 min
ghi789        web.3   nginx   manager1  Running        Running 1 min
```

### Update Service
```bash
# Scale up
docker service scale web=5

# Update image (rolling update)
docker service update --image nginx:alpine web
```

---

## Part 9: Overlay Networks in Swarm

### Create an Overlay Network
```bash
docker network create \
  --driver overlay \
  --attachable \
  my-network
```

### Attach Service to Network
```bash
docker service create \
  --name app \
  --network my-network \
  myapp:latest
```

### How It Works
```
Worker 1:
  Container app.1 (10.0.0.2)
       ↓ (overlay network)
       ↓ (VXLAN encapsulation)
       ↓
  Physical NIC (192.168.1.20)

       ↓ (Internet/LAN)

Worker 2:
  Physical NIC (192.168.1.30)
       ↓
       ↓ (VXLAN decapsulation)
       ↓
  Container app.2 (10.0.0.3)
```

Containers can communicate via `10.0.0.x` IPs, even though they're on different physical hosts!

---

## Part 10: Load Balancing

### Ingress Network
Swarm automatically creates an **ingress network** for published ports.

```bash
docker service create \
  --name web \
  --replicas 3 \
  --publish 80:80 \
  nginx
```

### How Ingress Works
```
User Request → Any Swarm Node:80
                    ↓
              Ingress Network
                    ↓
         Load balances to one of:
         - web.1 on Worker 1
         - web.2 on Worker 2
         - web.3 on Worker 3
```

**Key Point**: You can hit **any node** on port 80, even if it's not running a replica!

### Internal Load Balancing (VIP)
Services also get a **Virtual IP** for internal communication:
```bash
# Inside a container
curl http://web  # Resolves to VIP (e.g., 10.0.0.2)
                 # Load balances to one of the replicas
```

---

## Part 11: Why All Nodes Need Reachable IPs

### The Requirement
For Swarm to work:
1. **Managers must reach all nodes** (to schedule tasks)
2. **Nodes must reach managers** (to report status)
3. **Nodes must reach each other** (for overlay networks)

### Your Tailscale Setup
```
Manager (GCP VM):
  Tailscale IP: 100.64.0.1 ✓

Worker (Windows PC):
  Advertised IP: 172.X.X.Y (WSL2 internal)
  ❌ Manager can't reach this IP!
```

**Result**:
- Manager can't send tasks to worker
- Overlay networks fail (no VXLAN connectivity)

### The Fix
```
Manager (GCP VM):
  Tailscale IP: 100.64.0.1 ✓

Worker (Windows PC):
  Tailscale IP: 100.64.0.2 ✓ (installed in WSL2)
  Advertised: 100.64.0.2
```

Now both nodes can reach each other over Tailscale!

---

## Part 12: Hands-On: Your First Swarm

### Step 1: Initialize Swarm
```bash
# On manager
docker swarm init --advertise-addr <your-ip>
```

### Step 2: Join Workers
```bash
# On worker nodes
docker swarm join --token <token> <manager-ip>:2377
```

### Step 3: Verify Cluster
```bash
docker node ls
```

### Step 4: Deploy a Service
```bash
docker service create \
  --name hello \
  --replicas 3 \
  --publish 8080:80 \
  nginx
```

### Step 5: Test Load Balancing
```bash
# Hit any node on port 8080
curl http://<any-node-ip>:8080
```

### Step 6: Scale the Service
```bash
docker service scale hello=5
docker service ps hello
```

---

## Part 13: Troubleshooting

### Symptom: "cannot assign requested address"
**Cause**: Docker can't bind to the advertised IP.

**Fix**: Ensure the IP exists on a local interface:
```bash
ip addr | grep <advertised-ip>
```

### Symptom: Worker can't join
**Cause**: Port 2377 blocked.

**Fix**: Check firewall:
```bash
sudo ufw allow 2377/tcp
```

### Symptom: Containers can't communicate across hosts
**Cause**: Port 4789 blocked.

**Fix**:
```bash
sudo ufw allow 4789/udp
```

### Symptom: Quorum lost
**Cause**: Too many managers down.

**Fix**: Bring managers back online or force new cluster:
```bash
docker swarm init --force-new-cluster
```

---

---

## Common Misconceptions: Can I Run a Manager in a Container?

### The Question

"Can I run a Swarm Manager inside a container to avoid the binding issue?"

### The Answer: No

Here's why this won't work:

### 1. There is No Such Thing as a "Manager Container"

In Docker Swarm, the "Manager" is **not** a separate program you can run inside a container. The Manager is **built into the Docker Engine itself**.

```
┌─────────────────────────────────────┐
│         Your Computer               │
│                                     │
│  Docker Engine (dockerd)            │
│  ├─ Swarm Mode: ON                  │
│  │  └─ Role: Manager                │
│  │                                  │
│  └─ Manages Containers:             │
│      ├─ Container 1 (nginx)         │
│      ├─ Container 2 (redis)         │
│      └─ Container 3 (app)           │
└─────────────────────────────────────┘
```

**Key points:**
- **Docker Engine** = The software installed on your computer (`dockerd`)
- **Swarm Mode** = A setting inside that Engine
- **Manager** = The Engine itself when Swarm Mode is enabled

When you run `docker swarm init`, you are **flipping a switch** inside the main Engine software. You cannot install a "Manager App" inside a container to take over this job, because the Docker Engine is designed to be the boss of its own containers.

### 2. The "Relay" Problem (Swarm is a Mesh, not a Star)

You might think: "Can I create a manager container to relay all communication between PCs?"

This is how centralized networks (like old VPNs or Proxies) work, but **Swarm is different**.

**Swarm uses a Mesh Network:**

```
Star Topology (Centralized - NOT how Swarm works):
        ┌─────────┐
        │ Manager │
        └────┬────┘
      ┌──────┼──────┐
      │      │      │
   ┌──▼─┐ ┌─▼──┐ ┌─▼──┐
   │ W1 │ │ W2 │ │ W3 │
   └────┘ └────┘ └────┘

Mesh Topology (How Swarm actually works):
   ┌─────────┐
   │ Manager │
   └────┬────┘
     ┌──┼──┐
     │  │  │
   ┌─▼┐ │ ┌▼─┐
   │W1├─┼─┤W2│
   └─┬┘ │ └┬─┘
     │ ┌▼┐ │
     └─┤W3├─┘
       └──┘

Every node talks directly to every other node!
```

**What this means:**
- **Your PC (Worker)** needs to talk directly to **Other PC (Worker)**
- **Your PC** needs to talk directly to **GCP VM (Manager)**
- **GCP VM** needs to talk directly to **Your PC**

**If you somehow managed to put a "Manager" in a container on your PC:**
- The Swarm logic would still require the **Docker Engine** (host) to open the ports (2377, 4789, 7946) to allow that direct traffic
- The container cannot "intercept" traffic that the host refuses to touch
- We are back to the same problem: **The Host Engine refuses to bind to the IP**

### Summary

- **Can I run a Manager Container?** No. The Manager is the Engine itself.
- **Can I make the Manager relay traffic?** No. Swarm requires every node to talk directly to every other node (mesh topology).

### What CAN You Do?

You are essentially asking for a **Client-Server** architecture where your PC doesn't have to bind a special IP.

**Yes, you can do this, but not with a "Manager Container."**

You do it by making **GCP the only Manager**.

```
GCP VM: Manager (Handles all the swarm logic/binding)
Your PC: Client-only
```

But wait—your PC still needs to be a "Worker" to run containers.
- If you try to join your PC as a Worker, it hits the same binding problem (Overlay networks)

### The Only "No Local Bind" Solution

**Use your PC purely as a Remote Controller:**

```
┌─────────────────────────────────────┐
│         Your Windows PC             │
│                                     │
│  Docker CLI only                    │
│  (no Engine, no binding)            │
│                                     │
│  $ docker context use my-gcp-swarm  │
│  $ docker service create nginx      │
│         │                           │
└─────────┼───────────────────────────┘
          │
          │ (sends command over network)
          │
          ▼
┌─────────────────────────────────────┐
│         GCP VM                      │
│                                     │
│  Docker Engine (dockerd)            │
│  ├─ Swarm Mode: ON                  │
│  │  └─ Role: Manager                │
│  │                                  │
│  └─ Runs Containers:                │
│      ├─ nginx.1                     │
│      ├─ nginx.2                     │
│      └─ nginx.3                     │
└─────────────────────────────────────┘
```

**Steps:**
1. Run Swarm **only** on the GCP VMs
2. On your Windows PC, run:
   ```powershell
   docker context create my-gcp-swarm --docker "host=ssh://user@gcp-ip"
   docker context use my-gcp-swarm
   ```
3. When you type `docker service create ...`, it sends the command to GCP
4. GCP runs the container
5. Your PC runs nothing but the command line interface

**This is the only "No Local Bind" solution that works 100%.**

**Trade-offs:**
- ✅ No binding issues on your PC
- ✅ Simple setup
- ❌ Containers don't run on your PC (only on GCP)
- ❌ Need network connection to GCP to run commands

---

## Key Takeaways

1. **Swarm = Multi-host container orchestration**
2. **Managers use Raft** for consensus (need quorum)
3. **Services → Tasks → Containers** (hierarchy)
4. **Ports 2377, 7946, 4789** must be open
5. **`--advertise-addr`** specifies which IP to use
6. **All nodes need reachable IPs** for overlay networks

---

## Further Reading
- [Docker Swarm Overview](https://docs.docker.com/engine/swarm/)
- [Raft Consensus Explained](https://raft.github.io/)
- [Swarm Mode Key Concepts](https://docs.docker.com/engine/swarm/key-concepts/)
