# Protocol Failure Walkthroughs: Step-by-Step Analysis
## Observing How Each Docker Swarm Protocol Fails on Docker Desktop

**Document Purpose:** This companion document provides detailed step-by-step walkthroughs of exactly what happens when Docker Swarm protocols attempt to operate on Docker Desktop. Rather than theoretical analysis, this shows the precise sequence of events, packet flows, and failure points you would observe if you traced the system in real-time.

**Relationship to Main Study:** This document assumes you've read the comprehensive protocol study and understand the architectural components. Here, we trace specific scenarios from start to finish, showing the exact failure mechanisms.

---

## Table of Contents

1. [Walkthrough 1: Swarm Initialization Attempt](#walkthrough1)
2. [Walkthrough 2: Manager Join Sequence](#walkthrough2)
3. [Walkthrough 3: Raft Leader Election Failure](#walkthrough3)
4. [Walkthrough 4: SWIM Gossip Probe Cycle](#walkthrough4)
5. [Walkthrough 5: VXLAN Packet Journey](#walkthrough5)
6. [Walkthrough 6: Service Creation and Task Distribution](#walkthrough6)
7. [Walkthrough 7: Overlay Network Container Communication](#walkthrough7)
8. [Observable Symptoms and Diagnostics](#diagnostics)

---

## Walkthrough 1: Swarm Initialization Attempt {#walkthrough1}

Let us trace exactly what happens when you run `docker swarm init` on Docker Desktop and examine what succeeds and what sets the stage for future failures.

### Initial Command Execution

You open a terminal and run:
```bash
docker swarm init --advertise-addr 192.168.65.3
```

This command reaches the Docker CLI on your host machine. The Docker CLI communicates with the Docker Engine running inside the VM through a Unix domain socket or named pipe that is actually proxied through AF_VSOCK. The command successfully reaches Docker Engine because this VM-to-host communication is precisely what Docker Desktop's architecture supports.

### Inside the VM: Engine Processing

Docker Engine (dockerd) receives the swarm init command and begins the initialization sequence. The Engine checks that it is not already part of a swarm by examining the state in `/var/lib/docker/swarm`. Finding no existing swarm state, it proceeds.

The Engine validates the advertise address you provided (192.168.65.3). It performs a crucial check by calling `ip addr show` internally to verify that this address exists on a network interface. Inside the VM, when we examine the interfaces, we find:

```
eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
    inet 192.168.65.3/24 brd 192.168.65.255 scope global eth0
```

The address exists! Docker Engine accepts it as valid. This is the first critical point where the architecture's limitations are not yet apparent. From the VM's perspective, everything looks normal. The VM has a network interface with an IP address, a default gateway, and working DNS. The VM has no way to know that its network is isolated.

### Raft Initialization

Docker Engine initializes the Raft consensus system. It creates a new Raft instance and assigns itself Raft ID 1 (the first manager always gets ID 1). The Engine writes this information to `/var/lib/docker/swarm/raft/wal`, creating a Write-Ahead Log for the Raft protocol.

The Raft log contains the initial cluster configuration entry that records:
- Node ID: randomly generated cryptographic identifier derived from TLS certificate
- Raft ID: 1
- Address: 192.168.65.3:2377
- Role: Leader

Because this is a single-node cluster, there are no other members to contact for consensus. The node immediately becomes the Raft leader because it constitutes a majority (1 out of 1 nodes). No network communication is required yet.

### SWIM Gossip Initialization

Docker Engine also initializes the gossip protocol using the memberlist library. This creates a SWIM instance that will manage membership information. For a single-node cluster, the memberlist contains only the local node.

The gossip system binds to port 7946 on the VM's IP address. We can verify this:

```bash
# Inside the VM
netstat -uln | grep 7946
udp        0      0 192.168.65.3:7946    0.0.0.0:*
```

The port is successfully bound because binding to the local IP address within the VM works perfectly. The VM's network stack accepts the bind() call because the address exists on eth0.

### Port Binding Success

Docker Engine also binds to port 2377 for Raft communication:

```bash
# Inside the VM
netstat -tln | grep 2377
tcp        0      0 192.168.65.3:2377    0.0.0.0:*    LISTEN
```

Again, this succeeds because we're operating within the VM's network namespace. The port is listening and ready to accept connections.

### TLS Certificate Generation

Docker Engine generates TLS certificates for secure cluster communication. A root CA is created, and a key pair is generated for this first node. The certificates are stored in `/var/lib/docker/swarm/certificates/`.

These certificates will be used to encrypt Raft logs and authenticate connections between nodes. The TLS infrastructure is fully functional because it's purely local cryptographic operations that don't depend on network architecture.

### Join Token Generation

Docker Engine generates join tokens for workers and managers. These tokens contain encrypted information that allows new nodes to authenticate and join the swarm. The token format looks like:

```
SWMTKN-1-3xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx-yyyyyyyyyyyyyyyyyyyyyy
```

The token embeds the address where nodes should connect (192.168.65.3:2377). This embedded address is the first sign of trouble, though it's not yet apparent. The token is valid and correctly formatted, but it contains an address that only makes sense within one machine's VM.

### Success Message

Docker CLI reports success:

```
Swarm initialized: current node (abc123...) is now a manager.

To add a worker to this swarm, run the following command:
    docker swarm join --token SWMTKN-1-... 192.168.65.3:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

From the user's perspective, everything worked. The swarm is initialized. The commands all succeeded. There were no errors. This is because everything that happened occurred within a single machine. No cross-machine communication was attempted yet, so no failures occurred.

### What Actually Succeeded

Let's be precise about what actually works at this point:

1. **Raft is operational** - The Raft instance is running and maintaining consensus, trivially, because there's only one node
2. **Gossip is operational** - The SWIM instance is running and tracking membership, which contains only the local node
3. **Ports are bound** - Ports 2377 and 7946 are listening on the VM's IP address
4. **TLS is configured** - Certificates exist and encryption is ready
5. **Cluster state exists** - The swarm has cluster state that can be queried with `docker info` and `docker node ls`

All of these operations are local to the VM. They involve no cross-machine communication. Docker Desktop's architecture handles local operations perfectly well.

### What Will Fail

The problems only manifest when you try to add a second node from a different machine. At that point:

- The second machine will try to connect to 192.168.65.3:2377
- That connection will fail because 192.168.65.3 is not routable from the second machine
- Even if both machines are on the same LAN, ARP resolution will fail
- The join operation will timeout and fail

But for now, with a single node, everything appears to work. This is why single-node Swarm on Docker Desktop is functional—it never exercises the cross-machine communication paths that are broken.

---

## Walkthrough 2: Manager Join Sequence {#walkthrough2}

Now let's trace what happens when you try to add a second manager node to the swarm from a different machine running Docker Desktop. This is where the architectural limitations become visible as concrete failures.

### The Setup

We have two machines:
- **Machine A** (IP 192.168.1.100): Running Docker Desktop, initialized swarm with advertise address 192.168.65.3
- **Machine B** (IP 192.168.1.101): Running Docker Desktop, about to join as manager

Both machines are on the same physical LAN (192.168.1.0/24 network). They can ping each other's physical IPs successfully. From the user's perspective, there should be no connectivity problem.

### Obtaining the Join Token

On Machine A, we run:
```bash
docker swarm join-token manager
```

This outputs:
```
To add a manager to this swarm, run the following command:
    docker swarm join --token SWMTKN-1-3xxxxx-yyyyyy 192.168.65.3:2377
```

We copy this command to run on Machine B. Note the address: 192.168.65.3:2377. This is Machine A's VM IP address, not Machine A's physical IP address.

### Join Command Execution on Machine B

On Machine B, we run:
```bash
docker swarm join --token SWMTKN-1-3xxxxx-yyyyyy 192.168.65.3:2377
```

The Docker CLI on Machine B's host sends this command through AF_VSOCK to Docker Engine inside Machine B's VM. The Engine receives the command and begins the join process.

### Token Parsing and Validation

Docker Engine on Machine B parses the join token. The token contains:
- Encrypted cluster ID
- Secret for authentication
- Manager address: 192.168.65.3:2377

The Engine validates the token format and extracts the manager address. No errors occur yet because token parsing is a purely local operation.

### Initial Connection Attempt

Docker Engine now attempts to establish a TCP connection to 192.168.65.3:2377. Inside Machine B's VM, the kernel executes the connect() system call. Let's trace this packet's journey in detail.

**Step 1: Routing Decision**

The VM kernel looks up the route to 192.168.65.3:

```bash
# Inside Machine B's VM
ip route get 192.168.65.3
```

This returns:
```
192.168.65.3 via 192.168.65.1 dev eth0 src 192.168.65.3
```

The kernel determines that the destination is on the local network (192.168.65.0/24) but requires routing through the gateway (192.168.65.1). Interestingly, the source address is also 192.168.65.3 because both Machine A's VM and Machine B's VM have been assigned the same IP address by their respective VPNKit instances.

**Step 2: Packet Construction**

The kernel constructs a TCP SYN packet:
```
IP Header:
  Source IP: 192.168.65.3
  Destination IP: 192.168.65.3
  Protocol: TCP

TCP Header:
  Source Port: 45678 (ephemeral port)
  Destination Port: 2377
  Flags: SYN
  Sequence Number: (random)
```

This packet is passed to the eth0 interface for transmission.

**Step 3: eth0 Driver Processing**

The eth0 network interface in the VM is not a traditional network device. It's a virtio-vsock device. Instead of sending the packet to a virtual NIC driver that would communicate with a software bridge, eth0 passes the entire Ethernet frame to the AF_VSOCK mechanism.

The frame, including Ethernet header, IP header, and TCP header, is serialized and written to the shared memory buffer that AF_VSOCK uses for VM-to-host communication.

**Step 4: Arrival at VPNKit**

On Machine B's host, VPNKit receives the Ethernet frame through AF_VSOCK. VPNKit's packet processing pipeline begins:

```
VPNKit Packet Processor:
1. Parse Ethernet header → EtherType: IPv4
2. Parse IP header → Source: 192.168.65.3, Dest: 192.168.65.3
3. Parse TCP header → Dest Port: 2377, Flags: SYN
4. Routing decision for destination 192.168.65.3
```

VPNKit's routing logic examines the destination address. It has several categories of addresses it knows how to handle:

- **Internet addresses** (not in RFC 1918 private ranges): Create outbound connection from host
- **VPNKit internal services** (192.168.65.1): Handle internally (DNS, DHCP)
- **Broadcast addresses**: Handle special broadcast logic
- **Everything else**: No handler

The destination 192.168.65.3 is not an internet address (it's RFC 1918 private). It's not 192.168.65.1 (the VPNKit gateway). It's not a broadcast. VPNKit has no handler for this destination.

**Step 5: Packet Drop**

VPNKit drops the packet. No connection is created on the host. No error message is generated back to the VM because VPNKit has no mechanism to send ICMP unreachable messages for dropped packets. The packet simply disappears.

### TCP Retransmission and Timeout

Back in Machine B's VM, the kernel's TCP stack is waiting for a SYN-ACK response to the SYN packet it sent. The initial retransmission timer is set (typically 1 second for the first retransmission).

After 1 second with no response, the kernel retransmits the SYN packet. The packet follows the exact same path and is dropped again by VPNKit. The retransmission timer doubles (exponential backoff): 2 seconds, 4 seconds, 8 seconds, 16 seconds, and so on.

After approximately 127 seconds (the default TCP SYN retransmission timeout on Linux), the kernel gives up. The connect() system call that Docker Engine made returns an error:

```
Error: ETIMEDOUT (Connection timed out)
```

### Error Propagation

Docker Engine receives the timeout error. It recognizes that it could not connect to the manager. The Engine returns an error to the Docker CLI:

```
Error response from daemon: Timeout was reached before node joined. The attempt to join the swarm will continue in the background.
```

Actually, Docker has a retry mechanism. The Engine doesn't give up after one timeout. It waits a bit and tries again. And again. Each attempt follows the same path: SYN packet → eth0 → AF_VSOCK → VPNKit → dropped → timeout.

### User Experience

From the user's perspective, the command hangs for a couple of minutes, then fails with a timeout error. If you try to be helpful and use Machine A's physical IP instead:

```bash
docker swarm join --token SWMTKN-1-3xxxxx-yyyyyy 192.168.1.100:2377
```

This also fails, but for a different reason. Now the packet goes through VPNKit, which recognizes 192.168.1.100 as being on the host's physical network. VPNKit attempts to create a connection from Machine B's host to 192.168.1.100:2377.

This connection attempt reaches Machine A's physical network interface. But nothing is listening on port 2377 on Machine A's host. Docker Engine is listening on port 2377, but inside Machine A's VM, not on the host. The host's kernel responds with a TCP RST (reset) packet, indicating that no service is listening on that port.

Machine B receives the RST and the connection fails immediately with:

```
Error: ECONNREFUSED (Connection refused)
```

This is a faster failure but still a failure. Either way—using the VM IP or the host IP—the join operation cannot succeed.

### Why This Failure Is Fundamental

This failure illustrates the core architectural problem. Docker Swarm needs to connect to the advertised address. The advertised address must be:
1. An address that Docker Engine can bind to (must exist in VM)
2. An address that other machines can connect to (must be routable from outside)

These two requirements are mutually exclusive in Docker Desktop's architecture:
- Addresses in the VM (like 192.168.65.3) satisfy requirement 1 but not requirement 2
- Addresses on the host (like 192.168.1.100) satisfy requirement 2 but not requirement 1

There is no address that satisfies both requirements simultaneously. This is not a configuration problem that can be fixed. It's an architectural impossibility.

---

## Walkthrough 3: Raft Leader Election Failure {#walkthrough3}

Let's examine what would happen if we could somehow get past the connection problem and multiple managers were actually running. We'll trace through a leader election scenario to show how the Raft protocol would fail even with connectivity.

### Hypothetical Setup

Imagine we've magically solved the connection problem (perhaps through elaborate VPN tunnels and port forwarding). We have three managers:
- Manager A at 192.168.1.100 (physical IP), 192.168.65.3 (VM IP)
- Manager B at 192.168.1.101 (physical IP), 192.168.65.3 (VM IP)
- Manager C at 192.168.1.102 (physical IP), 192.168.65.3 (VM IP)

Manager A is currently the Raft leader. All three VMs have the same IP address (192.168.65.3) because VPNKit assigns addresses independently.

### Normal Operation

Under normal operation, Manager A (the leader) sends heartbeat messages to Managers B and C every 50 milliseconds (configurable). These are empty AppendEntries RPC calls that serve only to assert leadership.

Each heartbeat message contains:
- Leader's term number (increases with each election)
- Leader's ID
- Log indices for consistency checking

Managers B and C respond with acknowledgments. As long as heartbeats arrive, B and C remain followers and do not attempt to become leaders.

### Leader Failure Trigger

Now suppose Manager A crashes completely. Its process dies, the VM might still be running but Docker Engine is gone. The heartbeat messages stop.

### Follower Timeout

Manager B has an election timeout configured (typically 150-300 milliseconds, randomized to prevent simultaneous elections). When Manager B doesn't receive a heartbeat within this timeout, it increments its term number and transitions to the candidate state.

### Election Initiation

Manager B as a candidate needs to:
1. Vote for itself
2. Send RequestVote RPCs to Manager A and Manager C
3. Wait for responses
4. Become leader if it receives a majority of votes (2 out of 3)

Let's trace the RequestVote RPC that Manager B sends to Manager C.

### RequestVote Message Construction

The RequestVote RPC contains:
```
Term: 5 (incremented from 4)
CandidateID: manager-B-node-id
LastLogIndex: 123
LastLogTerm: 4
```

This message needs to be sent to Manager C. But what's Manager C's address?

### The Address Ambiguity Problem

In Raft's internal state, Manager B has a mapping:
```
Manager C → Address: 192.168.65.3:2377
```

Wait—this is a problem. Manager B's own address is also 192.168.65.3. When Manager B tries to send a RequestVote to 192.168.65.3:2377, the packet is routed locally via loopback. Manager B ends up trying to connect to itself!

Let's trace what happens. Manager B's TCP stack sees a connection attempt to 192.168.65.3:2377. It checks: "Is this a local address?" Yes, it's configured on eth0. The connection is established via loopback, not via the network.

Manager B connects to port 2377 on its own machine. Docker Engine on Manager B receives an incoming connection on its Raft port. The connection appears to be from itself (source IP is 192.168.65.3). Docker Engine is confused—why is it connecting to itself?

The TLS handshake begins. Manager B presents its certificate to authenticate the connection. Manager B receives this certificate and examines it. The certificate's identity matches Manager B's own identity. Docker Engine recognizes that this is itself and rejects the connection as invalid.

The connection fails. Manager B does not receive a vote from Manager C because the message never reached Manager C. It reached Manager B itself.

### Port Forwarding Workaround Attempt

Someone might suggest: "Use port forwarding to expose each manager's port 2377 on the host, then use the physical IPs!"

Suppose we try this. We configure port forwarding so that:
- Machine A: host port 2377 → VM port 2377
- Machine B: host port 2377 → VM port 2377
- Machine C: host port 2377 → VM port 2377

Now, Raft's address mappings use physical IPs:
```
Manager A → 192.168.1.100:2377
Manager B → 192.168.1.101:2377
Manager C → 192.168.1.102:2377
```

Let's trace the RequestVote from Manager B to Manager C through this port forwarding setup.

**Step 1:** Manager B's Docker Engine calls connect() to 192.168.1.102:2377

**Step 2:** The VM kernel sees this as an outbound connection to an internet address (not 192.168.65.0/24). The packet goes through eth0 → AF_VSOCK → VPNKit

**Step 3:** VPNKit sees destination 192.168.1.102:2377. This is on the physical network! VPNKit creates a socket on Manager B's host and connects to 192.168.1.102:2377

**Step 4:** The connection reaches Machine C's host. The host's Docker Desktop backend has a listener on port 2377 due to port forwarding. It accepts the connection

**Step 5:** The backend establishes a connection through AF_VSOCK to Manager C's VM, to port 2377 inside the VM

**Step 6:** Docker Engine on Manager C receives an incoming Raft connection

So far, this works! The connection is established. But now the critical problem emerges.

### Source IP Loss

When Manager C's Docker Engine checks the source of the connection (by examining the socket peer address), what does it see?

It sees the connection as coming from the VM's internal network, typically from the docker_gwbridge or from 192.168.65.1. The port forwarding proxy has broken the connection into two separate connections, and the source IP from the original connection (Manager B's VM IP) is lost.

Manager C's Raft implementation tries to identify which manager sent this RequestVote. It looks at the source IP and tries to match it to a manager in its configuration. The source IP doesn't match any manager. All managers are configured with their VM IPs (192.168.65.3) or physical IPs (192.168.1.x), but the actual source IP is the proxy's internal address.

Raft requires knowing which node sent each message to track votes, log replication progress, and membership. Without accurate source IP, it cannot function correctly.

Additionally, the RequestVote message itself contains CandidateID (Manager B's identifier). Manager C could theoretically use this ID instead of source IP. But other Raft messages like heartbeats and AppendEntries are used to track which nodes are up to date on log replication. For these, Raft maintains per-follower state indexed by node address. The architecture breaks down when addresses don't match network reality.

### Vote Response Complexity

Even if Manager C sends a vote response back to Manager B, the response travels through the reverse path: VM → AF_VSOCK → proxy → network → proxy → AF_VSOCK → VM. The latency through this path is unpredictable and likely exceeds Raft's timeout thresholds.

Raft's election timeouts are typically 150-300 milliseconds. Adding two proxy layers, multiple context switches, and network traversal could easily exceed this, causing spurious timeouts and repeated elections.

### Split Brain Scenario

The most dangerous failure mode is split brain. If network partitions cause some managers to lose contact with the leader but can still communicate with each other through proxy chains, they might elect a new leader while the old leader is still running. Now there are two leaders, both believing they have authority to commit log entries.

Raft's design prevents split brain through its quorum requirement—a leader must have received votes from a majority. But if source IP confusion causes managers to misidentify each other, or if proxy timeouts cause false detection of failed nodes, the quorum mechanism breaks down.

This scenario would result in cluster state divergence, corrupted logs, and potentially data loss in any services managed by the swarm. This is catastrophic for a distributed system.

---

## Walkthrough 4: SWIM Gossip Probe Cycle {#walkthrough4}

The SWIM gossip protocol operates continuously in the background, independent of Raft. Let's trace a complete probe cycle to see how it fails on Docker Desktop.

### Normal Probe Cycle

Every 200 milliseconds (configurable), each node in the gossip network randomly selects another node to probe. The probe is a simple UDP message: "Are you alive?"

### Probe Initiation

Manager A decides to probe Manager B. Manager A has Manager B in its memberlist with the address 192.168.65.3:7946 (remember, all VMs have the same IP).

Manager A constructs a ping message:
```
Message Type: PING
Sequence Number: 42
Node: manager-A-node-id
Incarnation: 5
```

This message is serialized and sent as a UDP datagram.

### UDP Transmission Attempt

The UDP stack in Manager A's VM creates a datagram:
```
IP Header:
  Source: 192.168.65.3
  Dest: 192.168.65.3
  Protocol: UDP

UDP Header:
  Source Port: 7946
  Dest Port: 7946

Data:
  [SWIM ping message]
```

This datagram is passed to eth0 for transmission.

### The Loopback Problem

Just as with the Raft scenario, the kernel sees that the destination (192.168.65.3) is a local address. The UDP datagram is delivered locally via loopback.

Manager A receives its own ping message. The SWIM protocol processes this and is confused—why did it receive a ping from itself? The message is discarded as invalid.

Manager A's probe of Manager B times out after approximately 500 milliseconds. Manager A marks Manager B as potentially failed and initiates an indirect probe.

### Indirect Probe Sequence

For the indirect probe, Manager A selects random nodes (let's say Manager C and Manager D, hypothetically) and sends them indirect probe requests: "Can you ping Manager B for me?"

These indirect probe requests follow the same path as the direct probe. They are sent to 192.168.65.3 because that's Manager C's and Manager D's address. They are delivered locally via loopback. Manager A receives its own indirect probe requests.

This recursive failure continues. Manager A cannot probe Manager B directly, cannot probe other managers to ask them to probe Manager B, and cannot receive probes from other managers.

### With Physical IP Routing

Let's imagine we somehow route via physical IPs instead. Manager A tries to probe Manager B at 192.168.1.101:7946.

The UDP datagram leaves Manager A's VM via eth0 → AF_VSOCK → VPNKit. VPNKit sees destination 192.168.1.101:7946. This is on the physical network, so VPNKit creates a UDP socket on Manager A's host.

But here's a critical difference from TCP: VPNKit's UDP handling is designed for request-response protocols like DNS. It sends the datagram, then waits for a response to the same socket. For SWIM, the "response" (ACK) might not arrive quickly or might come from a different port or address depending on NAT behavior.

More critically, the datagram reaches Manager B's host at 192.168.1.101:7946. But nothing on the host is listening on UDP port 7946. The gossip protocol is listening on port 7946 inside the VM, not on the host.

The host's kernel checks: "Any sockets bound to 0.0.0.0:7946 or 192.168.1.101:7946?" No. The kernel might send an ICMP port unreachable message back, or it might just drop the packet depending on firewall configuration.

Either way, the probe doesn't reach the actual gossip process inside Manager B's VM. The probe times out.

### Unsolicited Inbound UDP

Even if we configure port forwarding for UDP port 7946, there's another problem. Gossip requires unsolicited inbound UDP packets.

When Manager B's gossip process probes Manager A, that probe packet must be able to reach Manager A without Manager A having initiated anything. This is fundamentally different from the request-response pattern that VPNKit and port forwarding support.

For TCP port forwarding, external connections arrive at the host's listener, which then establishes connections to the VM. For UDP, there's no clear "connection" to forward. The port forwarding system would need to:
1. Listen for any UDP packet to port 7946 on the host
2. Determine which VM it should go to (if multiple VMs existed)
3. Forward it through AF_VSOCK
4. Track the session so responses can be routed back

This is complex and fragile, and Docker Desktop doesn't implement it for UDP because its use case (local development) doesn't require it.

### Failure Detection Cascade

When probes fail consistently, the SWIM protocol marks nodes as suspicious. After a timeout period, suspicious nodes are marked as dead if they don't refute the suspicion.

In a Docker Desktop multi-manager scenario, all managers would suspect all other managers of being dead because probes never succeed. The memberlist would converge to a state where each manager believes it's the only alive node.

This breaks cluster operations. Managers need gossip for distributing information like which nodes are running which tasks, network membership, and load balancing metadata. Without working gossip, even if Raft somehow functioned, the data plane would be broken.

### Observable Symptoms

If you could somehow observe the gossip layer (it's not exposed in normal Docker CLI), you would see:
- Constant probe timeouts
- All nodes marked as suspicious or dead
- Failed indirect probe attempts
- High churn in the memberlist
- Repeated attempts to refute false suspicions

The logs would be filled with warnings like:
```
[WARN] memberlist: Failed to receive ack from node manager-B
[WARN] memberlist: Suspect node manager-B has failed
[WARN] memberlist: Node manager-C is suspicious
```

These warnings indicate that the gossip protocol is attempting to function but cannot due to network unreachability.

---

## Walkthrough 5: VXLAN Packet Journey {#walkthrough5}

Now let's trace the most complex scenario: a VXLAN packet carrying actual application traffic between containers on different hosts. This shows how the data plane itself is broken, even if we could somehow fix the control plane.

### The Scenario

We have:
- Machine A: Container C1 (IP 10.0.1.5 on overlay network "app-net")
- Machine B: Container C2 (IP 10.0.1.6 on overlay network "app-net")
- Overlay network "app-net" uses VNI 4097
- Docker has created VXLAN interfaces on both hosts

Container C1 wants to ping Container C2.

### Application Layer

Inside Container C1, an application runs:
```bash
ping 10.0.1.6
```

The ping utility creates an ICMP echo request packet and sends it through the socket.

### Container Network Namespace

Container C1 has its own network namespace. Inside this namespace, the routing table shows:
```
10.0.1.0/24 dev eth0 scope link
```

The kernel routes the ICMP packet to eth0 inside the container's network namespace. This eth0 is one end of a veth pair.

### Bridge and VXLAN Interface

The other end of the veth pair is attached to a Linux bridge. Also attached to this bridge is a VXLAN interface. Let's examine the VXLAN interface in detail:

```bash
# On Manager A's VM
ip -d link show vxlan1
5: vxlan1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450
    link/ether 02:42:0a:00:01:01 brd ff:ff:ff:ff:ff:ff
    vxlan id 4097 remote 192.168.65.3 dev eth0 dstport 4789
```

This shows:
- VNI: 4097 (identifies the overlay network)
- Remote: 192.168.65.3 (the other VTEP, which should be Manager B but is actually the same IP)
- Device: eth0 (the VXLAN tunnel uses eth0 as its underlay)
- Dest Port: 4789 (standard VXLAN port)

### Forwarding Database Lookup

The kernel's VXLAN module needs to determine where to send this packet. It performs an FDB lookup:

```bash
bridge fdb show dev vxlan1
```

This might show:
```
02:42:0a:00:01:06 dst 192.168.65.3 self permanent
```

This entry says: "To reach MAC address 02:42:0a:00:01:06 (Container C2), send to VTEP at 192.168.65.3."

The FDB entry exists because Docker's control plane distributed this information via the gossip protocol (if the gossip protocol were working). In reality, the entry might not exist because the gossip failed, but let's assume it does for this walkthrough.

### VXLAN Encapsulation

The kernel's VXLAN module encapsulates the original packet. The encapsulation process proceeds layer by layer:

**Original Packet (from Container C1):**
```
Ethernet: src=02:42:0a:00:01:05, dst=02:42:0a:00:01:06, type=IPv4
IPv4: src=10.0.1.5, dst=10.0.1.6, protocol=ICMP
ICMP: type=echo-request, id=1, seq=1
```

**After VXLAN Encapsulation:**
```
Outer Ethernet: src=<VM's MAC>, dst=<gateway MAC>, type=IPv4
Outer IPv4: src=192.168.65.3, dst=192.168.65.3, protocol=UDP
Outer UDP: src=45678, dst=4789
VXLAN Header: flags=0x08 (I flag set), VNI=4097, reserved=0
Inner Ethernet: src=02:42:0a:00:01:05, dst=02:42:0a:00:01:06, type=IPv4
Inner IPv4: src=10.0.1.5, dst=10.0.1.6, protocol=ICMP
Inner ICMP: type=echo-request, id=1, seq=1
```

The encapsulated packet is now a UDP datagram ready to be sent to the remote VTEP at 192.168.65.3 (which is problematically the same as the source IP).

### Transmission Attempt

The encapsulated packet is passed to eth0 for transmission. The kernel's routing table is consulted:

```bash
ip route get 192.168.65.3
192.168.65.3 via 192.168.65.1 dev eth0 src 192.168.65.3
```

The packet is sent toward the gateway (192.168.65.1, which is VPNKit) via eth0.

### eth0 and AF_VSOCK

The virtio-vsock driver on eth0 serializes the entire outer Ethernet frame and transmits it through AF_VSOCK to VPNKit on the host.

### VPNKit Processing

VPNKit receives the frame and begins parsing:

```
Layer 1 (Ethernet): Parse OK
  Dest MAC: <gateway MAC>
  Src MAC: <VM MAC>
  EtherType: 0x0800 (IPv4)

Layer 2 (IP): Parse OK
  Src IP: 192.168.65.3
  Dst IP: 192.168.65.3
  Protocol: 17 (UDP)

Layer 3 (UDP): Parse OK
  Src Port: 45678
  Dst Port: 4789

Decision: What to do with this packet?
```

VPNKit's routing logic examines the destination. The destination 192.168.65.3 is not an internet address. It's not a VPNKit internal service address. VPNKit has no handler for this.

### Packet Drop and Black Hole

VPNKit drops the packet. The VXLAN encapsulated packet containing the ping request is discarded. No error is generated. From the kernel's perspective, the packet was successfully sent out eth0. There's no indication of failure.

### Application Timeout

Back in Container C1, the ping utility waits for a reply. The ICMP echo request timeout is typically 1 second. After 1 second, no reply has arrived. The ping utility reports:

```
64 bytes from 10.0.1.6: icmp_seq=1 timeout
```

Actually, wait—the ping utility doesn't even receive a timeout message. It simply never gets a reply. After the configured timeout, it gives up and reports the packet as lost.

The user sees:
```
PING 10.0.1.6 (10.0.1.6): 56 data bytes

--- 10.0.1.6 ping statistics ---
4 packets transmitted, 0 packets received, 100% packet loss
```

### Why Using Physical IPs Doesn't Help

Someone might suggest: "Configure the VXLAN to use physical IPs!" Let's trace what happens.

We somehow reconfigure the VXLAN interface to use 192.168.1.101 (Machine B's physical IP) as the remote VTEP:

```bash
ip link add vxlan1 type vxlan id 4097 remote 192.168.1.101 dev eth0 dstport 4789
```

Now when a packet needs to go to Container C2, the VXLAN module encapsulates with:
```
Outer IPv4: src=192.168.65.3, dst=192.168.1.101
```

This packet goes through eth0 → AF_VSOCK → VPNKit.

VPNKit sees destination 192.168.1.101:4789. This is on the physical network! VPNKit creates a UDP socket on Machine A's host and sends the datagram to 192.168.1.101:4789.

The datagram reaches Machine B's host. But nothing on Machine B's host is listening on UDP port 4789. The VXLAN interface is inside the VM, not on the host.

Machine B's host kernel receives a UDP packet for port 4789. It checks for listeners. Finding none, it might:
- Send an ICMP port unreachable message back
- Drop the packet silently (if a firewall blocks ICMP)

Either way, the VXLAN packet doesn't reach the VXLAN interface inside Machine B's VM.

### Port Forwarding for UDP 4789

Could we forward UDP port 4789? This has several problems:

First, VXLAN is high-rate bidirectional traffic. Port forwarding systems are designed for occasional connections, not constant high-rate flows.

Second, when a packet arrives at Machine B's host on UDP port 4789, the port forwarding system would forward it to the VM. But which VM? If there were multiple VMs (which is not the case currently but theoretically possible), there's ambiguity.

Third, and most critically, FDB learning breaks. When Machine B's VXLAN interface receives the forwarded packet, it examines the outer source IP to learn where Container C1's MAC is located. But the outer source IP has been changed by the forwarding process. It might show Machine A's host IP (192.168.1.100) instead of Machine A's VM IP (192.168.65.3).

Machine B's FDB learns: "MAC 02:42:0a:00:01:05 is at VTEP 192.168.1.100"

When Container C2 tries to send a reply back to Container C1, the VXLAN module on Machine B consults the FDB, sees that it should send to 192.168.1.100, and creates a VXLAN packet with outer destination IP 192.168.1.100.

But 192.168.1.100 is Machine A's host, not a VTEP. Machine A's host doesn't have a VXLAN interface. The packet arrives and is dropped.

The FDB learning breaks because VXLAN requires the outer source IP to be the actual VTEP IP, not a proxy's IP.

### The Kernel Performance Argument

Even if we solved all the addressing problems, there's a performance concern. VXLAN packet processing must happen in kernel space for production performance.

In the broken port-forwarding scenario, each VXLAN packet would:
1. Be created by kernel VXLAN module (kernel space)
2. Exit eth0 → cross to userspace (VPNKit)
3. VPNKit parses and makes forwarding decision (userspace)
4. VPNKit creates a UDP socket and sends (userspace → kernel)
5. Packet travels through host network (kernel)
6. Arrives at destination host (kernel)
7. Forwarded to port forwarding system (might involve userspace)
8. Sent through AF_VSOCK to VM (crosses boundary)
9. Received by VXLAN module in VM (kernel)

This involves multiple kernel-userspace boundaries and much more overhead than direct kernel-to-kernel VXLAN, which would be:
1. Created by kernel VXLAN module
2. Sent through physical NIC driver
3. Travels through network
4. Received by physical NIC driver
5. Processed by kernel VXLAN module

The difference is 2 steps vs 9+ steps, and more importantly, kernel-only processing vs multiple userspace transitions. At scale, this performance difference matters enormously.

---

## Walkthrough 6: Service Creation and Task Distribution {#walkthrough6}

Let's trace what happens when you try to create a service in a hypothetical multi-manager swarm and see where task distribution fails.

### Service Creation Command

On a manager node (Machine A), you run:
```bash
docker service create --name web --replicas 3 --network app-net -p 8080:80 nginx
```

This command creates a service definition that Docker needs to distribute to workers.

### Manager Processing

Docker Engine on Machine A receives the service creation request. The manager:
1. Validates the service definition
2. Creates a service object in the cluster state
3. Assigns a virtual IP to the service for service discovery
4. Creates a Raft log entry for this new service

### Raft Log Replication

The service creation is a state change that must be replicated via Raft. Machine A (the leader) creates a log entry:

```
Entry Index: 234
Term: 5
Operation: CreateService
Data: {
  Name: "web",
  Image: "nginx",
  Replicas: 3,
  Networks: ["app-net"],
  PublishedPorts: [8080:80]
}
```

The leader needs to replicate this entry to follower managers. It sends AppendEntries RPCs to each follower. As we've traced before, these RPCs fail due to network unreachability. The leader waits for acknowledgments that never arrive.

Raft requires a majority of nodes to acknowledge before committing. If there are 3 managers total, the leader needs acknowledgments from at least 2 (itself plus one other). Without network connectivity, it cannot get the required acknowledgments.

After a timeout, the service creation fails:
```
Error: context deadline exceeded
```

The service is never committed to the cluster state.

### Hypothetical: If Raft Succeeded

Let's imagine Raft magically worked and the service was created in cluster state. Now the scheduler needs to distribute tasks.

The scheduler determines that it needs to create 3 tasks (one per replica). It allocates task IDs and assigns them to nodes. Let's say:
- Task 1 → Worker A on Machine A
- Task 2 → Worker B on Machine B
- Task 3 → Worker C on Machine C

These task assignments are written to the cluster state via Raft (which we're assuming works for this thought experiment).

### Task Dispatch

The manager needs to communicate task assignments to workers. In Docker Swarm, this doesn't happen via direct manager-to-worker communication. Instead, workers constantly poll the manager for tasks.

Workers establish long-lived gRPC connections to managers and receive task assignments over these connections. This is an important architectural detail—workers initiate the connections, not managers.

### Worker Polling

Worker B on Machine B periodically connects to a manager to check for tasks. The connection process:
1. Worker B resolves manager addresses (from cluster state)
2. Worker B connects to a manager (let's say Manager A at 192.168.65.3:2377)
3. Connection fails (we've traced this failure already)

Without being able to connect to managers, workers cannot receive task assignments. The tasks remain in the "assigned" state but are never dispatched to workers.

### With Physical IP Connections

If we use port forwarding to allow connections to managers' physical IPs, Worker B can connect to Manager A at 192.168.1.100:2377. The connection succeeds through port forwarding.

Worker B sends a request: "Any tasks for me?"

Manager A checks its state and sees Task 2 assigned to Worker B. It responds with the task details:
```json
{
  "TaskID": "task2",
  "ServiceID": "web",
  "Image": "nginx",
  "Networks": ["app-net"],
  "PublishedPorts": [8080:80],
  "DesiredState": "running"
}
```

Worker B receives this task assignment!

### Container Creation

Worker B's Docker Engine creates the container. It:
1. Pulls the nginx image (if not present)
2. Creates a container with the specified configuration
3. Attaches it to the app-net overlay network
4. Publishes port 8080:80

Container creation succeeds! The container starts running nginx.

### Overlay Network Attachment

The container gets an IP address on the app-net overlay: 10.0.1.7. The VXLAN infrastructure is activated on Worker B.

But now we encounter the VXLAN problems we traced earlier. Containers on different hosts cannot communicate because VXLAN packets are dropped or misrouted.

### Service Discovery

Docker's service discovery uses DNS. When a container queries for the service name "web", it should resolve to the service's virtual IP, which then load-balances to all task instances.

The DNS query works locally—the embedded DNS server in the Docker Engine responds with the VIP. But when the container tries to connect to the VIP, the routing mesh needs to forward the connection to an actual task instance. This forwarding uses the overlay network.

The routing mesh tries to establish a connection through VXLAN to a task instance. The VXLAN packet is created, encapsulated, sent to eth0, forwarded to VPNKit, and dropped. The connection fails.

From the container's perspective, the service is unreachable even though it can resolve the service name via DNS.

### Load Balancing Failure

The ingress load balancing mechanism also fails. When you access port 8080 on any node, the ingress network should route the request to a running task instance.

The ingress network is another overlay network with its own VXLAN. When a request arrives at Machine A's port 8080, Docker's ingress routing mesh needs to forward it to a task. If the task is on Machine B, it must traverse the VXLAN overlay. This fails as we've traced.

The result is that published ports only work for tasks running on the same machine where you make the request. If you hit Machine A's port 8080, it only works if a task is running on Machine A. If all tasks are on other machines, it fails.

### Health Checks and Task Status

Manager nodes need to monitor task health. They periodically send health check requests to tasks. These health checks might travel over the overlay network or might use direct connections.

If health checks use the overlay, they fail due to VXLAN problems. Managers mark tasks as unhealthy even though they're actually running fine. The scheduler might try to restart tasks or create replacements, leading to constant churn.

If health checks use direct connections (not through overlay), they still face the network unreachability problems. Managers cannot connect to workers on other machines to check task status.

The service enters a degraded state where tasks are running but the control plane believes they're failed.

---

## Walkthrough 7: Overlay Network Container Communication {#walkthrough7}

Let's do one final detailed trace of container-to-container communication over an overlay network, examining every layer from application to physical network.

### The Complete Stack

Container C1 on Machine A wants to send HTTP request to Container C2 on Machine B. Both are on the "app-net" overlay network (VNI 4097).

### Application Layer (Container C1)

Inside Container C1, a curl command executes:
```bash
curl http://10.0.1.6:80/
```

The curl utility resolves the IP address (already provided), creates a TCP socket, and calls connect().

### Container Network Stack

The container's network namespace has its own routing table:
```
root@container-c1:/# ip route
10.0.1.0/24 dev eth0 proto kernel scope link src 10.0.1.5
```

The kernel routes the TCP SYN packet for 10.0.1.6 to eth0.

### VEth Pair

The container's eth0 is one end of a veth pair. The packet crosses through the veth pair to the other end, which is attached to a bridge on the host's (VM's) network namespace.

### Linux Bridge

The bridge receives the packet. It performs a MAC address lookup:
```bash
bridge fdb show dev veth1234
02:42:0a:00:01:06 dst vxlan1 self permanent
```

Wait, this entry doesn't make sense for a bridge FDB. Let me correct this. The bridge forwards based on MAC addresses. The packet's destination MAC is 02:42:0a:00:01:06 (Container C2's MAC).

The bridge checks: "Do I have this MAC on any of my ports?" It might have learned that this MAC is accessible via the vxlan1 interface (if any VXLAN packets from C2 have previously arrived). Or it might not have this entry and will flood to all ports.

Either way, the packet reaches the vxlan1 interface.

### VXLAN Interface

The vxlan1 interface receives the Ethernet frame:
```
Dst MAC: 02:42:0a:00:01:06
Src MAC: 02:42:0a:00:01:05
EtherType: 0x0800 (IPv4)
Payload: TCP SYN packet (10.0.1.5 → 10.0.1.6)
```

The VXLAN module needs to encapsulate this frame. It performs an FDB lookup on the VXLAN interface (not the bridge):

```bash
bridge fdb show dev vxlan1
02:42:0a:00:01:06 dst 192.168.65.3 self permanent
```

This says: "To reach MAC 02:42:0a:00:01:06, send to VTEP at 192.168.65.3."

### VXLAN Encapsulation (Detailed)

The kernel's VXLAN module (`drivers/net/vxlan.c`) performs encapsulation. Let's trace the exact bytes:

**Original Frame (56 bytes):**
```
Ethernet Header (14 bytes):
  Dst MAC: 02:42:0a:00:01:06
  Src MAC: 02:42:0a:00:01:05
  EtherType: 0x0800

IPv4 Header (20 bytes):
  Version: 4, IHL: 5, TOS: 0, Length: 40
  ID: 54321, Flags: 0, Fragment: 0
  TTL: 64, Protocol: 6 (TCP)
  Checksum: 0x1234
  Src IP: 10.0.1.5
  Dst IP: 10.0.1.6

TCP Header (20 bytes):
  Src Port: 45678
  Dst Port: 80
  Seq: 1000000, Ack: 0
  Flags: SYN
  Window: 65535
  Checksum: 0x5678

Data (2 bytes):
  MSS option
```

**After Encapsulation (106 bytes + outer Ethernet):**

The VXLAN module adds:

```
VXLAN Header (8 bytes):
  Flags: 0x08 (I flag set, VNI valid)
  Reserved: 0x00
  VNI: 0x001001 (4097 in hex)
  Reserved: 0x00000000

UDP Header (8 bytes):
  Src Port: 45678 (hashed from inner packet)
  Dst Port: 4789
  Length: 72
  Checksum: 0x0000 (often zero for performance)

Outer IPv4 Header (20 bytes):
  Version: 4, IHL: 5, TOS: 0, Length: 106
  ID: 11111, Flags: DF, Fragment: 0
  TTL: 64, Protocol: 17 (UDP)
  Checksum: 0xabcd
  Src IP: 192.168.65.3
  Dst IP: 192.168.65.3

Outer Ethernet Header (14 bytes):
  Dst MAC: [gateway MAC for 192.168.65.1]
  Src MAC: [VM's eth0 MAC]
  EtherType: 0x0800
```

Total encapsulated packet: 120 bytes (14 Ethernet + 20 IP + 8 UDP + 8 VXLAN + 70 inner packet).

### Transmission Through VM Network Stack

This encapsulated packet is now a regular UDP/IP packet from the kernel's perspective. It's passed to the routing layer with destination 192.168.65.3.

The routing lookup returns:
```
192.168.65.3 dev eth0 scope link src 192.168.65.3
```

Wait, this indicates the destination is local (on the same link as eth0). The kernel might try to ARP for 192.168.65.3. Let's trace this.

### ARP Resolution (if needed)

If the kernel doesn't have an ARP entry for 192.168.65.3, it broadcasts an ARP request:
```
ARP Request:
  Who has 192.168.65.3?
  Tell 192.168.65.3 (same IP!)
```

This ARP broadcast goes out eth0. Via AF_VSOCK, it reaches VPNKit. VPNKit's ARP responder sees this request. The request asks for the MAC address of 192.168.65.3.

VPNKit might respond with a fake MAC address (its own virtual MAC), or it might not respond at all depending on implementation. Let's assume it responds.

The kernel receives the ARP reply and adds an entry:
```
192.168.65.3 dev eth0 lladdr [VPNKit's MAC] REACHABLE
```

Now the kernel can construct the Ethernet frame.

### Ethernet Frame Construction

The kernel builds the final Ethernet frame:
```
Dst MAC: [VPNKit's MAC from ARP]
Src MAC: [VM's eth0 MAC]
EtherType: 0x0800
Payload: [The 106-byte IP+UDP+VXLAN+inner packet]
```

This frame is passed to eth0's transmission queue.

### eth0 Driver Processing

The eth0 network device driver is virtio-vsock. It doesn't actually transmit to a physical medium. Instead, it:
1. Takes the Ethernet frame
2. Serializes it into a byte stream
3. Writes the byte stream to the shared memory buffer used by AF_VSOCK
4. Signals the host via eventfd that data is available

### Host-Side Reception (VPNKit)

On Machine A's host, the vhost-vsock kernel module receives the signal that data is available. It reads the byte stream from shared memory and delivers it to VPNKit's socket.

VPNKit's network receive thread wakes up and reads the Ethernet frame. It has received 120 bytes of data representing a complete Ethernet frame.

### VPNKit Packet Parsing

VPNKit begins its layer-by-layer parsing:

```ocaml
(* Ethernet layer *)
let eth_dst = parse_mac frame 0
let eth_src = parse_mac frame 6
let ethertype = parse_uint16 frame 12

(* IPv4 layer *)
let ip_src = parse_ipv4 frame 26  (* 192.168.65.3 *)
let ip_dst = parse_ipv4 frame 30  (* 192.168.65.3 *)
let ip_proto = parse_uint8 frame 23  (* 17 = UDP *)

(* UDP layer *)
let udp_src_port = parse_uint16 frame 34
let udp_dst_port = parse_uint16 frame 36  (* 4789 *)
```

VPNKit now has parsed up to the UDP layer. It sees destination port 4789 and recognizes this might be VXLAN traffic (though VPNKit typically doesn't have special VXLAN handling).

### Routing Decision

VPNKit's routing logic examines the destination IP: 192.168.65.3.

It checks:
1. Is this an internet address? No (RFC 1918 private range)
2. Is this the VPNKit gateway (192.168.65.1)? No
3. Is this a broadcast? No
4. Do I have a handler for this address? No

VPNKit has no routing entry for 192.168.65.3. It's not configured to know that 192.168.65.3 on another machine exists or how to reach it.

### Packet Drop

VPNKit drops the packet. The 120 bytes of carefully constructed VXLAN encapsulated data are discarded. No error message is generated. No ICMP unreachable is sent. The packet simply ceases to exist.

### Back to Container C1

In Container C1, the TCP stack is waiting for a SYN-ACK response to the SYN it sent. The initial retransmission timer (typically 1 second for SYN) counts down.

No SYN-ACK arrives. After 1 second, the TCP stack retransmits the SYN. The entire process repeats: container → veth → bridge → VXLAN encapsulation → eth0 → AF_VSOCK → VPNKit → dropped.

The retransmission timer doubles: 2 seconds, then 4, then 8, etc. After about 127 seconds of retransmissions, the TCP stack gives up. The connect() system call that curl made returns an error:

```
ETIMEDOUT: Connection timed out
```

Curl reports:
```
curl: (7) Failed to connect to 10.0.1.6 port 80: Connection timed out
```

From the user's perspective, the overlay network simply doesn't work. Containers cannot reach each other across hosts.

---

## Observable Symptoms and Diagnostics {#diagnostics}

Having traced the failure modes in detail, let's summarize what you would actually observe if you tried to run multi-node Swarm on Docker Desktop, and how you could diagnose these issues.

### Symptom 1: Node Join Failures

**Observation:** When trying to join a second manager or worker, the command hangs for 2-3 minutes then fails with:
```
Error response from daemon: Timeout was reached before node joined.
```

**Diagnosis:**
```bash
# On the joining machine's VM
docker swarm join --token TOKEN 192.168.65.3:2377

# In another terminal, watch connection attempts:
docker run --rm --privileged --pid=host debian nsenter -t 1 -m -u -n -i sh
watch 'netstat -tan | grep 2377'

# You'll see SYN_SENT state that never transitions:
tcp 0 1 192.168.65.3:45678 192.168.65.3:2377 SYN_SENT
```

The connection attempt goes nowhere because both VMs have 192.168.65.3.

### Symptom 2: Service Creation Timeout

**Observation:** Creating a service hangs and eventually fails:
```bash
docker service create --name web nginx
# Hangs for 30+ seconds
Error: context deadline exceeded
```

**Diagnosis:**
```bash
# Check Raft status
docker info | grep -A 10 Swarm

# You might see:
Swarm: active
  Managers: 1
  Nodes: 1
  Error: (none)

# Try to see logs (if accessible):
journalctl -u docker | grep raft

# You might see repeated Raft timeout errors:
WARN[...] raft: Failed to contact leader
```

### Symptom 3: Overlay Network Container Isolation

**Observation:** Containers on the same overlay network cannot ping each other across hosts:
```bash
# Container on Machine A
ping 10.0.1.6
# 100% packet loss

# Container on Machine B  
ping 10.0.1.5
# 100% packet loss
```

But containers on the same machine CAN ping each other.

**Diagnosis:**
```bash
# Check VXLAN interface
ip -d link show | grep vxlan

# Check FDB
bridge fdb show dev vxlan1

# Capture packets
tcpdump -i eth0 -n port 4789 -w vxlan.pcap

# You'll see VXLAN packets being created but never received:
# Outbound: yes
# Inbound: none
```

### Symptom 4: Gossip Membership Issues

**Observation:** If you could see memberlist state (not exposed normally), you'd see:
- All nodes marked as suspicious or dead
- Constant probe failures
- Memberlist size = 1 (only local node)

**Diagnosis:**
```bash
# There's no direct way to observe gossip in Docker
# But you can infer from service distribution failures

docker service ls
# Shows services exist

docker service ps <service-name>
# Shows tasks stuck in "assigned" state, never "running"
```

### Symptom 5: Published Ports Only Work Locally

**Observation:**
```bash
# Service created with published port
docker service create --name web -p 8080:80 nginx

# On the same machine where task is running:
curl localhost:8080
# Works!

# On a different machine:
curl <machine-ip>:8080
# Connection refused or timeout
```

**Diagnosis:**
The port is published on the host where the task runs, but the routing mesh cannot forward requests across machines due to overlay network failures.

### Key Diagnostic Commands

If you want to explore these failures empirically:

```bash
# 1. Access the Docker Desktop VM
wsl -d docker-desktop  # Windows
# or: screen ~/Library/Containers/com.docker.docker/Data/vms/0/tty  # Mac

# 2. Install diagnostic tools
apk add tcpdump iproute2 bridge-utils iputils

# 3. Check network configuration
ip addr show
ip route show
netstat -tulpn

# 4. Monitor VXLAN
ip -d link show type vxlan
bridge fdb show

# 5. Capture packets
tcpdump -i eth0 -w /tmp/capture.pcap
# Transfer this file to analyze in Wireshark

# 6. Check Swarm state
docker info
docker node ls
docker service ps <service>
```

### What You'll Observe

With these tools, you'll see:
1. Packets being created correctly with proper VXLAN encapsulation
2. Packets leaving eth0 (as seen in tcpdump)
3. No packets arriving from remote VTEPs
4. Connection timeouts in netstat
5. Services stuck in pending state
6. Single-node membership despite join attempts

This empirical observation confirms the theoretical analysis: the protocols are operating correctly within their local context, but cross-machine communication fails at the network architecture level.

---

## Conclusion: From Theory to Observable Reality

This walkthrough document has taken you from the theoretical understanding of why multi-node Swarm cannot work on Docker Desktop to the concrete, observable reality of how these failures manifest.

Every failure we traced—from TCP connection timeouts to dropped VXLAN packets to Raft consensus failures—is a direct consequence of the architectural decisions documented in the main protocol study. The failures are not random bugs or edge cases. They are the deterministic outcome of attempting to run distributed system protocols on an architecture designed for single-machine local development.

By understanding both the theory (why it cannot work) and the practice (what actually happens when it fails), you have complete knowledge of this architectural limitation. This knowledge helps you:
1. Understand that this is not a configuration problem you can fix
2. Recognize similar architectural constraints in other systems
3. Make informed decisions about when to use Docker Desktop versus other solutions
4. Explain these limitations to others who encounter them

The protocols themselves are working correctly. VXLAN creates proper encapsulation. Raft implements consensus correctly. SWIM performs failure detection as designed. The limitation is not in the protocols but in the network architecture they're running on—an architecture that was intentionally designed for a different purpose and makes different trade-offs.
