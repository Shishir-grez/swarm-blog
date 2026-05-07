# Practical Verification Guide: Empirical Testing of Docker Swarm Limitations on Docker Desktop
## A Hands-On Companion to the Comprehensive Protocol Study

**Document Purpose:** This guide provides step-by-step procedures to empirically verify every major claim made in the comprehensive protocol study. Rather than accepting theoretical explanations, you will observe the actual behavior of Docker Desktop, capture real packet traces, examine actual routing tables, and witness protocol failures in real-time.

**Prerequisite:** Read the comprehensive protocol study first to understand what we're testing and why each test matters.

**Testing Philosophy:** We adopt a scientific approach - make predictions based on theory, design experiments to test those predictions, collect data, and draw conclusions. Each test is designed to isolate one specific claim and demonstrate it conclusively.

---

## Part I: Environment Setup and Baseline Verification

Before we can test failures, we must establish that our environment is correctly configured and that successful operations work as expected. This baseline verification ensures that when things fail later, we know the failure is meaningful rather than due to misconfiguration.

### Test 1.1: Verify Docker Desktop is Running Correctly

**Theoretical Claim:** Docker Desktop runs Docker Engine inside a Linux VM that is isolated from the physical network but can access the internet through VPNKit.

**Test Procedure:**

First, let us verify that Docker Desktop is functioning normally with a simple container that accesses the internet. Open a terminal and execute this sequence:

```bash
# Pull a standard image (this tests internet connectivity)
docker pull alpine:latest

# Run a container and verify internet access
docker run --rm alpine:latest sh -c "ping -c 3 8.8.8.8 && echo 'Internet connectivity works'"

# Run a container and verify DNS resolution
docker run --rm alpine:latest sh -c "nslookup google.com && echo 'DNS resolution works'"
```

**Expected Results:** Both commands should succeed. The ping should show successful responses from Google's DNS server, and nslookup should resolve google.com to IP addresses. This demonstrates that VPNKit is successfully proxying internet traffic from the VM.

**What This Proves:** Docker Desktop's basic networking functions correctly for the VM-to-internet use case it was designed for. Any failures we see later are not due to broken networking in general, but rather due to specific limitations in cross-host scenarios.

**Detailed Observation:** Notice that the ping succeeds even though the container has a private IP address like 172.17.0.2. This is VPNKit at work - it receives the ping packets from the VM, creates corresponding ICMP requests from the host, receives responses, and crafts synthetic ICMP replies back to the container. The container has no idea this proxying is happening.

### Test 1.2: Access the Docker Desktop VM Shell

**Theoretical Claim:** Docker Desktop's VM is accessible for diagnostic purposes but is isolated from normal system access.

**Test Procedure:**

The method to access the VM varies by platform. Let me provide instructions for each:

On Windows with WSL2 backend:
```bash
# From PowerShell or Windows Terminal
wsl -d docker-desktop

# You should now be inside the VM
# Verify with:
uname -a
# Should show: Linux docker-desktop ...
```

On macOS:
```bash
# Method 1: Using screen (older versions)
screen ~/Library/Containers/com.docker.docker/Data/vms/0/tty

# Method 2: Using docker run with privileged access (works on all platforms)
docker run -it --rm --privileged --pid=host debian nsenter -t 1 -m -u -n -i sh

# You should now be inside the VM
# Verify with:
cat /etc/os-release
# Should show Alpine Linux or similar minimal distribution
```

On Windows with Hyper-V backend:
```powershell
# Direct access is more limited, use the privileged container method:
docker run -it --rm --privileged --pid=host debian nsenter -t 1 -m -u -n -i sh
```

**Expected Results:** You should gain shell access to the VM and see a minimal Linux environment. The uname or os-release output should confirm you're in a Linux VM.

**What This Proves:** We can access the VM to run diagnostic commands, examine network configuration, and observe system behavior directly. This access is essential for all subsequent tests.

**Important Note:** Once inside the VM, install diagnostic tools if needed:
```bash
# Install network diagnostic tools (in the VM shell)
apk update
apk add tcpdump iproute2 bridge-utils iputils bind-tools ethtool net-tools curl netcat-openbsd iperf3
```

### Test 1.3: Examine VM Network Configuration

**Theoretical Claim:** The VM has a non-routable IP address in the 192.168.65.0/24 range, with eth0 connected to AF_VSOCK rather than a traditional virtual NIC.

**Test Procedure:**

Inside the VM shell, execute this series of commands to examine the network configuration:

```bash
# Show all network interfaces with detailed information
ip addr show

# Show routing table
ip route show

# Show default gateway
ip route | grep default

# Check eth0 driver type
ethtool -i eth0

# Show ARP table
ip neigh show

# Check for vsock devices
ls -la /dev/vsock* 2>/dev/null || echo "No vsock devices visible"
ls -la /dev/vhost* 2>/dev/null || echo "No vhost devices visible"

# Check loaded kernel modules related to networking
lsmod | grep -E 'virtio|vsock|vhost'

# Examine network namespaces
ip netns list
```

**Expected Results Analysis:**

When you run `ip addr show`, you should see output similar to:
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    inet 127.0.0.1/8 scope host lo
    
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    inet 192.168.65.3/24 brd 192.168.65.255 scope global eth0
```

The key observations here are critical. Notice that eth0 has IP address 192.168.65.3 with netmask /24, meaning the VM believes it is on a network with addresses from 192.168.65.0 to 192.168.65.255. This is VPNKit's assigned address space.

When you run `ip route show`, expect to see:
```
default via 192.168.65.1 dev eth0
192.168.65.0/24 dev eth0 proto kernel scope link src 192.168.65.3
```

This tells us two crucial things. First, the default route points to 192.168.65.1, which is VPNKit acting as the gateway. All traffic that doesn't match a more specific route will go through this gateway. Second, the VM believes the entire 192.168.65.0/24 network is directly attached to eth0.

When you run `ethtool -i eth0`, you should see:
```
driver: virtio_net
version: 1.0.0
firmware-version:
expansion-rom-version:
bus-info: virtio0
```

The critical detail is `driver: virtio_net`. This indicates that eth0 is using the virtio network driver, which is the virtualization-aware driver used for communication between guest and host. This is the adapter that connects to AF_VSOCK rather than to a traditional bridged network.

**What This Proves:** 
- The VM has a non-routable IP in VPNKit's private address space
- All traffic routes through VPNKit's gateway at 192.168.65.1
- The eth0 interface uses virtio, indicating hypervisor communication rather than physical network access
- This confirms the architectural claim that the VM's network is isolated from the physical network

**Additional Verification:**

Try to determine if this IP address exists anywhere on the host. Exit the VM shell and run on your host machine:

On macOS or Linux:
```bash
ip addr show | grep 192.168.65
# or
ifconfig | grep 192.168.65
```

On Windows:
```powershell
ipconfig | Select-String "192.168.65"
```

**Expected Result:** The host has no interface with the VM's IP address. The address exists only inside the VM's network namespace, not on the host's network stack.

### Test 1.4: Verify Single-Node Swarm Works

**Theoretical Claim:** Docker Swarm works perfectly in single-node mode on Docker Desktop because all communication is local.

**Test Procedure:**

Initialize a Swarm on Docker Desktop:

```bash
# Initialize Swarm (this should succeed)
docker swarm init

# Check Swarm status
docker info | grep -A 10 Swarm

# List nodes
docker node ls

# Create a simple service
docker service create --name test-nginx --replicas 3 nginx:alpine

# Watch the service deploy
docker service ps test-nginx

# Check service status
docker service ls
```

**Expected Results:** All commands should succeed. You should see one node (the current machine) in manager status. The service should deploy successfully with three replicas running on the single node.

**What This Proves:** Swarm's mechanisms (Raft, gossip, overlay networks) all function correctly when operating on a single machine. The protocols work - it's only cross-host communication that fails.

**Detailed Analysis:** When you run `docker node ls`, examine the advertised address:

```bash
docker node inspect self --format '{{ .Status.Addr }}'
```

This will likely show 192.168.65.3 (the VM's IP). For single-node Swarm, this doesn't matter because the node only communicates with itself. But this address would be what the node advertises to other nodes if they tried to join, and we'll prove that other nodes cannot reach this address.

**Cleanup:**
```bash
docker service rm test-nginx
docker swarm leave --force
```

---

## Part II: Testing Raft Consensus Layer Requirements

Now we begin systematic testing of each protocol requirement. We'll demonstrate that while the protocols function correctly in principle, the Docker Desktop architecture prevents them from working across hosts.

### Test 2.1: TCP Connectivity to Swarm Port 2377

**Theoretical Claim:** The Raft port (2377) is bound correctly inside the VM but is not reachable from external machines due to non-routable IPs and lack of direct network access.

**Test Procedure:**

First, initialize Swarm again and verify the port is listening:

```bash
# Initialize Swarm
docker swarm init

# Access the VM shell
docker run -it --rm --privileged --pid=host debian nsenter -t 1 -m -u -n -i sh

# Inside VM, check what's listening on port 2377
netstat -tulpn | grep 2377
# or with ss:
ss -tulpn | grep 2377
```

**Expected Output:**
```
tcp    0    0 192.168.65.3:2377    0.0.0.0:*    LISTEN    12345/dockerd
```

This shows dockerd is listening on 192.168.65.3:2377. Now let's test connectivity:

**Test 2.1.1 - Local Connectivity Within VM:**
```bash
# Still inside VM
# Test connection to localhost
nc -zv 127.0.0.1 2377
# Should succeed

# Test connection to VM's own IP
nc -zv 192.168.65.3 2377
# Should succeed
```

**What This Proves:** The port is correctly bound and accessible locally. The Raft service is working.

**Test 2.1.2 - Connectivity From Host:**

Exit the VM and try to connect from the host:

```bash
# On macOS/Linux host:
nc -zv localhost 2377
# Expected: Connection refused (nothing is listening on host's port 2377)

# Try the VM's IP from the host
nc -zv 192.168.65.3 2377
# Expected: Connection timeout or "No route to host"
```

On Windows host:
```powershell
Test-NetConnection -ComputerName localhost -Port 2377
# Expected: Failed
```

**What This Proves:** Even though dockerd is listening on port 2377 inside the VM, that port is not accessible from the host. This is because port forwarding only works for container ports (specified with -p), not for Docker Engine's own ports.

**Test 2.1.3 - Simulate Remote Machine Attempt:**

Since we only have one Docker Desktop instance, we'll simulate what would happen if another machine tried to connect. From the host:

```bash
# Try to reach what would be another VM's address
ping -c 3 192.168.65.10
# Expected: 100% packet loss

# Try to connect to a hypothetical second manager
nc -zv -w 2 192.168.65.10 2377
# Expected: Connection timeout
```

**What This Proves:** Addresses in the 192.168.65.0/24 range are not reachable from the host or from the network. Even if another Docker Desktop instance existed on another machine with a VM at 192.168.65.4, it would be unreachable because these addresses exist in separate, isolated virtualization contexts.

**Deep Dive - Packet Tracing:**

Let's trace what actually happens to packets. Inside the VM:

```bash
# Start a packet capture on eth0
tcpdump -i eth0 -n 'port 2377' -w /tmp/swarm_packets.pcap &
TCPDUMP_PID=$!

# Try to connect to a non-existent peer VM
nc -zv -w 2 192.168.65.10 2377

# Stop tcpdump
kill $TCPDUMP_PID

# Examine the capture
tcpdump -r /tmp/swarm_packets.pcap -n -v
```

**Expected Observation:** You'll see SYN packets being sent to 192.168.65.10:2377, but no SYN-ACK responses. The packets are being sent out eth0, but they're reaching VPNKit which has no handler for that destination address. VPNKit silently drops them, and the connection times out.

### Test 2.2: Routing Table Analysis and Packet Flow

**Theoretical Claim:** All outbound traffic from the VM goes through the default gateway (VPNKit) which cannot route to other VMs.

**Test Procedure:**

Inside the VM, let's trace the routing decision for various destinations:

```bash
# Show full routing table with all details
ip route show table all

# Trace route to internet (should work)
ip route get 8.8.8.8

# Trace route to local network IP (your actual LAN IP - adjust as needed)
ip route get 192.168.1.100

# Trace route to another hypothetical VM
ip route get 192.168.65.10

# Trace route to a Docker Desktop instance on another machine
# (use your actual network setup if testing with multiple machines)
ip route get 192.168.1.200
```

**Expected Results for Each:**

For `ip route get 8.8.8.8`:
```
8.8.8.8 via 192.168.65.1 dev eth0 src 192.168.65.3
```
This shows internet traffic goes through VPNKit (192.168.65.1), which is correct and will work.

For `ip route get 192.168.1.100`:
```
192.168.1.100 via 192.168.65.1 dev eth0 src 192.168.65.3
```
This shows that even traffic to the host's local network goes through VPNKit. VPNKit will drop this because 192.168.1.100 is not an internet address and VPNKit has no handler for it.

For `ip route get 192.168.65.10`:
```
192.168.65.10 dev eth0 src 192.168.65.3
```
This shows the VM thinks 192.168.65.10 is on the local network segment. It will try to send packets directly through eth0, but that eth0 connects to AF_VSOCK, not to a network where other VMs exist.

**What This Proves:** The routing table directs all traffic through VPNKit or directly to eth0 (which also leads to VPNKit/AF_VSOCK). There is no routing path to other physical machines or their VMs.

**Advanced Test - ARP Behavior:**

Let's examine ARP (Address Resolution Protocol) behavior to see how the VM tries to find MAC addresses:

```bash
# Clear ARP cache
ip neigh flush all

# Show current ARP table (should be mostly empty)
ip neigh show

# Try to ping another hypothetical VM address
ping -c 1 -W 1 192.168.65.10

# Check ARP table again
ip neigh show

# Look specifically for the attempted address
ip neigh show 192.168.65.10
```

**Expected Observation:** You might see an entry like:
```
192.168.65.10 dev eth0 FAILED
```

This indicates the VM tried to use ARP to find the MAC address for 192.168.65.10 but got no response. This is because:
1. The address doesn't exist on the "network" (really AF_VSOCK)
2. VPNKit doesn't respond to ARP for addresses it doesn't manage
3. There is no actual Ethernet segment connecting VMs

**What This Proves:** Layer 2 (Ethernet/ARP) communication doesn't work between VMs because they're not on the same Layer 2 network segment. They're connected through AF_VSOCK, which is not an Ethernet network.

### Test 2.3: Source IP Preservation Test

**Theoretical Claim:** Docker Desktop's port forwarding mechanism loses source IP information, breaking Raft's node identification.

**Test Procedure:**

We'll create a simple server inside a container that reports the source IP of connections, then connect to it through Docker Desktop's port forwarding to observe the source IP loss.

```bash
# Create a server container that reports connection details
docker run -d --name source-ip-test -p 8888:8888 alpine:latest sh -c '
while true; do
  nc -l -p 8888 -e sh -c "echo \"Connection received from: \$(echo \$NCAT_REMOTE_ADDR:\$NCAT_REMOTE_PORT)\" && date"
done'

# Alternative using Python for more details:
docker run -d --name source-ip-test -p 8888:8888 python:3-alpine sh -c '
import socket
import datetime
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.bind(("0.0.0.0", 8888))
s.listen(5)
while True:
    conn, addr = s.accept()
    msg = f"Connection from {addr[0]}:{addr[1]} at {datetime.datetime.now()}\\n"
    print(msg)
    conn.send(msg.encode())
    conn.close()
'

# Connect from your host
# On macOS/Linux:
echo "Test connection" | nc localhost 8888

# On Windows PowerShell:
Test-NetConnection -ComputerName localhost -Port 8888

# Check the container logs to see what source IP it observed
docker logs source-ip-test
```

**Expected Observation:**

The container logs will show something like:
```
Connection from 172.17.0.1:54321 at 2026-02-04 12:34:56
```

The source IP is **172.17.0.1**, which is the docker0 bridge IP or the VM's internal gateway IP. It is **NOT** your host's actual IP address, nor your external IP if connecting from another machine.

**What This Proves:** Port forwarding creates two separate connections (host-to-container, and the original client-to-host), and the container only sees the proxied connection. The original source IP is lost.

**Implications for Raft:** If Swarm manager nodes received connections through this port forwarding mechanism, they would all appear to come from the same proxy address. The managers could not distinguish between different remote nodes. Raft's node tracking would break because it relies on source IPs to identify which node is which.

**Cleanup:**
```bash
docker rm -f source-ip-test
```

---

## Part III: Testing SWIM Gossip Protocol Requirements

The gossip protocol has different requirements from Raft - it uses UDP instead of TCP and requires unsolicited packet delivery. Let's test these specific requirements.

### Test 3.1: UDP Communication Test

**Theoretical Claim:** UDP communication to remote VM addresses fails because VPNKit cannot handle peer-to-peer UDP patterns.

**Test Procedure:**

We'll create UDP servers and clients to test bidirectional communication patterns:

```bash
# Initialize Swarm to activate gossip protocol
docker swarm init

# Access VM to examine gossip port
docker run -it --rm --privileged --pid=host debian nsenter -t 1 -m -u -n -i sh

# Inside VM, check what's listening on port 7946 (gossip port)
netstat -ulpn | grep 7946
# or:
ss -ulpn | grep 7946
```

**Expected Output:**
```
udp    0    0 0.0.0.0:7946    0.0.0.0:*    12345/dockerd
```

This shows dockerd is listening for UDP on port 7946. Now test UDP connectivity:

```bash
# Test UDP to localhost (should work)
echo "test message" | nc -u -w1 127.0.0.1 7946

# Test UDP to VM's own IP (should work)
echo "test message" | nc -u -w1 192.168.65.3 7946

# Test UDP to hypothetical remote VM (should fail silently)
echo "test message" | nc -u -w1 192.168.65.10 7946
# No response - packet sent into the void
```

**What This Proves:** UDP packets can be sent locally but disappear when destined for addresses outside the local VM. Unlike TCP where you get a connection refused, UDP just silently loses packets.

**Packet Capture Analysis:**

Let's capture UDP packets to see exactly what happens:

```bash
# Start capture on eth0
tcpdump -i eth0 -n 'udp port 7946' -v -X -c 20 -w /tmp/gossip_udp.pcap &
TCPDUMP_PID=$!

# Send some UDP packets to different destinations
echo "local" | nc -u -w1 127.0.0.1 7946
echo "own-ip" | nc -u -w1 192.168.65.3 7946  
echo "fake-remote" | nc -u -w1 192.168.65.10 7946

# Give it time to capture
sleep 2

# Stop capture
kill $TCPDUMP_PID

# Analyze the capture
tcpdump -r /tmp/gossip_udp.pcap -n -v
```

**Expected Observations:**

- Packets to 127.0.0.1 might not appear on eth0 (they use loopback)
- Packets to 192.168.65.3 appear as loopback or local delivery
- Packets to 192.168.65.10 appear going out eth0 but never return

The critical observation is that outbound UDP packets successfully leave eth0, but they're going through AF_VSOCK to VPNKit, which has no handler for the destination address 192.168.65.10. VPNKit silently drops them.

**Compare with Internet UDP:**

```bash
# Send UDP to a real internet server (like Google DNS)
echo "test" | nc -u -w1 8.8.8.8 53
```

This will pass through VPNKit successfully because VPNKit recognizes 8.8.8.8 as an internet address and creates a UDP socket on the host to forward it.

**What This Proves:** VPNKit selectively handles UDP based on destination. Internet destinations work (VPNKit's design purpose). Other VM destinations don't work (outside VPNKit's design scope).

### Test 3.2: Unsolicited Inbound UDP Test

**Theoretical Claim:** Docker Desktop cannot receive unsolicited inbound UDP packets, which breaks gossip's probe mechanism where any node can probe any other node at random times.

**Test Procedure:**

This test requires attempting to send UDP from outside into the VM:

```bash
# From the host, try to send UDP to the VM's port 7946
# (This simulates another Swarm node trying to probe this one)

# On macOS/Linux:
echo "probe" | nc -u -w1 192.168.65.3 7946

# On Windows:
# Use PowerShell UDP client
$udp = New-Object System.Net.Sockets.UdpClient
$bytes = [Text.Encoding]::ASCII.GetBytes("probe")
$udp.Send($bytes, $bytes.Length, "192.168.65.3", 7946)
$udp.Close()
```

**Expected Result:** The packet is sent but never arrives at the VM. There's no error returned because UDP is connectionless - the OS doesn't know if the destination received it or not.

**Verification:**

Inside the VM, run a packet capture while trying the above:

```bash
# Inside VM
tcpdump -i any -n 'udp port 7946' -v

# In another terminal, from host, send the UDP packet
# You should see NO packets in the tcpdump output
```

**What This Proves:** External UDP packets cannot reach the VM. There's no listener on the host that accepts them and forwards through AF_VSOCK. The gossip protocol requires that nodes accept unsolicited probes, which is architecturally impossible in Docker Desktop.

**Real-World Scenario:**

If you had two Docker Desktop instances (Machine A and Machine B), and Swarm on Machine A tried to probe Machine B:
1. Machine A's VM sends UDP to 192.168.65.4 (Machine B's VM IP)
2. Packet reaches Machine A's VPNKit
3. VPNKit has no handler for 192.168.65.4
4. Packet is dropped
5. Machine B never receives the probe
6. Machine A's gossip protocol marks Machine B as suspicious
7. Eventually marks it as dead, even though Machine B is running fine

---

## Part IV: Testing VXLAN Requirements

VXLAN is the most technically complex component, requiring specific packet structures, kernel processing, and FDB learning. Let's test each requirement.

### Test 4.1: VXLAN Interface Creation and Configuration

**Theoretical Claim:** Docker creates VXLAN interfaces correctly inside the VM, but they cannot communicate cross-host due to non-routable VTEP IPs.

**Test Procedure:**

Create an overlay network and examine the VXLAN infrastructure:

```bash
# Create overlay network
docker network create -d overlay --attachable test-overlay

# Access VM
docker run -it --rm --privileged --pid=host debian nsenter -t 1 -m -u -n -i sh

# Inside VM, list all network interfaces including VXLAN
ip -d link show | grep -A 10 vxlan

# Get detailed info on VXLAN interfaces
for iface in $(ip -d link show | grep vxlan | cut -d: -f2 | tr -d ' '); do
    echo "=== Interface: $iface ==="
    ip -d link show $iface
    echo ""
done

# Show VXLAN-specific configuration
ip -d link show | grep -B 2 -A 8 "vxlan id"
```

**Expected Output:**

You should see VXLAN interfaces with output like:
```
10: vxlan1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br-abc123 state UNKNOWN
    link/ether 02:42:ac:14:00:00 brd ff:ff:ff:ff:ff:ff promiscuity 1
    vxlan id 4097 dev eth0 srcport 0 0 dstport 4789 nolearning ageing 300
```

**Key Details to Note:**

- `vxlan id 4097`: The VNI (VXLAN Network Identifier) is 4097
- `dev eth0`: VXLAN is attached to eth0 (which connects to AF_VSOCK)
- `dstport 4789`: Using standard VXLAN port
- `master br-abc123`: Connected to a bridge (which containers attach to)

**What This Proves:** VXLAN infrastructure is created correctly. The kernel VXLAN module is functioning. The configuration matches VXLAN specifications.

### Test 4.2: VXLAN Forwarding Database Examination

**Theoretical Claim:** The FDB (Forwarding Database) contains entries for local containers but no entries for remote VTEPs because there are no remote VTEPs reachable.

**Test Procedure:**

```bash
# Still inside VM
# Show VXLAN forwarding database
bridge fdb show | grep vxlan

# Or more specifically, show FDB for each VXLAN interface
for iface in $(ip -d link show | grep vxlan | cut -d: -f2 | tr -d ' '); do
    echo "=== FDB for $iface ==="
    bridge fdb show dev $iface
    echo ""
done
```

**Expected Output:**

You'll see entries like:
```
00:00:00:00:00:00 dst 192.168.65.3 via eth0 self permanent
02:42:ac:14:00:02 dev veth123 master br-abc123
02:42:ac:14:00:03 dev veth456 master br-abc123
```

**Analysis:**

The first entry `00:00:00:00:00:00 dst 192.168.65.3` is the default entry for unknown destinations - it points to the local VTEP IP (192.168.65.3).

The subsequent entries show MAC addresses of local containers connected to the bridge. There are NO entries showing remote VTEP IPs like `dst 192.168.65.4` because there are no remote VTEPs reachable.

**What This Proves:** The FDB works correctly for local learning but has no remote entries because Docker Desktop's architecture prevents communication with remote VTEPs.

### Test 4.3: VXLAN Packet Encapsulation Test

**Theoretical Claim:** VXLAN encapsulates packets correctly, but they cannot reach remote VTEPs due to routing limitations.

**Test Procedure:**

Create containers on the overlay network and capture VXLAN traffic:

```bash
# Create two containers on the overlay network
docker run -d --name c1 --network test-overlay alpine:latest sleep 3600
docker run -d --name c2 --network test-overlay alpine:latest sleep 3600

# Access VM
docker run -it --rm --privileged --pid=host debian nsenter -t 1 -m -u -n -i sh

# Inside VM, find the VXLAN interface for test-overlay
VXLAN_IFACE=$(ip -d link show | grep -B 2 "test-overlay" | grep vxlan | cut -d: -f2 | tr -d ' ' | head -1)
echo "VXLAN interface: $VXLAN_IFACE"

# Start packet capture on both VXLAN interface and eth0
tcpdump -i $VXLAN_IFACE -w /tmp/vxlan_inner.pcap -c 50 &
tcpdump -i eth0 'udp port 4789' -w /tmp/vxlan_outer.pcap -c 50 &

# Generate traffic between containers
docker exec c1 ping -c 5 c2

# Wait for captures to complete
sleep 5

# Analyze inner VXLAN traffic
tcpdump -r /tmp/vxlan_inner.pcap -n -v

# Analyze outer encapsulated traffic
tcpdump -r /tmp/vxlan_outer.pcap -n -v -X | head -100
```

**Expected Observations:**

The inner capture (vxlan_inner.pcap) shows the original ICMP packets between the containers' IP addresses.

The outer capture (vxlan_outer.pcap) shows the encapsulated packets with structure:
```
Outer IP: 192.168.65.3 → 192.168.65.3 (both local, same VM)
UDP: port X → port 4789
VXLAN Header: VNI = 4097
Inner Ethernet: container MACs
Inner IP: container IPs
Inner ICMP: ping request/reply
```

**What This Proves:** VXLAN encapsulation works perfectly. The kernel correctly wraps packets in VXLAN headers. For local communication (same VM), packets are encapsulated and decapsulated successfully.

**Critical Test - Attempt Cross-Host Pattern:**

Now let's see what happens if we tried to create a scenario with a "remote" VTEP:

```bash
# Inside VM, manually add a fake FDB entry pointing to a non-existent remote VTEP
# (This simulates what would exist if another Swarm node joined)
VXLAN_IFACE=$(ip link show | grep vxlan | head -1 | cut -d: -f2 | tr -d ' ')
bridge fdb append 00:00:00:00:00:00 dev $VXLAN_IFACE dst 192.168.65.10

# Verify entry was added
bridge fdb show dev $VXLAN_IFACE | grep 192.168.65.10

# Try to send traffic that would match this entry
# (We can't actually do this without a real remote container, but we can observe the FDB)
```

**What This Proves:** We can add FDB entries pointing to remote VTEPs, but those VTEPs are unreachable. If real Swarm operations tried to send traffic to such an entry, packets would be encapsulated with destination IP 192.168.65.10, sent through eth0, reach VPNKit, and be dropped because VPNKit has no handler for that address.

---

## Part V: Testing Component Behavior in Depth

Let's examine each architectural component's behavior through direct interaction.

### Test 5.1: AF_VSOCK Detection and Limitations

**Theoretical Claim:** AF_VSOCK provides VM-host communication through shared memory, which is inherently local to one machine.

**Test Procedure:**

```bash
# Access VM
docker run -it --rm --privileged --pid=host debian nsenter -t 1 -m -u -n -i sh

# Check for vsock kernel modules
lsmod | grep vsock

# Check for vsock devices
ls -la /dev/vsock /dev/vhost-vsock 2>/dev/null

# Check PCI devices for virtio
lspci | grep -i virtio

# Examine virtio modules
ls -la /sys/bus/virtio/devices/
```

**Expected Output:**

You should see:
- Kernel modules: `vsock`, `vmw_vsock_virtio_transport`, `vhost_vsock`
- Device: `/dev/vhost-vsock` or `/dev/vsock`
- Virtio devices in `/sys/bus/virtio/devices/`

**What This Proves:** The VM uses vsock for communication, confirming it's isolated from traditional networking.

**Advanced Test - VSOCK Socket Creation:**

If you have programming tools available in the VM, you can create an AF_VSOCK socket:

```bash
# Install Python in VM
apk add python3

# Create test script
cat > /tmp/vsock_test.py << 'EOF'
import socket

# AF_VSOCK is address family 40 on Linux
AF_VSOCK = 40

# Create VSOCK socket
sock = socket.socket(AF_VSOCK, socket.SOCK_STREAM)
print("VSOCK socket created successfully")

# Try to connect to host (CID 2)
try:
    sock.connect((2, 1234))  # CID 2 = host, port 1234
    print("Connected to host (this shouldn't actually work without a listener)")
except Exception as e:
    print(f"Connection failed (expected): {e}")

# Try to connect to a "remote" CID (would be another machine's VM)
try:
    sock.connect((10, 1234))  # CID 10 = hypothetical remote VM
    print("Connected to remote CID (this shouldn't work)")
except Exception as e:
    print(f"Remote CID connection failed (expected): {e}")
EOF

python3 /tmp/vsock_test.py
```

**Expected Result:** The socket can be created, but connections to remote CIDs fail because AF_VSOCK only knows about the local hypervisor's CID space.

**What This Proves:** AF_VSOCK is limited to the local CID space. There's no concept of remote CIDs on other machines.

### Test 5.2: VPNKit Traffic Pattern Analysis

**Theoretical Claim:** VPNKit handles VM-to-internet traffic but drops peer-to-peer traffic.

**Test Procedure:**

Create comprehensive packet captures showing VPNKit's selective behavior:

```bash
# Access VM
docker run -it --rm --privileged --pid=host debian nsenter -t 1 -m -u -n -i sh

# Start comprehensive capture
tcpdump -i eth0 -w /tmp/all_traffic.pcap -s 65535 &
TCPDUMP_PID=$!

# Generate various types of traffic
echo "1. Internet HTTP:"
curl -s -o /dev/null http://example.com

echo "2. Internet HTTPS:"
curl -s -o /dev/null https://www.google.com

echo "3. DNS Query:"
nslookup github.com

echo "4. Ping internet:"
ping -c 2 8.8.8.8

echo "5. Try local network address:"
ping -c 2 -W 1 192.168.1.1 || echo "Failed as expected"

echo "6. Try peer VM address:"
ping -c 2 -W 1 192.168.65.10 || echo "Failed as expected"

echo "7. Try connecting to non-existent local service:"
timeout 2 curl http://192.168.1.100:8080 || echo "Failed as expected"

# Stop capture
sleep 2
kill $TCPDUMP_PID

# Analyze captured traffic
echo "=== Traffic Summary ==="
tcpdump -r /tmp/all_traffic.pcap -n | head -50

echo "=== Internet Traffic (should see responses) ==="
tcpdump -r /tmp/all_traffic.pcap -n 'host 8.8.8.8 or host example.com' | head -20

echo "=== Local/Peer Traffic (should see requests but no responses) ==="
tcpdump -r /tmp/all_traffic.pcap -n 'net 192.168.1.0/24 or host 192.168.65.10' | head -20
```

**Expected Observations:**

- Internet traffic shows both outbound requests AND inbound responses
- Local network traffic shows outbound requests but NO responses
- Peer VM traffic shows outbound requests but NO responses

**Statistical Analysis:**

```bash
# Count packet types
echo "Total packets:"
tcpdump -r /tmp/all_traffic.pcap -n | wc -l

echo "Packets to/from internet IPs:"
tcpdump -r /tmp/all_traffic.pcap -n 'not net 192.168.0.0/16' | wc -l

echo "Packets to local/peer (requests only, no responses):"
tcpdump -r /tmp/all_traffic.pcap -n 'net 192.168.0.0/16 and not host 192.168.65.3' | wc -l
```

**What This Proves:** VPNKit selectively processes traffic. Internet destinations are proxied successfully (bidirectional communication works). Local and peer VM destinations are attempted but fail (unidirectional - requests only, no responses).

### Test 5.3: Performance Comparison - Kernel vs Userspace Path

**Theoretical Claim:** Kernel networking is orders of magnitude faster than userspace proxying like VPNKit.

**Test Procedure:**

We'll measure latency for different communication patterns:

```bash
# Prepare containers for testing
docker run -d --name latency-test --network test-overlay alpine:latest sleep 3600

# Test 1: Loopback (pure kernel)
docker exec latency-test sh -c 'ping -c 100 -i 0.01 127.0.0.1 | grep rtt'

# Test 2: Container-to-container on overlay (kernel with VXLAN)
docker exec c1 sh -c 'ping -c 100 -i 0.01 c2 | grep rtt'

# Test 3: Internet via VPNKit (kernel + userspace + network)
docker exec latency-test sh -c 'ping -c 100 8.8.8.8 | grep rtt'
```

**Expected Results:**

```
Loopback: min/avg/max = 0.020/0.050/0.100 ms
Overlay: min/avg/max = 0.100/0.200/0.500 ms  
Internet: min/avg/max = 10.000/25.000/50.000 ms
```

**Analysis:**

- Loopback is fastest (pure kernel, no network stack)
- Overlay adds VXLAN overhead but still kernel-level (very low latency)
- Internet adds VPNKit userspace processing + actual network latency (much higher)

**What This Proves:** Kernel processing (loopback and overlay) is much faster than userspace processing (VPNKit). The numbers demonstrate why VXLAN must be in kernel for production performance.

### Test 5.4: MTU and Fragmentation Test

**Theoretical Claim:** VXLAN adds 50 bytes overhead, so overlay MTU is reduced to avoid fragmentation.

**Test Procedure:**

```bash
# Check MTU on various interfaces
docker run --rm --privileged alpine:latest sh -c '
echo "=== Loopback MTU ==="
ip link show lo | grep mtu

echo "=== Host bridge MTU ==="
ip link show docker0 | grep mtu

echo "=== Overlay network MTU ==="
ip link show | grep vxlan -A 1 | grep mtu
'

# Test actual packet sizes
docker exec c1 sh -c '
# Try to ping with large packet (near MTU)
ping -c 3 -s 1422 c2  # Should work (1422 + 28 ICMP/IP headers = 1450)
ping -c 3 -s 1472 c2  # Might fragment or fail (1472 + 28 = 1500, too large for overlay)
'
```

**Expected Results:**

- Loopback MTU: 65536 (very large, no physical constraints)
- Docker bridge MTU: 1500 (standard Ethernet)
- VXLAN overlay MTU: 1450 (reduced by 50 bytes for encapsulation overhead)

**What This Proves:** Docker correctly configures overlay MTU to account for VXLAN encapsulation overhead.

---

## Part VI: Integration Testing - The Complete Picture

Now let's run comprehensive tests that demonstrate how all the limitations work together to prevent multi-node Swarm.

### Test 6.1: Simulated Multi-Node Failure Sequence

**Test Objective:** Demonstrate the complete sequence of events when a second node tries to join a Swarm.

**Setup:**

We'll document what WOULD happen if you had two Docker Desktop instances. Since we likely only have one, I'll create a detailed diagnostic sequence showing each failure point.

```bash
# On Machine A - Initialize Swarm
docker swarm init
docker swarm join-token manager

# This outputs something like:
# docker swarm join --token SWMTKN-1-... 192.168.65.3:2377
```

**Theoretical Sequence if Machine B tried to join:**

Let me document this as a step-by-step troubleshooting sequence:

```bash
# Save the diagnostic script
cat > /tmp/multinode_diagnostic.sh << 'EOF'
#!/bin/sh
echo "=== Multi-Node Swarm Diagnostic ==="
echo "This script documents what would fail if attempting multi-node Swarm"
echo ""

MANAGER_IP="192.168.65.3"  # Typical Docker Desktop VM IP
JOIN_PORT="2377"

echo "Step 1: Can we resolve the manager IP?"
if ping -c 1 -W 1 ${MANAGER_IP} > /dev/null 2>&1; then
    echo "✓ PING successful (unexpected - probably local VM)"
else
    echo "✗ PING failed (expected - non-routable IP)"
fi
echo ""

echo "Step 2: Can we reach Swarm port via TCP?"
if timeout 2 nc -zv ${MANAGER_IP} ${JOIN_PORT} 2>&1; then
    echo "✓ TCP connection successful (unexpected unless same machine)"
else
    echo "✗ TCP connection failed (expected - no network path)"
fi
echo ""

echo "Step 3: What does routing say?"
echo "Route to manager:"
ip route get ${MANAGER_IP} 2>/dev/null || echo "No route to host"
echo ""

echo "Step 4: ARP resolution:"
ping -c 1 -W 1 ${MANAGER_IP} > /dev/null 2>&1
echo "ARP table entry for manager:"
ip neigh show ${MANAGER_IP} || echo "No ARP entry (address unreachable)"
echo ""

echo "Step 5: Check our own Swarm advertise address:"
echo "If we were in a Swarm, we'd advertise:"
ip addr show eth0 | grep "inet " | awk '{print $2}' | cut -d/ -f1
echo "This address is ALSO non-routable from other machines"
echo ""

echo "=== Conclusion ==="
echo "Multi-node Swarm cannot work because:"
echo "1. Manager IP (${MANAGER_IP}) is non-routable"
echo "2. No network path exists between VMs on different machines"
echo "3. Even if path existed, our advertise IP is also non-routable"
echo "4. AF_VSOCK provides VM-host communication only, not VM-VM"
EOF

chmod +x /tmp/multinode_diagnostic.sh

# Run inside VM
docker run --rm --privileged alpine:latest sh -c '
apk add --no-cache iproute2 iputils netcat-openbsd
/tmp/multinode_diagnostic.sh
' < /tmp/multinode_diagnostic.sh
```

**What This Test Shows:** A systematic demonstration of each failure point in the multi-node scenario.

---

## Part VII: Comparative Testing

To truly understand the limitations, let's compare Docker Desktop with scenarios that DO work.

### Test 7.1: Comparison with Native Docker Engine

If you have access to a Linux machine or a Linux VM with Docker Engine (not Docker Desktop), run these comparative tests:

**On Linux with native Docker Engine:**

```bash
# Check network interfaces
ip addr show

# Note: You'll see real network interfaces (eth0, wlan0, etc.)
# The IP address will be from your actual network (e.g., 192.168.1.x)

# Initialize Swarm
docker swarm init --advertise-addr <your-actual-ip>

# The advertise address is a REAL, ROUTABLE IP

# Create overlay network and service
docker network create -d overlay test-overlay
docker service create --name web --network test-overlay nginx

# Check VXLAN interfaces
ip -d link show | grep vxlan

# Key difference: VXLAN dev will be on a real network interface (eth0)
# not on a virtualized interface connected to AF_VSOCK
```

**What This Comparison Proves:** On native Linux Docker Engine, Swarm works because:
- IP addresses are real and routable
- Network interfaces connect to actual networks
- No VPNKit or AF_VSOCK intermediary
- Direct kernel-to-kernel communication possible

---

## Part VIII: Conclusion and Verification Checklist

Having completed these tests, you now have empirical evidence for each theoretical claim. Let's create a verification checklist:

### Verification Checklist

**✓ Single-Node Functionality (These WORK):**
- [ ] Docker containers run successfully
- [ ] Internet access from containers works
- [ ] Single-node Swarm initializes
- [ ] Overlay networks create successfully
- [ ] Containers communicate on same-host overlays
- [ ] Port forwarding works for container ports

**✓ Architectural Observations (These are CONFIRMED):**
- [ ] VM has non-routable IP (192.168.65.x)
- [ ] eth0 uses virtio_net driver (hypervisor connection)
- [ ] Default route points to VPNKit (192.168.65.1)
- [ ] AF_VSOCK modules are loaded
- [ ] VXLAN interfaces are created correctly
- [ ] FDB has local entries only

**✓ Multi-Node Failures (These FAIL as predicted):**
- [ ] Cannot ping other VM IPs from host
- [ ] Cannot connect to Swarm port 2377 from host
- [ ] Cannot route to addresses in 192.168.65.0/24 from network
- [ ] UDP packets to peer VMs are dropped
- [ ] No ARP responses for peer VM addresses
- [ ] Port forwarding loses source IP information

**✓ Protocol Requirements Not Met:**
- [ ] Raft cannot identify remote nodes (source IP loss)
- [ ] SWIM cannot probe remote nodes (UDP delivery fails)
- [ ] VXLAN cannot reach remote VTEPs (non-routable IPs)
- [ ] FDB cannot learn remote MACs (no remote traffic)

### Final Summary

Every test you've run provides empirical evidence that transforms theoretical understanding into observable reality. You've seen packets being dropped, connections timing out, and addresses being unreachable. This isn't abstract computer science - it's the concrete behavior of the actual system.

The architectural barriers are not bugs to be fixed but design choices made for specific purposes. Docker Desktop provides excellent single-machine container development. It cannot provide multi-node orchestration because doing so would require abandoning the architectural choices that make it excellent for single-machine use.

**Next Steps:**
- Review the comprehensive protocol study with this empirical evidence in mind
- Understand why each design choice was made
- Appreciate the trade-offs inherent in software architecture
- Use the right tool for each job: Docker Desktop for local development, real infrastructure for distributed systems

---

**END OF PRACTICAL VERIFICATION GUIDE**

*You now have both theoretical understanding and empirical verification of Docker Desktop's architectural limitations. Use this knowledge to make informed infrastructure decisions and to understand the fundamental constraints that shape distributed systems.*
