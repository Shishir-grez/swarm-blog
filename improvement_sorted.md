# Swarm Blog Improvements: Chronological Roadmap
*Sorted by appearance in `blog.md` (Ascending LOC)*

---

## 📄 Primary: `blog.md`

### 🏗️ Pre-Architecture / Foundations (Target: L48)
- **[Added] VM Fundamentals**: What is a VM and its core components (vCPU, vRAM, vDisk).
  - **SOURCE**: `missed_topics.md:L7-36` | `guides/00_comprehensive_guide.md:L20-55`
- **[Added] Hypervisor Deep Dive**: Definition, responsibilities (resource allocation, isolation), and types (Bare Metal vs Hosted).
  - **SOURCE**: `missed_topics.md:L38-47` | `guides/00_comprehensive_guide.md:L58-94`
- **[Added] VM Networking Modes**: Bridged, NAT, and Host-Only modes.
  - **SOURCE**: `missed_topics.md:L48-55` | `claims/complete_conversation_takeaways.md:L196-218`
- **[Added] Container vs VM**: Architectural differences and encapsulation hierarchy.
  - **SOURCE**: `missed_topics.md:L84-107` | `guides/00_comprehensive_guide.md:L132-160`
- **[Added] Encapsulation & Tunneling**: How packets are wrapped for transport across networks.
  - **SOURCE**: `missed_topics.md:L70-82` | `guides/00_comprehensive_guide.md:L356-384`, `L546-613`
- **[Added] Networking Vocabulary**: Definitions for Gateway, Network Interface, and Network Bridge.
  - **SOURCE**: `missed_topics.md:L58-76` | `guides/00_comprehensive_guide.md:L444-460`
- **[Added] Userspace vs Kernelspace Networking**: Context switches and performance trade-offs.
  - **SOURCE**: `missed_topics.md:L111-127` | `guides/temp_qa_clarifications.md:L463-800`
- **[Added] AF-VSOCK Deep Dive**: Hypervisor sockets and shared memory limitations.
  - **SOURCE**: `missed_topics.md:L143-147` | `claims/docker_desktop_networking_deep_dive.md:L74-112`

### 📍 Inline Improvements (Sorted by Line Number)
- **[Removed] Redundant Phrasing**: Remove "from" in formatting contexts. (LOC: L16)
  - *Note: Found near analogy description.*
- **[Improved] Raft Simplification**: "How it works" section needs better clarity. (LOC: L673)
  - **SOURCE**: `guides/temp_qa_clarifications.md:L112-182`, `L1081-1336`
- **[Added] Internal IP Mention**: Clarify that discussed IPs are for internal use only. (LOC: L695)
- **[Tech] Raft Heartbeats**: Clarify that they are unsolicited from the follower's perspective. (LOC: L747)
- **[Tech] Gossip Protocol**: Note that it is symmetric but unpredictable in direction. (LOC: L749)
- **[Tech] Routing Mesh (Ingress)**: Emphasize that ingress is the only channel for external traffic. (LOC: L753)
- **[Flow] Takeaways/Conclusions**: Add to end of block to justify technical depth. (LOC: L755)
  - **SOURCE**: `claims/complete_conversation_takeaways.md` (Topic 7)
- **[Added] Linux Ecosystem**: Define "Linux Daemon" and its role. (LOC: L765)
  - **SOURCE**: `missed_topics.md:L526-542`
- **[Tech] Isolation (Core Principle)**: Explain shared kernel vs. namespace/cgroup isolation. (LOC: L773)
- **[Side Quest] Docker Image Layers**: Explain how image layers work near Mount Namespace. (LOC: L837)
  - **SOURCE**: `guides/00_comprehensive_guide.md:L162-190`
- **[Added] IPC & Sockets**: Explain pipes, named pipes, and Unix sockets (`/var/run/docker.sock`). (LOC: L888)
  - **SOURCE**: `missed_topics.md:L129-136` | `guides/temp_qa_clarifications.md:L1578-1600+`
- **[Flow] Takeaways/Conclusions**: Add to end of block to justify technical depth. (LOC: L1025)
- **[Added] Container Networking 101**: Explain `veth`, `bridge`, and general container architecture. (LOC: L1027)
  - **SOURCE**: `missed_topics.md:L67-69` | `claims/complete_conversation_takeaways.md:L324-330`
- **[Removed] Redundant Phrasing**: Remove "thus" from VXLAN header. (LOC: L1027)
- **[Improved] Routing Table Formatting**: Refactor analogy table. (LOC: L1033)
- **[Flow] Summarize Swarm Creation**: Refactor the list into a narrative summary. (LOC: L1067)
- **[Removed] Sections to Cut**: "The ingress overlay gets most of the attention...". (LOC: L1097)
- **[Added] Transparent Proxy**: Add fun fact about inspection and data visibility. (LOC: L1116)
- **[Removed] Information Density**: Hide IP address allocation details. (LOC: L1140)
- **[Removed] Information Density**: Hide Gossip implementation details. (LOC: L1189)
- **[Improved] Terminology**: Standardize T=3, T=4 terminology. (LOC: L1251)
- **[Flow] Takeaways/Conclusions**: Add to end of block to justify technical depth. (LOC: L1260)
- **[Flow] Transitions**: Insert explicit "Now we know..." transition lines. (LOC: L1261)
- **[Side Quest] VNI Selection**: Explain the VNI assignment pattern. (LOC: L1284)
- **[Diagram] VXLAN Packet Structure**: Show Before/After & MAC-in-UDP. (LOC: L1328)
  - **SOURCE**: `guides/00_comprehensive_guide.md:L655-674`
- **[Flow] Transitions**: Insert explicit "Now we know..." transition lines. (LOC: L1378)
- **[Improved] VTEP Interface**: Clarify specific use cases. (LOC: L1391)
  - **SOURCE**: `guides/00_comprehensive_guide.md:L708-754`
- **[Improved] VM Networking (Failure)**: Explain why traditional modes fail for Swarm. (LOC: L2885)
  - **SOURCE**: `missed_topics.md:L156-170` | `claims/docker_desktop_networking_deep_dive.md:L343-395`
- **[Diagram] WSL2 Architecture**: High-level Windows solution diagram. (LOC: L2885)
  - **SOURCE**: `missed_topics.md:L218-236`
- **[Side Quest] Shared Memory vs TCP/IP**: Rationale for choosing shared memory in VPNKit. (LOC: L3200)
  - **SOURCE**: `claims/docker_desktop_networking_deep_dive.md:L51-112`
- **[Improved] Terminology**: Standardize T=3, T=4 terminology (Part 2). (LOC: L3208)
- **[Side Quest] Tailscale Sidecar**: Explain why it won't work in this specific context. (LOC: L3258)
- **[Diagram] VPNKit Architecture**: Full architecture Mermaid diagram. (LOC: L3258)
  - **SOURCE**: `missed_topics.md:L174-216`
- **[Improved] Port Forwarding**: Bind and bidirectional limits refactor. (LOC: L3269)
  - **SOURCE**: `claims/docker_desktop_networking_deep_dive.md:L5-50`
- **[Tech] VPNKit Translation**: Explain translation for VM identifiers and IP addresses. (LOC: L3304)
  - **SOURCE**: `claims/docker_desktop_networking_deep_dive.md:L180-335`

---

## 📄 Secondary: `blog_1.md`

- **[Removed] Sections to Cut**: "Port numbers are essential" in Tailscale section. (LOC: L139)
- **[Improved] NAT & Connection State**: (LOC: L241)
  - **SOURCE**: `claims/complete_conversation_takeaways.md:L129-147` | `guides/00_comprehensive_guide.md:L386-413`
- **[Improved] Port Forwarding**: (LOC: L277)
- **[Diagram] Tailscale Mechanics**: STUN, Hole Punching, and DERP diagrams. (LOC: L390, L427, L482)
  - **SOURCE**: `guides/temp_qa_clarifications.md:L112-182`
- **[Improved] Hole Punching**: Simplify "How it works". (LOC: L419)
- **[Improved] TUN Interface**: Simplify technical explanation. (LOC: L534)
  - **SOURCE**: `guides/temp_qa_clarifications.md:L614-744`

---

## 📂 Verification & Resources
- **[Verification] Technical Claims**: Reference `claims/` folder for exhaustive blockers.
- **[Verification] Claims Checker**: `consolidated_technical_claims.md` and `exhaustive_technical_claims_deep_dive.md`.
- **[Diagrams] General Repo**: See `claims/docker_architecture_diagrams.md` and `claims/docker-swarm-architecture-diagrams.md`.
