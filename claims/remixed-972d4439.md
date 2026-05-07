# Comprehensive Technical Study: Docker Swarm and VXLAN Protocols
## A First-Principles Analysis of Protocol Requirements vs Docker Desktop Architecture

**Document Purpose:** This study provides a comprehensive examination of the Docker Swarm and VXLAN protocols at a fundamental level, analyzing their specific requirements at each stage of operation. We then systematically examine Docker Desktop's architectural components and demonstrate precisely where and why they cannot fulfill the protocol requirements needed for multi-node orchestration.

**Approach:** We will build understanding from the ground up, starting with the protocols themselves before examining the architecture that attempts to support them. This methodical approach reveals not just that incompatibilities exist, but exactly why they are fundamental and unavoidable.

---

## Part I: Understanding Docker Swarm Protocol Architecture

To understand why Docker Desktop cannot support multi-node Swarm, we must first deeply understand what Swarm actually requires at a protocol level. Docker Swarm mode is not a single protocol but rather an orchestration system built on three distinct communication layers, each with specific requirements that must be satisfied simultaneously.

### 1.1 The Three-Layer Communication Model

Docker Swarm operates through three distinct but interconnected communication channels. The first layer handles cluster consensus and state management through the Raft protocol running on TCP port 2377. The second layer manages membership and failure detection through a gossip protocol based on SWIM running on both TCP and UDP port 7946. The third layer provides the data plane for container-to-container communication through VXLAN overlay networks on UDP port 4789. Each of these layers has fundamentally different characteristics and requirements, yet all three must function correctly for Swarm to operate.

Understanding why this three-layer model matters requires appreciating that each layer solves a different problem in distributed systems. Raft solves the problem of agreeing on cluster state even when nodes fail. The gossip protocol solves the problem of quickly detecting failures and disseminating information without overwhelming the network. VXLAN solves the problem of allowing containers on different hosts to communicate as if they were on the same local network. A failure in any one of these layers causes the entire Swarm to become non-functional.

### 1.2 Raft Consensus Layer: The Foundation of Cluster State

The Raft consensus algorithm forms the bedrock of Docker Swarm's cluster management. When you initialize a Swarm or join nodes to it, create services, update configurations, or perform any cluster management operation, these actions must be recorded in a distributed log that all manager nodes agree upon. This agreement must persist even if some managers fail, and it must guarantee that all surviving managers have exactly the same view of what the cluster should look like.

Raft operates through a well-defined sequence of behaviors that we need to understand in detail. When a Docker Swarm cluster starts, the first manager node initializes itself and becomes the leader. This leader status is not permanent but rather earned through an election process. The leader is the only node that can accept new state changes, acting as the single source of truth for the cluster. When subsequent manager nodes join the cluster, they connect to the leader over TCP port 2377 and begin a synchronization process.

This synchronization process is crucial to understand because it reveals the first fundamental requirement for Swarm networking. When a new manager node wants to join, it must establish a direct TCP connection to the current leader's IP address on port 2377. The new node sends a join request containing its own IP address and identity information. The leader receives this request and must be able to determine the source IP address of the connection accurately. This source IP becomes the identity of the new node in the cluster.

The leader then adds a configuration change entry to its Raft log proposing that this new node be added to the consensus group. This log entry must be replicated to a majority of existing managers before it can be committed. Here we encounter another critical requirement: each manager must be able to initiate direct TCP connections to every other manager. This is not a hub-and-spoke model where all nodes only talk to a leader. Instead, managers form a fully meshed network where any manager can become the leader and must be able to communicate directly with all others.

Let me explain why this full mesh requirement exists by examining what happens when a leader fails. If the current leader crashes or becomes unreachable, the remaining managers must elect a new leader. This election happens through a voting process where managers send RequestVote messages to each other. Each manager must be able to send these messages to and receive responses from all other managers. The first manager to receive votes from a majority of the cluster becomes the new leader. This process cannot work if managers cannot directly communicate with each other.

Once a new leader is elected, it begins accepting new state changes and replicating them to followers. Every state change (creating a service, updating a network, adding a node) becomes an entry in the Raft log. The leader sends these log entries to all followers through AppendEntries messages. Followers must acknowledge receipt of these entries. Only when a majority of nodes have acknowledged a log entry does the leader consider it committed and apply it to the state machine.

This replication model requires bidirectional communication with source IP preservation. The leader needs to know which followers have acknowledged each log entry. It tracks this by maintaining a mapping of node IDs to IP addresses. When an acknowledgment arrives, the leader checks the source IP address to determine which follower sent it. If the source IP is wrong or if it appears that all acknowledgments are coming from the same address, the leader cannot properly track which nodes are up to date and the replication process breaks down.

The Raft protocol also implements heartbeats for leader validation. The leader periodically sends heartbeat messages (empty AppendEntries requests) to all followers to prove it is still alive and maintain its authority. Followers expect to receive these heartbeats at regular intervals. If a follower does not receive a heartbeat within the election timeout period, it assumes the leader has failed and initiates a new election. These heartbeats must come from the actual leader's IP address so that followers can verify the messages are legitimate.

Now let us examine the data structures that Raft maintains and why they require specific network properties. Each manager node stores the complete Raft log on disk in the directory `/var/lib/docker/swarm/raft`. This log contains every state change that has occurred in the cluster since its creation. The log is encrypted using TLS certificates that are unique to each node. When you examine the Raft state, you will find a mapping of Raft IDs to IP addresses. Each manager has both a Raft ID (an integer used internally by the Raft protocol) and a node ID (a cryptographic hash derived from its TLS certificate). The Raft ID is mapped to the IP address that the node advertised when it joined the cluster.

This mapping is permanent for the lifetime of that Raft ID in the cluster. Once a node with Raft ID 3 is recorded as having IP address 192.168.1.101, that association cannot change. If the node's IP address changes, it effectively becomes a new node and must rejoin the cluster with a new Raft ID. The old Raft ID is added to a blacklist to prevent reuse. This blacklist is critical for preventing split-brain scenarios where an old instance of a node might rejoin and cause conflicts.

The Raft consensus layer therefore establishes several non-negotiable requirements. First, manager nodes must have stable IP addresses that do not change. Second, all manager nodes must be able to establish direct TCP connections to all other managers on port 2377. Third, the source IP address of these connections must be accurate and preserved so that nodes can be properly identified. Fourth, connections must be bidirectional, with any node able to initiate communication to any other node. Fifth, the network must support connection-oriented TCP communication with proper flow control and ordering.

### 1.3 SWIM Gossip Layer: Distributed Membership and Failure Detection

While Raft manages the authoritative cluster state, the gossip protocol handles the equally critical task of membership management and failure detection. Docker Swarm implements a variant of the SWIM protocol (Scalable Weakly-consistent Infection-style Process Group Membership protocol) using HashiCorp's memberlist library. This gossip layer serves multiple purposes that we need to examine carefully.

The primary purpose of the gossip protocol is rapid failure detection. In a distributed system, nodes can fail in various ways. They might crash completely, become network partitioned, or slow down due to resource exhaustion. The cluster needs to detect these failures quickly so that workloads can be rescheduled to healthy nodes. Traditional heartbeat mechanisms where every node sends heartbeats to every other node create N-squared communication overhead and do not scale well.

SWIM solves this scalability problem through randomized, indirect probing. Let me walk through exactly how this works because the mechanism reveals critical network requirements. The SWIM protocol operates in rounds. In each round, a node randomly selects another node from its membership list and sends a direct ping message to it over UDP port 7946. If the target node is alive and reachable, it immediately responds with an ack (acknowledgment) message.

This direct ping-ack exchange is the fast path for failure detection. When a node receives an ack, it knows with certainty that the target is alive. However, if no ack arrives within a configured timeout period (typically a few hundred milliseconds), the protocol enters the indirect probing phase. The probing node cannot yet conclude that the target has failed because the lack of response might be due to a network partition affecting only the path between these two specific nodes.

To handle potential network partitions, the probing node selects a small number of other random nodes (typically three) and sends them indirect ping requests. These requests essentially ask "Can you ping node X for me?" The selected nodes then attempt to directly ping the target node and report back whether they received acknowledgments. If the probing node receives confirmation from any of the indirect probes that the target responded, it knows the target is alive even though the direct path between them is not working.

Here is where we encounter a crucial requirement. When node A asks node B to indirectly probe node C, node B must be able to send a UDP packet directly to node C and receive a UDP response directly from node C. This creates a requirement that all nodes can send UDP datagrams to all other nodes and receive responses. The protocol cannot work through a hub or proxy because the goal is to determine if node C is reachable from multiple different network locations.

If both the direct probe and all indirect probes fail to get responses, the protocol marks the target node as suspicious. This suspicious state is important and reveals another network requirement. When a node is marked suspicious, this information is gossiped to other nodes in the cluster. The suspicious state is not an immediate death sentence because the node might merely be slow or experiencing temporary network issues. The protocol gives the node an opportunity to refute the suspicion.

If the suspected node is actually alive, it will eventually receive gossip messages informing it that it has been marked suspicious. It can then broadcast "I am alive" messages to refute the suspicion. These refutation messages must be sent from the node's actual IP address so other nodes can verify their authenticity. If source IP addresses were wrong or if messages appeared to come from a proxy, nodes could not reliably refute false suspicions.

If a node remains suspicious for a configured period without refuting the suspicion, it is finally marked as dead. This death notification is also gossiped throughout the cluster. The protocol maintains three states for each member: alive, suspicious, and dead. Transitions between these states must be agreed upon cluster-wide through the infection-style gossip mechanism.

The gossip mechanism itself is fascinating and reveals additional network requirements. SWIM uses a technique called piggybacking to disseminate membership updates efficiently. Every message sent by the protocol (ping, ack, indirect ping request) can carry additional "gossip payload" containing recent membership updates. When node A pings node B, it includes information like "I heard that node D joined" or "I think node C is suspicious." Node B then incorporates this information into its own membership view and includes it in subsequent messages it sends.

This gossip payload propagates through the cluster in a manner similar to how rumors spread in human social networks. A piece of information quickly reaches all nodes through exponential dissemination, taking only logarithmic time relative to cluster size. However, this relies on each node being able to send messages to any other node it chooses. If nodes could only communicate through a central relay, the exponential spread would be throttled to linear propagation.

Docker Swarm extends the basic SWIM protocol with several enhancements from HashiCorp's memberlist implementation. One enhancement is the use of TCP in addition to UDP. While the fast-path probes use UDP for efficiency, memberlist can also establish TCP connections for full state synchronization. When a new node joins, it receives a complete membership list over TCP from an existing member. This helps the cluster converge to a consistent state faster than pure gossip alone would achieve.

Another enhancement is a dedicated gossip layer that runs independently of the failure detection mechanism. In addition to piggybacking gossip on ping messages, memberlist can send standalone gossip packets at a configurable rate. This allows for faster propagation of membership changes than would be possible if gossip only traveled with failure detection probes. However, this also means that nodes must be able to send and receive UDP packets at a relatively high rate without packet loss.

The gossip protocol also handles the initial discovery when a node wants to join a Swarm. When you run "docker swarm join" with a token and manager address, the joining node establishes a TCP connection to port 2377 on the specified manager. During this initial connection, the manager provides the joining node with its cryptographic identity information and a list of other manager nodes to contact. The joining node then begins participating in both the Raft consensus (if it is a manager) and the gossip protocol (all nodes participate in gossip).

Let us examine the join process in detail because it highlights the integration between Raft and gossip and their combined network requirements. When a worker node joins, it connects to a manager on port 2377. The manager adds the worker to its membership list and begins including the worker in gossip messages. Other nodes in the cluster learn about the new worker through gossip and add it to their membership lists. The worker also begins probing other nodes and receiving probes, participating fully in the failure detection mechanism.

When a manager node joins, the process is more complex. The new manager must first be added to the Raft consensus group, which requires a log entry to be committed by the current Raft leader. Once admitted to the Raft group, the new manager begins receiving the complete Raft log to synchronize its state. Simultaneously, it participates in the gossip protocol to track membership across the entire cluster including workers.

This dual participation reveals a critical requirement. Manager nodes must be reachable on multiple ports simultaneously. They must accept Raft connections on TCP port 2377, accept and send gossip messages on both TCP and UDP port 7946, and accept VXLAN traffic on UDP port 4789 (for overlay networks). All of these ports must be accessible from other nodes in the cluster using the node's advertised IP address.

The gossip protocol's network requirements can be summarized as follows. First, all nodes must be able to send UDP datagrams to all other nodes on port 7946. Second, these datagrams must be delivered with high reliability, as packet loss degrades failure detection accuracy. Third, nodes must be able to receive unsolicited UDP packets from any other node, as probes can come from random sources. Fourth, the source IP address of UDP packets must be accurate so nodes can be properly identified. Fifth, TCP connections must also be possible between any pair of nodes for full state synchronization.

### 1.4 VXLAN Overlay Network Layer: The Data Plane

The third layer of Swarm networking is the data plane that carries actual application traffic between containers. Docker Swarm uses VXLAN (Virtual eXtensible Local Area Network) to create overlay networks that provide Layer 2 connectivity to containers running on different hosts. To understand why VXLAN has specific requirements that Docker Desktop cannot meet, we need to examine the protocol in detail starting from its fundamental purpose.

VXLAN was created to address a problem in large-scale cloud environments. Traditional VLANs (Virtual LANs) use a 12-bit identifier allowing for 4,094 isolated broadcast domains. In a large multi-tenant datacenter, this limit is quickly exhausted. VXLAN extends this to a 24-bit identifier called the VNI (VXLAN Network Identifier), providing 16 million possible overlay networks. This massive expansion makes VXLAN suitable for cloud-scale deployments with hundreds of thousands of tenants.

However, VXLAN is not merely about expanding the address space. It fundamentally changes how we think about Layer 2 networking by overlaying it on top of Layer 3 infrastructure. To understand this transformation, consider how traditional VLANs work. In a traditional VLAN, switches at Layer 2 learn MAC addresses and forward frames based on MAC address tables. The physical topology of switch interconnections determines which devices can communicate. Spanning tree protocols prevent loops but also limit redundancy and path diversity.

VXLAN breaks free from these constraints by tunneling Layer 2 frames inside Layer 4 UDP packets. This encapsulation allows Ethernet frames to traverse IP networks that have no concept of Ethernet or VLANs. Two virtual machines running on different physical servers in different buildings or even different cities can communicate as if they were connected to the same Ethernet switch, as long as there is IP connectivity between their hosts.

Let me explain the encapsulation format precisely because the structure reveals the protocol's requirements. When a container sends an Ethernet frame to another container on the same overlay network but different host, that frame must be encapsulated before leaving the source host. The encapsulation occurs at a component called the VXLAN Tunnel Endpoint or VTEP.

A VTEP is the ingress and egress point for VXLAN tunnels. In Docker Swarm, each host participating in an overlay network runs a VTEP. The VTEP can be implemented in hardware on a network switch or in software in a hypervisor or container runtime. Docker implements VTEPs in the Linux kernel using the kernel's VXLAN module. This kernel implementation is critical for performance because it means packet encapsulation and decapsulation happen in kernel space with zero userspace involvement.

When an Ethernet frame arrives at the VTEP from a local container, the VTEP must determine where to send it. This is where the Forwarding Database (FDB) comes into play. The FDB is similar to a MAC address table in a traditional switch, but instead of mapping MAC addresses to switch ports, it maps MAC addresses to remote VTEP IP addresses. The FDB tells the VTEP "to reach a container with MAC address 02:42:ac:11:00:03, send the frame to the VTEP at IP address 192.168.1.102."

The VTEP looks up the destination MAC address in the FDB. If an entry exists, the VTEP knows which remote VTEP to send the packet to. It then performs encapsulation by adding multiple headers to the original frame. Starting from the innermost header and working outward, the structure is as follows.

The innermost component is the original Ethernet frame exactly as it was sent by the source container. This frame contains the source container's MAC address, the destination container's MAC address, and the original payload (typically an IP packet containing TCP or UDP data for an application).

Next comes the VXLAN header, which is 8 bytes long. The first byte contains flags, where the I flag must be set to 1 to indicate a valid VNI. This is specified in RFC 7348 and is non-negotiable. The next three bytes are the VNI itself, a 24-bit identifier that indicates which overlay network this frame belongs to. The remaining four bytes are reserved and set to zero.

Outside the VXLAN header comes a UDP header. This is a standard UDP header where the destination port is 4789, the IANA-assigned port for VXLAN. The source port is typically derived from a hash of the inner frame's headers to provide entropy for Equal-Cost Multi-Path (ECMP) routing. This entropy allows the network to load-balance VXLAN traffic across multiple paths when redundant links exist. The UDP checksum field may be set to zero to reduce computational overhead, as the encapsulated payload already has its own checksums.

Outside the UDP header is an IP header. This is where the source and destination IP addresses of the VTEPs appear. The source IP is the IP address of the local VTEP (the host sending the encapsulated frame), and the destination IP is the IP address of the remote VTEP (the host where the destination container is running). This outer IP header is what the physical network uses to route the packet. Routers and switches in the underlay network have no knowledge of VXLAN or containers. They simply see an IP packet with a particular source and destination and forward it according to their routing tables.

Finally, there is an outer Ethernet header that contains the MAC addresses needed to traverse the local network segment. These MAC addresses change at each Layer 3 hop as the packet is forwarded through the network, as is normal for IP routing.

Now let us examine the receiving side to understand the complete picture. When a VXLAN-encapsulated packet arrives at a remote host, the host's network interface receives it as a normal IP packet. The kernel's networking stack processes the outer IP header and sees that the destination IP matches the host's IP address. It then examines the UDP header and sees destination port 4789. The kernel recognizes this as VXLAN traffic and hands it to the VXLAN module.

The VXLAN module extracts the VXLAN header and checks that the I flag is set. It reads the VNI to determine which overlay network this frame belongs to. The module then extracts the inner Ethernet frame and performs an important learning operation. It updates its FDB with a mapping that says "frames destined for the source MAC address of this frame should be sent to the source IP address of the outer IP header." This is how the FDB is populated through learning.

After learning, the VXLAN module forwards the inner Ethernet frame to the appropriate local container. From the container's perspective, it simply receives an Ethernet frame as if it came from a directly connected peer. The container has no knowledge that the frame was encapsulated and sent across an IP network.

This encapsulation and learning mechanism reveals several critical requirements. First, each VTEP must have a real IP address that is routable within the network. The outer IP header must contain an actual IP address that routers know how to reach. Second, VTEPs must be able to send UDP datagrams to other VTEPs on port 4789. Third, these UDP datagrams must be delivered reliably and in order, as they carry actual application traffic. Fourth, the source IP address of VXLAN packets must be the actual IP address of the sending VTEP so that FDB learning works correctly.

Let me expand on this last point because it is particularly important. When a VTEP receives a VXLAN frame and learns "MAC address X is reachable via IP address Y," that association is stored in the FDB. Future frames destined for MAC address X will be sent to IP address Y. If the source IP address is wrong (for example, if a proxy or NAT changed it), the FDB will learn incorrect information. When the destination container tries to send a response frame back to the source container, the return path will fail because the FDB has the wrong IP address for the source.

This source IP preservation requirement eliminates the possibility of using NAT or proxies in the VXLAN data path. The kernel VXLAN module must be able to send and receive UDP packets directly with no userspace intermediaries and no address translation. This is a fundamental characteristic of how VXLAN is designed and cannot be worked around.

The VXLAN protocol also specifies behavior for broadcast, unknown unicast, and multicast (BUM) traffic. When a container sends a broadcast frame (destination MAC address FF:FF:FF:FF:FF:FF) or when it sends a frame to a MAC address not in the FDB, the VTEP must flood the frame to all other VTEPs in the same overlay network. This flooding can be accomplished through IP multicast or through head-end replication.

Docker Swarm typically uses head-end replication, where the source VTEP duplicates the frame and sends a copy to each known remote VTEP individually. This requires the VTEP to maintain a list of all other VTEPs participating in the overlay network and to be able to send UDP packets to each of them. The VTEP list is distributed through the Swarm control plane, but the actual transmission of replicated packets requires direct UDP connectivity to all peers.

The performance characteristics of VXLAN also matter. Because encapsulation and decapsulation happen for every packet, they must be extremely efficient. The Linux kernel's VXLAN implementation achieves this by processing packets entirely in kernel space. When a packet arrives from a container, it goes from the container's network namespace to the kernel's VXLAN module to the physical network interface driver without ever entering userspace. This allows for line-rate performance on modern hardware.

If VXLAN processing were moved to userspace, every packet would need to cross the kernel-userspace boundary twice (once for encapsulation, once for transmission). At gigabit speeds, this context switching overhead would quickly overwhelm the system. The kernel implementation also enables hardware offloading features where modern NICs can perform VXLAN encapsulation and decapsulation in hardware, further improving performance.

VXLAN requires a minimum MTU on the underlay network to avoid fragmentation. The encapsulation adds 50 bytes of overhead (8 bytes VXLAN header, 8 bytes UDP header, 20 bytes IPv4 header, 14 bytes outer Ethernet header). If the overlay network supports 1500-byte frames, the underlay must support at least 1550 bytes. Docker handles this by reducing the MTU of overlay networks to 1450 bytes by default, ensuring that encapsulated packets fit within the standard 1500-byte Ethernet MTU.

The VXLAN layer of Docker Swarm can be summarized with these requirements. First, VTEPs must have routable IP addresses on the underlay network. Second, VTEPs must be able to send and receive UDP packets on port 4789 to and from all other VTEPs. Third, source IP addresses must be preserved accurately for FDB learning. Fourth, packet processing must occur in kernel space for acceptable performance. Fifth, the underlay network must support the necessary MTU. Sixth, any VTEP must be able to send to any other VTEP without restriction.

---

## Part II: Docker Desktop Architecture Components

Having established what Docker Swarm requires, we now turn to examining Docker Desktop's architecture. Docker Desktop was designed to provide a simple, user-friendly way to run containers on Windows and macOS. The design choices made to achieve this simplicity are precisely what make multi-node Swarm impossible. Let us examine each architectural component in detail.

### 2.1 The Hypervisor Layer and VM Isolation

Docker Desktop runs Docker Engine inside a Linux virtual machine rather than natively on the Windows or macOS host. This design is necessary because Docker Engine requires Linux kernel features (namespaces, cgroups, overlay filesystems) that do not exist on Windows or macOS. The virtualization layer varies by platform. On macOS, Docker Desktop uses Apple's Hypervisor.framework (formerly HyperKit). On Windows, it uses either Hyper-V or WSL2 depending on configuration.

The virtual machine running Docker Engine is minimal and purpose-built. It is based on LinuxKit, a toolkit for creating lean, immutable Linux distributions designed for containers. The VM includes only the components necessary to run Docker Engine and support containers. Users cannot shell into this VM through normal means, cannot install packages, and cannot modify its configuration beyond what Docker Desktop's settings allow.

This VM isolation has important security and operational benefits. If a container escape occurs and an attacker breaks out of container isolation, they land inside the VM rather than on the host. The VM provides a second layer of defense. The VM also makes resource management cleaner. Docker Desktop can limit the CPU, memory, and disk resources allocated to the VM, preventing containers from starving the host of resources.

However, this isolation creates the first barrier to multi-node Swarm. The Docker Engine instance running inside the VM only sees the VM's network environment. It has no direct access to the host's network interfaces, routing tables, or IP addresses. The Docker Engine believes it is running on a regular Linux machine with its own IP address, but this IP address is local to the VM and not visible on the physical network.

### 2.2 AF_VSOCK: The VM-Host Communication Channel

Communication between the VM and the host operating system occurs through a mechanism called AF_VSOCK (Address Family Virtual Sockets) on Linux and Hyper-V sockets on Windows. Both are implementations of the same concept: a communication channel specifically designed for VM-to-hypervisor communication that bypasses traditional networking entirely.

AF_VSOCK provides a socket API similar to TCP or Unix domain sockets but with fundamentally different behavior. Instead of IP addresses, AF_VSOCK uses Context IDs (CIDs). The hypervisor assigns a CID to each VM and to itself. The host always has CID 2 (VMADDR_CID_HOST), and each VM gets a unique CID like 3, 4, 5, etc. There is also a loopback CID 1 for connections within the VM to itself.

AF_VSOCK communication occurs through shared memory regions that the hypervisor allocates in the host's physical RAM. When the VM wants to send data to the host, it writes to this shared memory region and uses interrupt signaling (ioeventfd/irqfd) to notify the host that data is available. The host reads from the shared memory and processes the data. This mechanism is extremely fast because it avoids the overhead of network stack processing, virtual NICs, and software bridging.

Docker Desktop uses AF_VSOCK for multiple purposes. The Docker CLI on the host communicates with the Docker Engine in the VM using AF_VSOCK. When you run "docker ps" from your Mac or Windows host, that command travels through AF_VSOCK to reach dockerd in the VM. Port forwarding control messages also travel through AF_VSOCK. When Docker Engine wants to publish a container port, it sends a request through AF_VSOCK to the host backend, which then creates a listener on the host.

Most importantly for our analysis, all network traffic from the VM passes through AF_VSOCK. The VM's eth0 interface is not connected to a traditional virtual NIC driver that would communicate with a software bridge or physical NIC. Instead, eth0 connects to the AF_VSOCK mechanism. When the VM's network stack sends a packet out eth0, that packet is serialized and sent through AF_VSOCK to a userspace process on the host called VPNKit.

The critical limitation of AF_VSOCK is that it only provides communication between a VM and its hypervisor on the same physical machine. The shared memory regions that AF_VSOCK uses exist in the RAM of one computer. Machine A's RAM and Machine B's RAM are physically separate. There is no mechanism for shared memory to span multiple computers. AF_VSOCK CIDs are scoped to a single hypervisor. If two different machines both have a VM with CID 3, those are two independent VMs with no relationship to each other.

This fundamental limitation means that AF_VSOCK cannot be used to communicate between VMs running on different physical hosts. When Docker Swarm running in Machine A's VM tries to send packets to Machine B's VM, those packets reach the AF_VSOCK layer. AF_VSOCK can deliver them to Machine A's host, but it has no mechanism to forward them to Machine B. The packets reach a dead end.

Even if we could somehow extend AF_VSOCK to support remote communication, doing so would defeat its purpose. AF_VSOCK exists precisely to avoid the complexity of networking. If we added IP addresses, routing, discovery protocols, and all the machinery needed for remote communication, we would have reinvented TCP/IP networking. At that point, there would be no reason to use AF_VSOCK instead of traditional networking.

### 2.3 VPNKit: The Userspace Network Stack

VPNKit is a sophisticated userspace process that provides network connectivity for the Docker Desktop VM. It is implemented in OCaml using libraries from the MirageOS unikernel project. VPNKit receives raw Ethernet frames from the VM via AF_VSOCK, parses them, and creates corresponding network connections on the host.

To understand VPNKit's role and limitations, we must first understand the problem it was designed to solve. Many corporate networks run VPNs that are configured to block traffic that appears to come from virtual machines or that has been NAT'd. These VPNs detect characteristics like changed TTL values or recognize that the source MAC addresses belong to VM vendors. When Docker Desktop was initially developed, users running it on corporate networks with strict VPNs found that their containers could not access the internet.

VPNKit solves this by making VM traffic appear as legitimate traffic from a host process. When a container makes an outbound connection to the internet, VPNKit intercepts the packet, parses it layer by layer, and then creates a real socket connection from the host. From the VPN's perspective, VPNKit is just another application running on the host making normal network requests. The VPN cannot tell that this traffic actually originated from a container inside a VM.

Let me trace through exactly how VPNKit processes a packet to illustrate both its capabilities and its limitations. Suppose a container wants to connect to google.com. The container's application makes a socket call to connect to Google's IP address on port 443. The kernel inside the VM creates a TCP packet with the container's IP as source and Google's IP as destination. This packet is encapsulated in an Ethernet frame and sent to the VM's eth0 interface.

The eth0 driver, instead of talking to a virtual NIC, writes the raw Ethernet frame to the AF_VSOCK channel. VPNKit, running in userspace on the host, receives this frame. It begins parsing from the outermost layer inward. First, it parses the Ethernet header to determine the protocol type (typically IPv4 or IPv6). Then it parses the IP header to extract source and destination IP addresses and determine the transport protocol (TCP or UDP).

If the protocol is TCP and the destination IP is an internet address, VPNKit recognizes this as an outbound connection attempt. It creates a real TCP socket on the host using the host's native socket API. This socket makes a real connection to Google's server. When the connection succeeds, Google sees it as coming from the host, not from a VM. The host's VPN processes the traffic and allows it.

VPNKit now maintains two separate connections. One is a virtual connection in VPNKit's memory representing the connection from the container to Google. The other is a real socket connection from the host process to Google's server. VPNKit proxies data between these two connections. When the container sends application data, VPNKit reads it from the virtual connection and writes it to the real socket. When Google sends response data, VPNKit reads it from the real socket and crafts a synthetic TCP packet to send back to the container through AF_VSOCK.

This connection recreation mechanism works well for outbound connections where the VM is the client and internet services are the servers. VPNKit knows how to handle common protocols like TCP, UDP, ICMP, and special cases like DNS and DHCP. It even includes an embedded DNS server and DHCP server to provide these services to the VM without the traffic leaving the host.

However, VPNKit's design is fundamentally unidirectional. It is built to handle VM-initiated connections to external destinations. It has no mechanism for handling peer-to-peer communication between VMs on different hosts. When VPNKit receives a packet from the VM, it makes a routing decision based on the destination IP address. If the destination is an internet address (not in RFC 1918 private ranges or special use ranges), VPNKit creates an outbound connection. If the destination is a special VPNKit service (like the embedded DNS server), VPNKit routes it internally.

But what if the destination is the IP address of another Docker Desktop VM? VPNKit has no handler for this case. It does not maintain a registry of peer VMs. It has no way to discover that 192.168.65.4 is a VM running on another machine somewhere. Even if such a registry existed, VPNKit would need a mechanism to forward packets to that remote machine. This would require implementing some form of VPN tunnel protocol between VPNKit instances on different hosts, which is far beyond its design scope.

VPNKit also cannot handle unsolicited inbound traffic. All the connections it manages are initiated from inside the VM. If an external source tried to connect directly to the VM, there would be no listener on the host to accept that connection. The port forwarding mechanism (which we will examine next) provides limited inbound connectivity, but only for explicitly published container ports, not for the Docker Engine's own listening ports.

The performance characteristics of VPNKit are also relevant. Because it operates in userspace and recreates connections rather than routing packets, it adds significant latency compared to kernel networking. For typical internet traffic, this latency is negligible compared to network round-trip times measured in tens or hundreds of milliseconds. But for VXLAN overlay traffic between containers on the same local network, the latency would be significant and the CPU overhead would be unacceptable.

Consider what would happen if VPNKit tried to proxy VXLAN traffic. Every VXLAN packet would need to cross from the kernel to VPNKit, be parsed, have some action taken (even if that action is just "drop because unknown destination"), and potentially cross back to the kernel. At the packet rates generated by overlay networks, this would create millions of context switches per second, overwhelming the CPU.

VPNKit's role and limitations can be summarized as follows. It provides excellent internet connectivity by making VM traffic appear as host traffic, solving the corporate VPN problem. It supports standard client-server protocols where the VM is the client. It handles DNS, DHCP, and basic networking services internally. However, it cannot route to peer VMs on other machines. It cannot handle unsolicited inbound connections. It cannot efficiently process high packet rate overlay network traffic. It is designed for single-machine, VM-to-internet communication and cannot be extended to support distributed systems.

### 2.4 Port Forwarding (slirp-proxy): The Inbound Connection Problem

While VPNKit handles outbound connections, Docker Desktop needs a separate mechanism for inbound connections to published container ports. When you run a container with `-p 8080:80`, you expect to be able to connect to localhost:8080 on your host and have that connection reach the container's port 80. This is accomplished through a userspace proxy called slirp-proxy (on some versions) or through the Docker Desktop backend process directly.

The port forwarding mechanism works as follows. When Docker Engine wants to publish a port, it sends a request to the Docker Desktop backend running on the host. This request specifies which host port to listen on (8080), which container IP and port to forward to (172.17.0.2:80), and the protocol (TCP or UDP). The backend validates that the host port is available and then creates a listener on that port.

When an external client connects to the published port, the backend accepts the connection on the host side. It then establishes a separate connection through AF_VSOCK to the VM. Inside the VM, a process called vpnkit-forwarder receives the connection request and opens yet another connection to the target container. The backend then proxies data between these two connections, reading from one socket and writing to the other.

This creates a two-connection proxy model. Connection 1 is between the external client and the host backend. Connection 2 is between the host backend and the container. The external client and the container never communicate directly. They each have a separate TCP connection, and the backend shuttles data between them.

This separation has important consequences. When the container's application calls getpeername() to determine who connected to it, it sees the connection as coming from the internal VM address, not from the actual external client. The source IP address of the original client is lost. The backend may log this information for diagnostic purposes, but the container application cannot access it through standard socket APIs.

For simple web services, this loss of source IP is often acceptable. Web servers typically log connections based on HTTP headers like X-Forwarded-For rather than relying on socket source addresses. But for protocols that need accurate peer identification, the source IP loss is fatal.

Docker Swarm's cluster management is precisely such a protocol. When a worker node connects to a manager node on port 2377, the manager needs to know the actual IP address of the worker. The manager records this IP address in its cluster state and uses it to track which nodes are members of the cluster. If all connections appear to come from localhost or from the host's internal address, the manager cannot distinguish between different nodes.

Port forwarding also faces technical challenges with UDP. TCP connections have clear start and end points (SYN to establish, FIN to close). This makes it relatively easy to track which client connection corresponds to which container connection. UDP is connectionless with no handshake or teardown. The proxy must use heuristics like tracking source/destination IP/port tuples and implementing session timeouts. For high-rate bidirectional UDP flows like VXLAN, these heuristics become unreliable and CPU-intensive.

Even more fundamentally, port forwarding only works for container ports, not for Docker Engine's own ports. The `-p` flag publishes a port that a containerized application is listening on. But Docker Swarm's ports are bound by dockerd itself, not by a container. There is no mechanism to "publish" port 2377 from Docker Engine because the port publishing system is specifically designed for containers.

You might wonder if we could run Docker Engine itself in a container and publish its ports. This creates a recursive problem. The Engine that runs the Engine container would need to be running somewhere, and we are back to the same problem. Additionally, Docker Engine needs access to the host's cgroups, namespaces, and filesystems to actually run containers. Running it in a container severely limits these capabilities.

The port forwarding mechanism reveals a fundamental design assumption in Docker Desktop: traffic flows in one direction, from external clients to internal services. This assumption works perfectly for development workloads where developers access containerized web apps from their browser. It breaks down completely for distributed systems that require bidirectional peer-to-peer communication.

### 2.5 Non-Routable IP Addresses: The Addressing Problem

The Docker Desktop VM receives its IP address from VPNKit's internal DHCP server. This address is typically in the 192.168.65.0/24 range, with the VM usually getting 192.168.65.3. This IP address exists only within the virtualization context and has no presence on the physical network.

To understand what "non-routable" means in this context, consider the sequence of events when another machine tries to reach this address. Suppose a Docker Swarm node on Machine B tries to connect to the advertised address 192.168.65.3 on Machine A. Machine B's kernel performs a routing table lookup for 192.168.65.3.

The routing table has entries for known networks. It might have an entry for 192.168.1.0/24 (the local LAN), 10.0.0.0/8 (corporate network), and a default route for everything else pointing to the gateway. The address 192.168.65.3 does not match any local network entry. It falls through to the default route and is forwarded to the router.

The router receives this packet and performs its own routing lookup. It has routes for various networks it knows about, but 192.168.65.0/24 is not among them. This network is not advertised by any routing protocol. No router announces "I know how to reach 192.168.65.0/24." The packet is either forwarded to a default route (typically toward the internet) or dropped as unreachable.

Even on the same local LAN, the problem persists. If Machine B tries to reach 192.168.65.3 and they are on the same Ethernet segment, Machine B must first resolve the IP to a MAC address using ARP. It broadcasts an ARP request asking "Who has 192.168.65.3?" Machine A's host receives this broadcast, but it does not respond because it does not have 192.168.65.3. The VM inside Machine A has that address, but the VM's eth0 is not connected to the physical Ethernet. Machine A's physical NIC has a completely different IP address like 192.168.1.100.

With no ARP response, Machine B cannot send packets to 192.168.65.3 even on the local LAN. The address is unreachable because it is not bound to any physical network interface that participates in the physical network.

You might propose using the host's IP address instead. If we could make Docker Engine advertise the host's IP (192.168.1.100) instead of the VM's IP, would that work? This runs into a different problem. Docker Engine can only bind sockets to IP addresses that exist on network interfaces in its own network namespace. The host IP 192.168.1.100 exists on the host's network interface, not on any interface inside the VM.

When Docker Engine tries to bind to 192.168.1.100:2377, the bind system call fails with EADDRNOTAVAIL (address not available). The kernel checks if the specified IP address is configured on any local interface. Finding no such interface, it rejects the bind request. Docker Engine cannot listen on an IP address that does not exist in its network environment.

The non-routable IP problem interacts with all the other Docker Desktop components to create an impenetrable barrier. AF_VSOCK ensures the VM's network is isolated. VPNKit provides internet connectivity but only for the VM's own address. Port forwarding could theoretically expose some services on the host IP, but not Docker Engine's ports and not without losing source IP information. The non-routable IP is both a consequence of the architecture and a contributor to its limitations.

---

## Part III: Mapping Protocol Requirements to Architectural Limitations

We have now examined both what Docker Swarm needs and what Docker Desktop provides. We can systematically map each protocol requirement to the specific architectural component that blocks it. This mapping will demonstrate that the incompatibilities are not superficial bugs but rather fundamental architectural mismatches.

### 3.1 Raft Consensus Requirements vs Docker Desktop

Let us examine each requirement of the Raft protocol and identify precisely which Docker Desktop component blocks it.

**Requirement 3.1.1: Stable, Routable IP Addresses for Manager Nodes**

Raft requires that each manager node advertise a stable IP address that other managers can use to connect to it. This address must remain constant because it gets recorded in the Raft log and in each manager's configuration. If a node's address changes, it must rejoin the cluster with a new identity, as the protocol cannot handle dynamic address reassignment.

Docker Desktop blocks this in multiple ways. First, the VM has IP address 192.168.65.3 (or similar) which is assigned by VPNKit's DHCP server. This address exists only within the virtualization context and is not routable on the physical network. Other machines cannot send packets to 192.168.65.3 because their routing tables have no entries for the 192.168.65.0/24 network. Even on the same LAN, ARP resolution would fail because no physical interface claims this address.

Second, even if we could make the address routable, it is not stable in the way Raft requires. The address is dynamically assigned and could theoretically change on VM restart. More fundamentally, if you wanted to use the host's IP address instead, Docker Engine cannot bind to it because that address does not exist on any interface within the VM's network namespace.

The AF_VSOCK layer compounds this problem. All network communication from the VM goes through AF_VSOCK to reach the host. There is no path from the VM's network stack to the physical network. The VM's eth0 interface terminates at AF_VSOCK, not at a bridge or virtual NIC that connects to the physical network. The architectural decision to use AF_VSOCK for isolation fundamentally prevents the VM from participating in the physical network as a peer.

**Requirement 3.1.2: Direct TCP Connectivity Between All Manager Nodes**

Raft requires that any manager node can establish a direct TCP connection to any other manager node on port 2377. This bidirectional connectivity is essential for leader election, log replication, and heartbeat mechanisms. The protocol does not support hub-and-spoke topologies where all nodes communicate through a central relay.

Docker Desktop blocks direct TCP connectivity in multiple ways. Consider two machines, A and B, both running Docker Desktop. When Swarm on Machine A's VM tries to connect to port 2377 on Machine B's advertised address (say, 192.168.65.3), the connection attempt goes through several layers, each of which fails.

First, the connection attempt is initiated from Docker Engine inside Machine A's VM. The kernel routing table lookup determines that the destination (Machine B's 192.168.65.3) is not on any local network, so the packet is forwarded to the default gateway, which is 192.168.65.1 (VPNKit). The packet reaches VPNKit through AF_VSOCK.

VPNKit receives the packet and attempts to parse it to determine how to handle it. VPNKit's decision logic looks at the destination IP address. For internet addresses, VPNKit creates an outbound connection from the host. For internal VPNKit services (like the DNS server at a specific address), VPNKit handles the request internally. For the address 192.168.65.3, VPNKit has no handler. This is not an internet address (it's in the RFC 1918 private address space), and it's not a VPNKit internal service address.

VPNKit has no registry of peer Docker Desktop VMs. It does not know that 192.168.65.3 exists on another machine somewhere. Even if such a registry existed, VPNKit would need a mechanism to forward packets to the other machine. This would require implementing a VPN tunnel protocol between VPNKit instances, which is far beyond its design scope and would essentially recreate functionality that regular networking already provides.

The packet is dropped. The connection attempt times out. Docker Engine waits for a response that never comes. From Docker Engine's perspective, the remote node is unreachable. The Raft protocol cannot function when nodes cannot reach each other.

Even if we tried to use port forwarding to expose port 2377 on the host, we encounter additional problems. Port forwarding only works for container ports, not for Docker Engine's own ports. The `-p` flag publishes ports that containerized applications bind to, but Swarm's port 2377 is bound by dockerd itself. There is no mechanism in Docker Desktop to publish the Engine's listening ports to the host.

Furthermore, even if port forwarding could somehow work, it would break source IP preservation. When a manager receives a connection via the port forwarding proxy, it sees the connection as coming from localhost or the internal gateway address, not from the actual remote manager. The manager cannot distinguish between different remote nodes because they all appear to come from the same proxy address. This violates Raft's requirement that it be able to identify which specific node is sending each message.

**Requirement 3.1.3: Source IP Preservation for Node Identification**

Raft maintains mappings of node IDs to IP addresses. When a message arrives, the recipient identifies the sender by looking at the source IP address and finding the corresponding node ID in its mapping. This identification is critical for tracking which nodes have acknowledged log entries, which nodes are up to date, and which nodes are participating in leader elections.

Docker Desktop's port forwarding mechanism breaks source IP preservation. Let me trace through exactly what happens when a connection is forwarded. An external client connects to the published port on the host. The Docker Desktop backend accepts this connection on the host side. It then establishes a separate connection through AF_VSOCK to reach the target container. These are two independent TCP connections with their own sequence numbers, acknowledgments, and state.

When data arrives from the client connection, the backend reads it from that socket and writes it to the container connection. From the container's perspective, the connection originates from the VM's internal network, not from the external client. If the container application calls getpeername() to determine the peer address, it receives the address of the proxy component, not the original client.

For typical web applications, this source IP loss is acceptable or even expected. Web servers typically rely on HTTP headers like X-Forwarded-For rather than socket peer addresses. But Raft operates at the TCP level and requires accurate source IP information. There is no application-layer header where source IP can be preserved, and the protocol spec does not account for proxying.

Additionally, the port forwarding system could not solve the Raft connectivity problem even without the source IP issue because it only works in one direction. Port forwarding allows inbound connections from external clients to containers. It does not provide a mechanism for containers to establish outbound connections to specific remote hosts' published ports. The Raft protocol requires bidirectional connectivity where any node can connect to any other node.

**Requirement 3.1.4: Log Replication with Persistent Storage**

Raft maintains a persistent log on disk at /var/lib/docker/swarm/raft. This log contains every state change that has occurred in the cluster. When a new manager joins, it receives the complete log (or a snapshot plus recent entries) from the leader. When managers communicate, they reference log indices and terms to ensure they're synchronized.

While Docker Desktop's VM does provide persistent storage through volume mounting, the isolation created by the VM architecture means that each Docker Desktop instance has its own completely independent log. There is no mechanism for logs from different Docker Desktop instances to be synchronized because the instances cannot communicate.

The Raft implementation expects to be able to send log entries over the network to other managers. When the leader has a new entry to replicate, it sends AppendEntries RPCs containing the log data to all followers. This requires network connectivity between managers, which we have established does not exist across Docker Desktop instances.

**Requirement 3.1.5: Leader Election Mechanism**

When a follower does not receive heartbeats from the leader within the election timeout period, it transitions to the candidate state and begins an election. It sends RequestVote RPCs to all other managers asking them to vote for it as the new leader. Managers respond with their votes, and if the candidate receives votes from a majority, it becomes the new leader.

This election mechanism has specific network requirements. The candidate must be able to send RequestVote messages to all other managers. These messages must arrive reliably, and responses must be received within a bounded time period. The protocol uses timeouts to detect when elections fail and must be restarted.

Docker Desktop blocks leader elections because the candidate cannot send RequestVote messages to remote managers. When a Docker Desktop instance tries to send these messages, they follow the same path as any other outbound packet: through the VM's network stack, out eth0, through AF_VSOCK, to VPNKit, where they are dropped because the destination addresses are not internet addresses and not internal VPNKit services.

Even in a scenario where some form of connectivity existed, the latency introduced by userspace proxying would likely exceed Raft's timeout thresholds. Raft assumes a reliable network with bounded latency, typically milliseconds. The context switches and userspace processing in VPNKit would add unpredictable latency spikes that could cause spurious election timeouts.

### 3.2 SWIM Gossip Protocol Requirements vs Docker Desktop

The gossip protocol presents a different set of requirements from Raft. While Raft uses TCP for reliable point-to-point communication, gossip uses UDP for efficient many-to-many communication. Let us examine how Docker Desktop blocks each gossip requirement.

**Requirement 3.2.1: Direct UDP Communication on Port 7946**

The SWIM protocol requires that every node can send UDP datagrams to every other node on port 7946 and receive UDP datagrams from any other node. This bidirectional UDP communication is fundamental to how the protocol operates. Nodes randomly select peers to probe, and any node can be the source or destination of these probes.

Docker Desktop blocks UDP communication between nodes for the same reasons it blocks TCP communication, but with additional complications specific to UDP. When a Docker Desktop instance tries to send a UDP datagram to another instance's address, the datagram goes through the VM's network stack, through eth0, through AF_VSOCK, to VPNKit.

VPNKit's handling of UDP is more complex than its handling of TCP because UDP is connectionless. For TCP, VPNKit can track connections from the SYN packet through the entire lifecycle until FIN. For UDP, there are no connection markers. VPNKit must use heuristics to group related datagrams into sessions.

When VPNKit receives a UDP datagram destined for an internet address, it creates a UDP socket on the host, sends the datagram, waits for a response, and associates the response with the original sender so it can be routed back. This works for request-response protocols like DNS where there is a clear stimulus (query) and response (answer).

However, for the SWIM gossip protocol, the traffic pattern is different. Probes and acknowledgments are short messages that do not follow a simple request-response pattern. Multiple nodes might probe the same target simultaneously. Indirect probes create triangular communication patterns where node A asks node B to probe node C. These patterns do not fit VPNKit's model of VM-initiated outbound connections to internet services.

Moreover, VPNKit has no mechanism to deliver unsolicited inbound UDP packets to the VM. If an external node tried to send a probe to a Docker Desktop VM, that datagram would arrive at the host's network interface. There is no listener on the host that would accept it and forward it through AF_VSOCK to the VM. The host's firewall would likely drop it as unsolicited traffic, or it would be ignored because no process is listening on port 7946.

The gossip protocol specifically requires that nodes accept unsolicited probes from random peers. This is how failure detection works—nodes randomly select others to probe without prior arrangement. Docker Desktop's architecture cannot support this because all communication must be initiated from inside the VM, and there is no path for external traffic to reach the VM unsolicited.

**Requirement 3.2.2: Indirect Probing for Network Partition Detection**

When a direct probe fails, the SWIM protocol performs indirect probing. Node A selects several random nodes (say B, C, and D) and asks them to probe target node E. Nodes B, C, and D then each attempt to send direct probes to E and report back to A whether they received acknowledgments.

This indirect probing mechanism requires a fully meshed network where any node can probe any other node directly. Docker Desktop cannot support this because the Docker Desktop instances cannot communicate with each other. When node A asks node B to probe node E, even if that request could somehow reach node B, node B would still be unable to probe node E for the same reasons A could not.

The indirect probing mechanism exists specifically to distinguish between node failure and network partition. If A cannot reach E but B can, then E is alive but there's a network partition between A and E. If neither A nor B can reach E, then E is likely failed. This distinction is impossible to make when no nodes can reach any other nodes.

**Requirement 3.2.3: Gossip Dissemination of Membership Updates**

The gossip protocol disseminates membership information by piggybacking it on probe and acknowledgment messages. When node A probes node B, the probe message includes recent membership updates that A has learned about. B incorporates these updates into its view and includes them in subsequent messages it sends. This creates an infection-style spread of information that quickly propagates throughout the cluster.

For gossip dissemination to work, there must be a connected graph of nodes where each node can send messages to other nodes. The connectivity does not need to be complete (every node to every other node simultaneously), but there must be paths through which information can flow. In a completely partitioned set of Docker Desktop instances, there are no paths. Each instance is an island with no connection to any other island.

Even if we could somehow establish limited connectivity (perhaps through elaborate VPN tunnels or cloud-based relay services), the gossip protocol's efficiency depends on direct peer-to-peer communication with low latency. Adding relays, proxies, or tunnels would add latency and complexity that would degrade the protocol's performance and defeat its design goals.

**Requirement 3.2.4: Fast Convergence Through High Message Rate**

The gossip protocol achieves fast convergence by operating at a relatively high message rate. Nodes typically send probes every few hundred milliseconds and may send standalone gossip messages even more frequently. This high message rate ensures that information propagates through the cluster in logarithmic time relative to cluster size.

Docker Desktop's userspace networking cannot efficiently support this message rate. Even if VPNKit could somehow forward gossip messages between instances, the overhead of userspace packet processing would be significant. Each message would require context switches between kernel and userspace, protocol parsing, routing decisions, and potential socket operations.

At a message rate of several packets per second per node, with dozens or hundreds of nodes, the aggregate packet rate would overwhelm userspace processing. Kernel networking can handle such rates efficiently because packet processing occurs in kernel space with optimized data structures and minimal context switching. Userspace networking like VPNKit incurs much higher per-packet overhead.

**Requirement 3.2.5: Accurate Detection of Node Failures vs Network Problems**

The SWIM protocol distinguishes between actual node failures and transient network problems through its suspicion mechanism. When probes fail, the target is marked as suspicious rather than immediately dead. The suspicious node can refute the suspicion by broadcasting "I am alive" messages. Only if the suspicion is not refuted within a timeout does the node get marked as dead.

This refutation mechanism requires that the suspected node can send messages to other nodes in the cluster. When node E is marked suspicious, it must be able to broadcast or multicast an "I am alive" message that reaches the nodes that suspect it. This requires bidirectional communication paths.

In Docker Desktop, a node cannot send refutation messages to other nodes because there are no communication paths. If one Docker Desktop instance somehow learned (through an external mechanism) that other instances suspected it of failure, it would have no way to send refutation messages to them. The instance would be marked as dead regardless of its actual health.

### 3.3 VXLAN Requirements vs Docker Desktop

VXLAN has perhaps the most specific and technical requirements of all the Swarm networking components. Let us examine each in detail.

**Requirement 3.3.1: VTEP IP Addresses Must Be Routable**

Each VXLAN Tunnel Endpoint must have an IP address that is reachable from all other VTEPs in the overlay network. This address appears in the outer IP header of VXLAN packets and is used by the underlay network to route packets between VTEPs. Routers in the underlay network must have routes to these VTEP addresses.

Docker Desktop's VMs have IP addresses in the 192.168.65.0/24 range assigned by VPNKit. These addresses are not routable in the underlay network for multiple reasons. First, they are not configured on any physical network interface. When a router receives a packet destined for 192.168.65.3, it has no routing table entry for that destination. The address is not advertised by any routing protocol like BGP or OSPF.

Second, even if routes were manually configured, the addresses would still be unreachable because they exist only within the hypervisor's virtualization context. The 192.168.65.3 address is a software construct inside VPNKit and the VM. It has no corresponding MAC address on the physical network, no ARP entry, and no presence in the Layer 2 domain.

Third, multiple Docker Desktop instances can have VMs with the same IP address because each VPNKit instance independently assigns addresses without coordination. If two different machines both have VMs at 192.168.65.3, these are two distinct addresses in separate virtualization contexts. They are not the same address on a shared network.

The underlay routing requirement is fundamental to VXLAN's design. VXLAN creates a Layer 2 overlay on top of Layer 3 infrastructure, but that Layer 3 infrastructure must actually work. The IP network must be able to deliver packets between VTEP addresses. Docker Desktop's architecture provides no such IP network between VMs on different machines.

**Requirement 3.3.2: Direct UDP Transmission on Port 4789**

VXLAN encapsulated packets are UDP datagrams with destination port 4789. The VTEP must be able to send these UDP datagrams directly to remote VTEP IP addresses, and the UDP stack must handle them with minimal overhead for performance reasons.

Docker Desktop blocks direct UDP transmission to remote VTEPs. When the kernel's VXLAN module on Machine A's VM creates a VXLAN packet destined for Machine B's VTEP, that packet flows through the VM's network stack to eth0. At eth0, the packet enters AF_VSOCK and is transmitted to VPNKit.

VPNKit receives this UDP packet. It parses the outer headers and sees that the destination IP is 192.168.65.4 (Machine B's VM). VPNKit has no knowledge of this address. It is not an internet address that VPNKit can reach through the host's network, and it is not a VPNKit internal service. VPNKit's routing logic has no handler for this destination, so the packet is dropped.

Even if VPNKit were modified to somehow recognize VXLAN traffic and attempt to forward it, there would be no mechanism to reach the destination VM. VPNKit runs on Machine A's host. It can create sockets and send packets through Machine A's network interface. But Machine B's VM is not reachable through the network because the VM's address is not on the network.

The architectural barrier is that AF_VSOCK provides VM-to-host communication on the same machine but cannot provide VM-to-VM communication across machines. VXLAN requires the latter, which Docker Desktop's architecture fundamentally cannot provide.

**Requirement 3.3.3: FDB Learning Requires Accurate Source IP Addresses**

The VXLAN Forwarding Database (FDB) learns MAC-to-VTEP mappings by observing the source addresses of incoming VXLAN packets. When a VXLAN packet arrives with outer source IP X and inner source MAC Y, the FDB records "MAC Y is reachable via VTEP X." Future packets destined for MAC Y will be encapsulated and sent to VTEP X.

This learning mechanism requires that the outer source IP in VXLAN packets be the actual IP address of the sending VTEP. If packets are proxied or NAT'd, the source IP will be wrong, and the FDB will learn incorrect information.

Docker Desktop breaks FDB learning in multiple ways. First, if VXLAN packets could somehow leave the VM and reach another machine, they would likely go through VPNKit which would create them from the host. The source IP would be the host's IP, not the VM's IP. The receiving VTEP would learn that MAC addresses from the sending container are reachable via the host's IP, not the VM's VTEP IP.

When the receiving container tries to send a response, it would send the response to the host's IP. But the host is not a VTEP. The host has no VXLAN infrastructure. The packet would arrive at the host's network interface with no listener to receive it. Even if the host somehow forwarded it to its own VM, that's the wrong VM—it should go to the remote machine's VM.

Second, even if source IPs were somehow preserved, the FDB would still learn incorrect information because the VM's IP addresses are non-routable. The FDB would record "MAC Y is reachable via VTEP 192.168.65.3," but this information is useless because no node can send packets to 192.168.65.3.

The FDB learning requirement is subtle but critical. VXLAN is designed as a learning protocol similar to traditional Ethernet bridging. VTEPs learn MAC locations automatically through data plane observation rather than requiring manual configuration or a centralized database. This automatic learning depends on accurate source addressing, which Docker Desktop cannot provide.

**Requirement 3.3.4: Kernel-Level Performance**

VXLAN processing must occur in kernel space for acceptable performance. The Linux kernel's VXLAN module is implemented in drivers/net/vxlan.c and operates entirely within the kernel networking stack. When a packet needs to be encapsulated, the kernel adds the VXLAN and UDP headers without copying data or crossing into userspace.

This kernel implementation achieves performance in the millions of packets per second on modern hardware. The processing is done in interrupt context or NAPI polling, with minimal per-packet overhead. Hardware offloading features allow modern NICs to perform VXLAN encapsulation and decapsulation in hardware, further improving performance.

If VXLAN processing were moved to userspace, every packet would require at least two context switches (kernel to userspace for processing, userspace to kernel for transmission). Context switching has measurable overhead, typically hundreds of nanoseconds to a few microseconds depending on system state. At millions of packets per second, this overhead accumulates to significant CPU consumption.

Docker Desktop's architecture would require userspace VXLAN processing if it tried to support cross-host overlays. Packets would flow from the kernel's VXLAN module, through AF_VSOCK to VPNKit, where VPNKit would need to examine them and make forwarding decisions, then potentially create new packets to send through the host's network. This userspace processing path would be far slower than kernel processing.

Additionally, VPNKit is written in OCaml and runs in its own process with its own memory space. Passing packet data between the kernel, through AF_VSOCK, to VPNKit's process would require memory copying that kernel networking can avoid through zero-copy techniques. This copying would further degrade performance.

The performance requirement is not just about speed but about scalability. In a production overlay network with hundreds of containers sending traffic, the aggregate packet rate can easily reach hundreds of thousands or millions of packets per second. Userspace processing cannot scale to these rates without consuming excessive CPU and adding unacceptable latency.

**Requirement 3.3.5: MTU Handling and Fragmentation Avoidance**

VXLAN adds 50 bytes of overhead to each encapsulated packet (8 bytes VXLAN header, 8 bytes UDP header, 20 bytes IP header, 14 bytes outer Ethernet header). To avoid fragmentation, the underlay network must support an MTU at least 50 bytes larger than the overlay MTU.

Docker Swarm handles this by setting the overlay network MTU to 1450 bytes by default, ensuring that encapsulated packets fit within the standard 1500-byte Ethernet MTU. The kernel's VXLAN implementation checks MTU and can fragment packets if necessary, though fragmentation is generally avoided for performance reasons.

While Docker Desktop's VM does handle MTU correctly for local VXLAN interfaces, the MTU handling would break down in any cross-host scenario because there is no underlay network connecting VMs. The entire concept of underlay MTU is meaningless when there's no underlay network.

If VPNKit tried to forward VXLAN packets, it would need to consider the MTU of the host's network interface and potentially fragment packets. But VPNKit is not designed for this kind of packet forwarding. It operates at the connection level for TCP and the session level for UDP, not at the raw packet level where MTU and fragmentation matter.

**Requirement 3.3.6: Multicast or Head-End Replication for BUM Traffic**

VXLAN requires a mechanism for handling Broadcast, Unknown unicast, and Multicast (BUM) traffic. The standard VXLAN specification uses IP multicast for this purpose. Each VXLAN is mapped to an IP multicast group, and VTEPs join the appropriate multicast groups to receive BUM traffic for their VXLANs.

Docker Swarm uses head-end replication instead of multicast. When a BUM frame needs to be sent, the source VTEP replicates the frame and sends a unicast copy to each remote VTEP in the VXLAN. This requires knowing the list of all VTEPs, which Docker Swarm distributes through its control plane.

Both multicast and head-end replication require the ability to send packets to multiple destinations. For multicast, the VTEP must join multicast groups and send to multicast addresses. For head-end replication, the VTEP must send unicast packets to each peer VTEP individually.

Docker Desktop cannot support either mechanism. For multicast, the VM would send multicast packets through AF_VSOCK to VPNKit. VPNKit has no multicast support because AF_VSOCK itself does not support multicast. Multicast is a network-level feature that requires router support for IGMP and multicast routing protocols. The AF_VSOCK channel is point-to-point between VM and host.

For head-end replication, the VM would need to send unicast packets to multiple remote VTEP addresses. We have already established that the VM cannot send packets to remote VTEP addresses through Docker Desktop's architecture. Each packet in the head-end replication would fail for the same reasons: destination address is non-routable, no path through AF_VSOCK/VPNKit to remote machines.

---

## Part IV: Detailed Component Analysis

To fully understand the architectural barriers, we must examine each Docker Desktop component in greater technical detail, exploring not just what they do but how they are fundamentally constrained by their design and implementation.

### 4.1 AF_VSOCK: The VM-Host Communication Primitive

AF_VSOCK is a socket address family defined by the VMware VMCI (Virtual Machine Communication Interface) specification and later adopted into the Linux kernel. Understanding its implementation reveals why it cannot be extended to support cross-host communication.

**Implementation Architecture:**

AF_VSOCK operates through a combination of kernel modules and shared memory regions. On the host side, the vhost-vsock kernel module manages the host endpoint. On the VM side, the virtio_vsock module manages the VM endpoint. These modules communicate through shared memory buffers that the hypervisor allocates in the host's physical RAM.

The shared memory architecture is fundamental to understanding AF_VSOCK's limitations. When the VM wants to send data to the host, it writes to a circular buffer in shared memory and uses an event notification mechanism (typicallyioeventfd) to signal the host. The host's vhost-vsock module receives the notification, reads from the shared memory buffer, and delivers the data to the receiving application.

This communication path never involves the host's network stack, network interfaces, or routing tables. Packets do not flow through IP routing, do not have Ethernet headers, and are not subject to firewall rules. The communication is purely between two endpoints that share physical memory.

**Address Space and Scope:**

AF_VSOCK uses Context IDs (CIDs) as addresses instead of IP addresses. The CID space is scoped to a single hypervisor. The hypervisor assigns CIDs to VMs when they are created. The assignment is arbitrary and local—there is no global CID allocation authority.

The special CID values are defined in the kernel:
- CID 0: Reserved
- CID 1: Local loopback (VM communicating with itself)
- CID 2: Host (VMADDR_CID_HOST)
- CID 3 and above: VMs

When a VM with CID 3 connects to CID 2, it is connecting to its hypervisor. There is no concept in the AF_VSOCK specification or implementation of a CID on a remote machine. The protocol has no provisions for routing, for discovering remote endpoints, or for tunneling over IP networks.

**Why Extension Is Impossible:**

One might ask whether AF_VSOCK could be extended to support cross-host communication. This would require several capabilities that are fundamentally incompatible with its design:

First, a global CID registry would be needed to ensure uniqueness across all machines. Currently, CIDs are assigned locally and can collide across machines. Making CIDs globally unique would require a distributed coordination system, essentially reinventing IP address allocation.

Second, a routing mechanism would be needed to direct packets to remote machines. Currently, the kernel modules directly write to shared memory. For remote communication, packets would need to be serialized, wrapped in some transport protocol (likely TCP or UDP), sent over the network, received on the remote side, deserialized, and delivered to the target VM. This is exactly what IP networking already does.

Third, discovery mechanisms would be needed so VMs could find each other. Currently, VMs only know about their local hypervisor. For remote communication, they would need to discover which CIDs exist on which machines and how to reach them. This is essentially service discovery, which distributed systems typically solve with DNS, service registries, or gossip protocols—none of which AF_VSOCK provides.

The moment you add global CID allocation, routing, and discovery to AF_VSOCK, you have reinvented IP networking. At that point, there is no advantage to using AF_VSOCK over just using IP. The simplicity of AF_VSOCK—its core value proposition—would be lost.

**Performance Characteristics:**

AF_VSOCK achieves excellent performance for VM-host communication because it avoids the network stack entirely. A socket write in the VM directly places data in shared memory that the host can read. There are no protocol headers to add, no checksums to compute, no routing lookups to perform, and no packet buffers to allocate.

Measurements show AF_VSOCK throughput approaching memory bandwidth limits and latency in the low microseconds. This is orders of magnitude better than network-based VM communication which must go through virtual NICs, software bridges, and the network stack.

However, this performance comes at the cost of scope. The shared memory that provides high performance exists only on one machine. Extending the communication to other machines would require using the network, at which point the performance advantages disappear and you might as well use regular networking.

### 4.2 VPNKit: The Connection Recreation Engine

VPNKit is implemented in OCaml using libraries from the MirageOS unikernel project. Understanding its architecture reveals why it cannot handle peer-to-peer VXLAN traffic.

**Protocol Stack Implementation:**

VPNKit implements a complete TCP/IP network stack in userspace. The stack includes:
- Ethernet frame parsing (mirage-tcpip/ethernet)
- ARP protocol handling (mirage/arp)
- IPv4 and IPv6 processing (mirage-tcpip/ipv4, ipv6)
- TCP protocol with connection tracking (mirage-tcpip/tcp)
- UDP session management (mirage-tcpip/udp)
- ICMP for ping and error messages
- DNS server for name resolution
- DHCP server for address assignment

Each layer is implemented as a functional module that processes packets and passes them to the next layer. This modular design is elegant and maintainable but necessarily operates in userspace with all the attendant performance costs.

**Connection Recreation Model:**

VPNKit does not route or NAT packets. Instead, it recreates connections. This distinction is crucial. When a container initiates a TCP connection to google.com, here's the full sequence:

1. Container creates a socket and calls connect() with Google's IP and port 443
2. VM kernel generates a SYN packet and sends it out eth0
3. eth0 driver serializes the Ethernet frame and writes it to AF_VSOCK
4. VPNKit receives the frame on the host
5. VPNKit parses: Ethernet → IP → TCP → identifies SYN packet
6. VPNKit creates a new TCP socket on the host using the standard socket API
7. VPNKit calls connect() to Google's IP:443 from the host
8. If Google's server accepts the connection, VPNKit receives a SYN-ACK from Google
9. VPNKit crafts a synthetic SYN-ACK packet with appropriate sequence numbers
10. VPNKit sends this synthetic packet through AF_VSOCK to the VM
11. VM's TCP stack receives the SYN-ACK and completes the connection setup
12. Container's connect() call succeeds

At this point, VPNKit maintains two completely separate TCP connections:
- Connection A: VM to VPNKit (virtual connection in VPNKit's memory)
- Connection B: VPNKit to Google (real connection via host socket)

VPNKit proxies data between these connections. When the container writes application data, it flows:
1. Container → VM TCP → VPNKit connection A
2. VPNKit reads from connection A
3. VPNKit writes to connection B
4. Connection B transmits to Google

Responses flow in reverse. VPNKit continuously reads from both connections and shuttles data between them until one side closes.

**Why This Model Breaks for VXLAN:**

The connection recreation model works for client-server protocols where the VM is always the client. It completely breaks for peer-to-peer protocols like VXLAN. Let's examine why:

First, VXLAN uses UDP, not TCP. While VPNKit can handle UDP, it expects request-response patterns like DNS queries. VXLAN traffic is bidirectional with no clear request-response structure. Both endpoints send and receive continuously.

Second, VPNKit expects destination addresses to be internet addresses. For each outbound packet, VPNKit checks if the destination is an internet address (not RFC 1918 private, not link-local, not multicast). For internet addresses, it creates a socket and sends through the host's network. For VXLAN packets destined for other VMs at addresses like 192.168.65.4, VPNKit has no handler.

Third, VPNKit cannot receive unsolicited inbound UDP packets. All the UDP sessions it manages start with an outbound packet from the VM. If a remote VTEP tried to send a VXLAN packet to a Docker Desktop VM, that packet would arrive at the host's network interface. There's no listener on the host's UDP port 4789 to receive it. The host's OS would drop it as unreachable.

Fourth, even if VPNKit could somehow recognize VXLAN traffic and knew that addresses like 192.168.65.x refer to VMs on other machines, it has no mechanism to locate those machines. VPNKit is not a service discovery system. It doesn't maintain a registry of peers. It doesn't implement any peer-to-peer networking protocol.

Fifth, the performance would be unacceptable. VXLAN overlay networks can generate millions of packets per second in aggregate. Every packet would need to traverse the full userspace path: kernel → AF_VSOCK → VPNKit process → parsing → routing decision → socket operations → kernel. The context switching and protocol processing overhead would overwhelm the system.

**VPNKit's Strengths Within Its Design Scope:**

It's important to recognize that VPNKit excellently solves the problem it was designed for. Corporate VPNs frequently block VM traffic by detecting characteristics like TTL changes, MAC addresses, or routing patterns. VPNKit makes VM traffic indistinguishable from native host traffic because it truly is native host traffic—created by a regular host process using normal sockets.

For developers working on corporate networks, VPNKit provides internet access to containers that would otherwise be blocked. It handles all the common protocols (HTTP, HTTPS, SSH, git, etc.) transparently. The fact that it cannot handle peer-to-peer overlay networks is not a bug in VPNKit—it's outside VPNKit's design scope.

### 4.3 Port Forwarding: The One-Way Proxy

Docker Desktop's port forwarding mechanism deserves detailed examination because it represents the closest thing to bidirectional communication in the architecture, yet still cannot solve the Swarm connectivity problem.

**Architecture and Data Flow:**

Port forwarding involves multiple components working together:

1. **Docker Engine in VM**: When a container is started with `-p 8080:80`, Docker Engine creates a port forwarding rule in its internal state
2. **Port forwarding request**: Docker Engine sends a request through a communication channel (variously implemented as 9P filesystem writes or HTTP API calls over AF_VSOCK) to the host
3. **Docker Desktop backend**: The com.docker.backend process on the host receives the request
4. **Validation**: The backend checks if port 8080 is available on the host
5. **Listener creation**: If available, the backend creates a TCP listener on 0.0.0.0:8080 or 127.0.0.1:8080
6. **Connection handling**: When a client connects, the backend accepts the connection on the host side

When a connection arrives:
1. Client connects to host:8080
2. Backend accepts the connection (call this socket S1)
3. Backend initiates a connection through AF_VSOCK to the VM
4. vpnkit-forwarder in the VM receives the connection request
5. vpnkit-forwarder opens a connection to container IP:80 (call this socket S2)
6. Backend proxies data: read from S1, write to S2 and vice versa

**The Two-Connection Model:**

This creates two independent TCP connections with separate sequence number spaces, separate flow control, and separate state. From the container's perspective, the connection originates from the VM's internal network. When the container application calls getpeername(), it receives an address like 172.17.0.1 (the docker0 bridge) or 192.168.65.1 (the VM's gateway), not the original client's address.

The backend may attach metadata indicating the original source IP, and some container runtimes can access this metadata for logging. But the container's application cannot access it through standard socket APIs. For protocols that require accurate peer identification, this is fatal.

**Why It Cannot Solve Swarm Connectivity:**

Port forwarding fails to solve Swarm connectivity for several reasons:

First, it only publishes container ports, not Docker Engine ports. The mechanism is triggered when containers are created with the `-p` flag. There is no equivalent mechanism for publishing dockerd's own listening ports. Swarm's port 2377 is bound by dockerd, not by a container.

Second, even if we could publish Engine ports, the source IP loss would break Swarm. When a remote node connects to a manager via forwarded port 2377, the manager would see all connections as coming from the same internal address. It could not distinguish between different remote nodes. The Raft protocol requires accurate node identification based on source IP.

Third, port forwarding is unidirectional. It allows inbound connections from external clients to containers. It does not provide a mechanism for containers to establish outbound connections to specific remote forwarded ports. Swarm requires bidirectional connectivity where any node can connect to any other node in either direction.

Fourth, UDP port forwarding has additional complications. TCP connections have clear lifecycle (SYN, data transfer, FIN). UDP has no such markers. The port forwarding system must use heuristics like tracking (source IP, source port, destination IP, destination port) tuples and implementing session timeouts. For high-rate bidirectional UDP like VXLAN port 4789, these heuristics become unreliable.

Fifth, the port forwarding system cannot publish the same port on multiple containers. When you run `-p 2377:2377`, that binds the host's port 2377. You cannot run another container with `-p 2377:2377` because the port is already in use. This means you cannot run multiple manager nodes on the same machine even if they're in different containers.

### 4.4 Non-Routable IP: The Addressing Conundrum

The VM's IP address of 192.168.65.3 exists in VPNKit's private address space and DHCP configuration but nowhere else. Let's examine exactly why this address is non-routable.

**Address Assignment:**

When the VM boots, its network interface (eth0) is configured via DHCP. The DHCP client in the VM (typically dhclient or systemd-networkd) broadcasts a DHCP discover message. This broadcast goes through AF_VSOCK to VPNKit. VPNKit's embedded DHCP server (implemented using the charrua OCaml library) responds with a DHCP offer.

The offer contains:
- IP address: 192.168.65.3
- Netmask: 255.255.255.0 (192.168.65.0/24)
- Gateway: 192.168.65.1
- DNS server: 192.168.65.1

The VM accepts this configuration and sets up its network interface accordingly. From the VM kernel's perspective, it has a valid IP configuration. `ip addr show eth0` returns a properly configured interface with an IP address, netmask, and gateway.

**Where the Address Exists:**

The address 192.168.65.3 exists in exactly three places:
1. VM kernel's network interface configuration
2. VPNKit's internal state tracking which VM has which address
3. Nowhere else

The address does NOT exist:
- On any interface on the host OS
- In the host's routing table
- In the physical network's routing infrastructure
- In any ARP tables on the physical network
- In any DNS records
- In any routing protocol advertisements

**Routing Table Analysis:**

Let's trace what happens when another machine tries to route to 192.168.65.3. Suppose Machine B wants to send a packet to this address. Machine B's kernel performs a routing table lookup:

```
$ ip route show
default via 192.168.1.1 dev eth0
192.168.1.0/24 dev eth0 proto kernel scope link
```

The lookup checks if 192.168.65.3 matches any route. It doesn't match 192.168.1.0/24. It falls to the default route pointing to 192.168.1.1 (the router). The packet is forwarded to the router.

The router performs its own lookup:
```
Router routing table (conceptual):
10.0.0.0/8 → Internal network
172.16.0.0/12 → Partner network
192.168.1.0/24 → Local LAN
default → Internet gateway
```

The address 192.168.65.3 matches none of these specific routes. It falls to the default route toward the internet. The router forwards it to its ISP. The ISP's router sees that 192.168.65.0/24 is a private address (RFC 1918) and is not routable on the internet. The packet is dropped.

**ARP Failure on Local Network:**

Even if Machine B and Machine A (running Docker Desktop) are on the same local 192.168.1.0/24 network, connectivity fails. When Machine B tries to send to 192.168.65.3, it must first resolve the IP to a MAC address using ARP.

Machine B broadcasts: "Who has 192.168.65.3? Tell 192.168.1.100"

Machine A's host network interface receives this ARP broadcast but does not respond. Why? Because Machine A's host does not have IP address 192.168.65.3. That address belongs to the VM, and the VM's eth0 is not connected to the physical network.

The VM does have the address, but it's not on a physical network segment. It's on a virtual network mediated by AF_VSOCK. The VM can't receive the ARP broadcast because broadcasts don't cross the AF_VSOCK boundary.

With no ARP reply, Machine B cannot construct an Ethernet frame to send to 192.168.65.3. The connection attempt fails even though both machines are on the same physical network.

**Why Using Host IP Doesn't Work:**

One might propose that Docker Engine advertise the host's IP (192.168.1.100) instead of the VM's IP. This fails because Docker Engine can only bind sockets to IP addresses that exist on network interfaces in its network namespace.

Inside the VM, `ip addr show` would display:
```
eth0: 192.168.65.3/24
lo: 127.0.0.1/8
```

The address 192.168.1.100 is not in this list. When Docker Engine tries `bind(socket, 192.168.1.100:2377)`, the kernel checks if 192.168.1.100 is configured on any interface. Finding no such interface, it returns EADDRNOTAVAIL (Cannot assign requested address).

The bind() system call is fundamental to how servers work. A server must bind to a specific address and port to listen for connections. If it cannot bind to the address it advertises, the entire architecture collapses.

---

## Part V: Synthesis and Conclusions

Having examined the protocols and architecture components in detail, we can now synthesize our understanding into a comprehensive picture of why Docker Desktop fundamentally cannot support multi-node Docker Swarm.

### 5.1 The Architectural Mismatch

Docker Swarm requires a full peer-to-peer network mesh where nodes can establish direct connections to each other. Docker Desktop provides a client-to-internet communication model where VMs can access external services but cannot communicate with peer VMs.

This is not a superficial incompatibility that could be fixed with configuration changes or minor software updates. It is a fundamental architectural mismatch between what distributed systems require and what single-machine developer tools provide.

The mismatch exists at multiple layers:

**At the network topology layer:** Swarm requires a flat network where all nodes are peers. Docker Desktop provides a hierarchical model where VMs are children of their host with no sibling relationships.

**At the addressing layer:** Swarm requires globally unique, routable addresses. Docker Desktop provides locally scoped addresses that are meaningful only within one machine's virtualization context.

**At the communication layer:** Swarm requires bidirectional peer-to-peer communication. Docker Desktop provides unidirectional VM-to-internet communication mediated by proxies.

**At the protocol layer:** Swarm requires direct UDP/TCP communication with source IP preservation. Docker Desktop provides connection recreation that loses source IPs and cannot efficiently handle high packet rates.

### 5.2 Why Each Blocker Is Necessary

It's important to understand that each architectural component that blocks Swarm exists for good reasons within Docker Desktop's design goals:

**AF_VSOCK provides:**
- Simple, high-performance VM-host communication without network configuration
- Isolation of the VM from the physical network for security
- Guaranteed connectivity regardless of host network state
- Compatibility with different hypervisors through standard interfaces

**VPNKit provides:**
- Internet access for containers even through restrictive corporate VPNs
- Traffic that appears as legitimate host traffic, not as VM traffic
- Compatibility with host proxy configurations and network policies
- Isolation from host network changes

**Port forwarding provides:**
- Convenient access to container services from the host
- Simple developer experience with `docker run -p`
- Integration with the host's localhost for familiar workflows

**Non-routable IPs provide:**
- Isolation of VM networking from host networking
- No requirement for administrator privileges to configure networking
- No conflicts with host network addressing
- Consistent experience regardless of host network configuration

Each component optimizes for Docker Desktop's primary use case: local development on a single machine. The components work together to provide an experience that just works out of the box without network configuration, without administrator privileges, and without understanding virtualization or networking concepts.

### 5.3 The Impossibility of Hybrid Solutions

One might propose hybrid solutions that try to get the best of both worlds. For example:
- "Add bridged networking as an option while keeping AF_VSOCK for other features"
- "Make VPNKit learn about peer VMs and forward traffic between them"
- "Use a VPN tunnel between Docker Desktop instances to connect the VMs"

Each of these faces fundamental problems:

**Bridged networking** would break VPN compatibility (the primary reason VPNKit exists), require administrator privileges (eliminating ease of installation), and expose the VM to the physical network (reducing security isolation). More fundamentally, it would require rebuilding Docker Desktop's entire networking architecture.

**Enhanced VPNKit** would need to implement peer discovery, connection routing, source IP preservation, efficient high-rate packet forwarding, and bidirectional connection support. At this point, you've reimplemented kernel networking in userspace with none of the performance benefits and all of the complexity.

**VPN tunnels between instances** would require manual configuration (eliminating ease of use), add encryption overhead (reducing performance), require stable addressing or dynamic DNS (adding complexity), and still potentially lose source IPs depending on implementation. This essentially admits that Docker Desktop can't do it natively and requires building a separate distributed system on top.

### 5.4 Appropriate Tools for Different Phases

The solution is not to make Docker Desktop do everything but to recognize that different tools serve different purposes in the container workflow:

**Docker Desktop is appropriate for:**
- Local development
- Running single containers
- Testing Docker Compose multi-container applications
- Building images
- Learning Docker concepts
- Development environments where production-like distribution isn't needed

**Multi-node infrastructure is appropriate for:**
- Testing distributed behavior
- Validating high availability
- Load testing across multiple nodes
- Production deployments
- Staging environments that mirror production

**Recommended approaches for multi-node:**
- Cloud VMs (AWS, GCP, Azure, DigitalOcean) with Docker Engine
- Local VMs with bridged networking (VirtualBox, VMware, Hyper-V)
- WSL2 + Docker Engine + Tailscale for Windows users
- Kubernetes for production orchestration
- Managed services (ECS, AKS, GKE) for production

### 5.5 Educational Value of Understanding the Limitation

Understanding why Docker Desktop can't support multi-node Swarm provides valuable lessons about distributed systems and software architecture:

**Design decisions have trade-offs:** Docker Desktop's choices optimize for local development at the expense of distributed capabilities. This is not a bug; it's an intentional trade-off.

**Protocols have specific requirements:** Raft, SWIM, and VXLAN were designed with specific assumptions about network connectivity. Violating these assumptions breaks the protocols.

**Abstraction layers have limits:** AF_VSOCK abstracts VM-host communication elegantly for its intended purpose but cannot be abstracted further without losing its benefits.

**Userspace vs kernel matters:** For high-performance networking, kernel implementation provides orders of magnitude better performance than userspace alternatives.

**Addressing and routing are foundational:** Non-routable IPs are not just inconvenient; they make entire classes of distributed systems impossible.

---

## Conclusion: Fundamental Incompatibility

Docker Desktop cannot support multi-node Docker Swarm not because of missing features or incomplete implementation, but because of fundamental architectural choices that optimize for different goals than distributed systems require.

The protocols (Raft, SWIM, VXLAN) require direct peer-to-peer communication with routable addresses and source IP preservation. The architecture (AF_VSOCK, VPNKit, port forwarding, non-routable IPs) provides VM-to-internet communication through proxies with local addresses.

These requirements and capabilities are mutually exclusive. Making Docker Desktop support multi-node Swarm would require abandoning the architectural choices that make it work well for local development. The result would be a different product serving a different purpose.

For users who need multi-node orchestration, the path forward is using appropriate infrastructure: cloud VMs, local VMs with bridged networking, or managed orchestration services. Docker Desktop remains an excellent tool for its intended purpose: providing a simple, reliable container development environment on Windows and macOS.

---

**END OF COMPREHENSIVE TECHNICAL STUDY**

*This document has analyzed Docker Swarm and VXLAN protocols from first principles, examined Docker Desktop's architectural components in detail, and mapped each protocol requirement to the specific architectural limitation that blocks it. The analysis demonstrates that multi-node Swarm on Docker Desktop is not merely unsupported but technically impossible without fundamental architecture changes that would eliminate Docker Desktop's core value proposition.*
