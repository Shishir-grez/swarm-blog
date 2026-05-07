# Kubernetes (k3s) as Alternative: Complete Guide

## What You'll Learn
- What k3s is and how it differs from Docker Swarm
- Why k3s works better with Tailscale
- k3s architecture and components
- Setting up a k3s cluster over Tailscale
- When to choose k3s vs Docker Swarm

**Prerequisites**: None! We'll explain Kubernetes and K3s from scratch.

---

## Kubernetes and K3s Basics

Before diving into K3s, let's understand Kubernetes. If you're familiar with Kubernetes, skip to [Part 1](#part-1-what-is-k3s).

### What is Kubernetes?

**Kubernetes** (K8s) is a container orchestration platform, like Docker Swarm but more feature-rich and complex.

**Docker Swarm vs Kubernetes:**
```
Docker Swarm:
- Simple, built into Docker
- Easy to learn
- Good for small-medium deployments
- Less features

Kubernetes:
- Complex, separate platform
- Steeper learning curve
- Industry standard
- More features, more flexible
```

**Both do the same core things:**
- Run containers across multiple servers
- Provide high availability
- Load balance traffic
- Auto-heal failures
- Scale applications

### Control Plane vs Data Plane

Kubernetes (and K3s) separate the **control plane** from the **data plane**.

**Control Plane** (the brain):
```
┌─────────────────────────────────┐
│      Control Plane              │
│  - API Server                   │
│  - Scheduler (assigns work)     │
│  - Controller Manager           │
│  - etcd (database)              │
└─────────────────────────────────┘
         ↓ Commands
    (tells workers what to do)
```

**Data Plane** (the workers):
```
┌─────────────────────────────────┐
│      Worker Nodes               │
│  - kubelet (agent)              │
│  - Container runtime (Docker)   │
│  - Runs your actual containers  │
└─────────────────────────────────┘
```

**Analogy:**
```
Control Plane = Management office
- Makes decisions
- Tracks what's happening
- Tells workers what to do

Data Plane = Factory floor
- Does the actual work
- Runs the containers
- Reports status back
```

### What is etcd?

**etcd** is a distributed key-value database that stores Kubernetes cluster state.

**What it stores:**
```
- Which pods are running where
- Network configuration
- Secrets and config maps
- Service definitions
- Everything about your cluster
```

**Why it's distributed:**
```
3 etcd instances:
├─ etcd-1 (has copy of data)
├─ etcd-2 (has copy of data)
└─ etcd-3 (has copy of data)

If one fails, others have the data
Uses Raft consensus (like Docker Swarm)
```

**K3s difference:**
- Standard Kubernetes: Uses etcd (complex, resource-heavy)
- K3s: Uses SQLite by default (simpler, lighter)
- K3s can also use etcd if you want

### What is a Manifest?

A **manifest** is a YAML file that describes what you want Kubernetes to create.

**Example manifest (deployment):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
```

**What this says:**
```
"I want 3 replicas of nginx version 1.21,
listening on port 80,
labeled as 'app: nginx'"
```

**Apply the manifest:**
```bash
kubectl apply -f deployment.yaml

# Kubernetes reads the file and creates what you described
```

**Declarative approach:**
- You describe the desired state (manifest)
- Kubernetes makes it happen
- If something fails, Kubernetes fixes it automatically

### What Makes K3s Different?

**K3s** = Lightweight Kubernetes

**Standard Kubernetes:**
```
Size: ~1GB binary
Memory: ~1GB minimum
Components: Many separate processes
Storage: etcd (complex)
Setup: Complex, many steps
```

**K3s:**
```
Size: ~50MB binary
Memory: ~512MB minimum
Components: Single binary
Storage: SQLite (simple)
Setup: One command
```

**What was removed:**
- Cloud provider integrations (AWS, Azure, GCP)
- Legacy features
- Optional add-ons
- Alpha features

**What was kept:**
- All core Kubernetes APIs
- Full Kubernetes compatibility
- Can run any Kubernetes workload

**Perfect for:**
- Edge computing
- IoT devices
- Development environments
- Small clusters
- Resource-constrained environments

---

## Part 1: What is k3s?

### The Simple Answer
> k3s is a **lightweight Kubernetes distribution** designed for edge, IoT, and resource-constrained environments.

### k3s vs Full Kubernetes
```
Full Kubernetes:
- Binary size: ~1.5 GB
- Memory: ~2-4 GB minimum
- Components: etcd, kube-apiserver, kube-scheduler, etc.

k3s:
- Binary size: ~50 MB
- Memory: ~512 MB minimum
- Components: Single binary with embedded SQLite/etcd
```

### Key Features
- **Single binary**: Everything in one executable
- **Low resource usage**: Runs on Raspberry Pi
- **Production-ready**: CNCF certified Kubernetes
- **Batteries included**: Traefik, CoreDNS, local storage

---

## Part 2: k3s vs Docker Swarm

### Comparison Table

| Feature | Docker Swarm | k3s |
|---------|--------------|-----|
| **Learning Curve** | Easy | Moderate |
| **Resource Usage** | Low | Low-Medium |
| **Ecosystem** | Limited | Huge (Kubernetes) |
| **Networking** | Overlay (VXLAN) | CNI (Flannel default) |
| **VPN Support** | Strict requirements | Flexible |
| **Production Use** | Declining | Growing |
| **Helm Charts** | No | Yes |

### When to Choose k3s
- ✅ You want Kubernetes ecosystem (Helm, operators)
- ✅ You need better VPN integration
- ✅ You plan to scale significantly
- ✅ You want industry-standard skills

### When to Choose Docker Swarm
- ✅ You already know Docker
- ✅ You want simplicity
- ✅ You have a small cluster (< 10 nodes)
- ✅ You don't need Kubernetes features

---

## Part 3: k3s Architecture

### Components
```
┌────────────────────────────────────────┐
│         k3s Server (Master)            │
│                                        │
│  ┌──────────────────────────────────┐  │
│  │  k3s binary                      │  │
│  │  - API Server                    │  │
│  │  - Scheduler                     │  │
│  │  - Controller Manager            │  │
│  │  - SQLite/etcd (datastore)       │  │
│  │  - Kubelet (can run pods)        │  │
│  └──────────────────────────────────┘  │
│                                        │
│  ┌──────────────────────────────────┐  │
│  │  Built-in Components             │  │
│  │  - Traefik (ingress)             │  │
│  │  - CoreDNS (DNS)                 │  │
│  │  - Flannel (CNI)                 │  │
│  │  - Local Path Provisioner        │  │
│  └──────────────────────────────────┘  │
└────────────────────────────────────────┘
              ↓
┌────────────────────────────────────────┐
│         k3s Agent (Worker)             │
│                                        │
│  ┌──────────────────────────────────┐  │
│  │  k3s binary                      │  │
│  │  - Kubelet                       │  │
│  │  - kube-proxy                    │  │
│  │  - containerd                    │  │
│  └──────────────────────────────────┘  │
└────────────────────────────────────────┘
```

### Server vs Agent
- **Server**: Control plane + can run workloads
- **Agent**: Worker node only

---

## Part 4: Why k3s Works Better with Tailscale

### The Key Difference: `--bind-address` Flag

#### Docker Swarm
```bash
docker swarm init --advertise-addr 100.64.0.1
# Tries to bind to the IP
# Fails if IP doesn't exist in namespace
```

#### k3s
```bash
curl -sfL https://get.k3s.io | sh -s - server \
  --node-ip 100.64.0.1 \
  --advertise-address 100.64.0.1 \
  --flannel-iface tailscale0
```

**k3s is more flexible**:
- Can specify which interface to use (`--flannel-iface`)
- Better handling of point-to-point interfaces
- More tolerant of `/32` subnets

### Network Plugin Flexibility
```
Docker Swarm:
  - Uses built-in overlay network (VXLAN)
  - Hard-coded behavior

k3s:
  - Uses CNI (Container Network Interface)
  - Pluggable (Flannel, Calico, Cilium, etc.)
  - Can be customized for VPNs
```

---

## Part 5: Setting Up k3s with Tailscale

### Prerequisites
```bash
# Install Tailscale on all nodes
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

### Server Node (Master)
```bash
# Get Tailscale IP
TAILSCALE_IP=$(tailscale ip -4)

# Install k3s server
curl -sfL https://get.k3s.io | sh -s - server \
  --node-ip $TAILSCALE_IP \
  --advertise-address $TAILSCALE_IP \
  --flannel-iface tailscale0 \
  --tls-san $TAILSCALE_IP

# Get the join token
sudo cat /var/lib/rancher/k3s/server/node-token
```

### Agent Node (Worker)
```bash
# Get Tailscale IP
TAILSCALE_IP=$(tailscale ip -4)

# Get server's Tailscale IP
SERVER_IP=100.64.0.1

# Join the cluster
curl -sfL https://get.k3s.io | K3S_URL=https://$SERVER_IP:6443 \
  K3S_TOKEN=<token-from-server> \
  sh -s - agent \
  --node-ip $TAILSCALE_IP \
  --flannel-iface tailscale0
```

### Verify
```bash
# On server
sudo k3s kubectl get nodes

# Output:
# NAME      STATUS   ROLES                  AGE   VERSION
# server1   Ready    control-plane,master   5m    v1.28.5+k3s1
# worker1   Ready    <none>                 2m    v1.28.5+k3s1
```

---

## Part 6: k3s Networking with Flannel

### How Flannel Works
```
Pod on Node 1 (10.42.0.5)
      ↓
Flannel encapsulates packet
      ↓
Sends via Tailscale interface (100.64.0.1)
      ↓
Tailscale tunnel
      ↓
Arrives at Node 2 (100.64.0.2)
      ↓
Flannel decapsulates
      ↓
Delivers to Pod (10.42.1.3)
```

### Flannel Backends
```bash
# VXLAN (default)
--flannel-backend=vxlan

# Host-GW (faster, requires L2 connectivity)
--flannel-backend=host-gw

# WireGuard (encrypted)
--flannel-backend=wireguard
```

### With Tailscale
```bash
# Use VXLAN over Tailscale
curl -sfL https://get.k3s.io | sh -s - server \
  --flannel-iface tailscale0 \
  --flannel-backend=vxlan
```

---

## Part 7: Deploying Applications

### Create a Deployment
```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

```bash
kubectl apply -f nginx-deployment.yaml
```

### Expose as a Service
```yaml
# nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
```

```bash
kubectl apply -f nginx-service.yaml
```

### Access the Service
```bash
# Get service IP
kubectl get svc nginx

# Access from any node on the tailnet
curl http://<service-ip>
```

---

## Part 8: High Availability

### Multi-Server Setup
```bash
# Server 1 (first server)
curl -sfL https://get.k3s.io | sh -s - server \
  --cluster-init \
  --node-ip $(tailscale ip -4) \
  --flannel-iface tailscale0

# Server 2 (join as server)
curl -sfL https://get.k3s.io | sh -s - server \
  --server https://<server1-tailscale-ip>:6443 \
  --token <token> \
  --node-ip $(tailscale ip -4) \
  --flannel-iface tailscale0

# Server 3 (join as server)
curl -sfL https://get.k3s.io | sh -s - server \
  --server https://<server1-tailscale-ip>:6443 \
  --token <token> \
  --node-ip $(tailscale ip -4) \
  --flannel-iface tailscale0
```

Now you have 3 control plane nodes for HA!

---

## Part 9: Storage

### Local Path Provisioner (Default)
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: local-path
```

### Distributed Storage (Longhorn)
```bash
# Install Longhorn
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/master/deploy/longhorn.yaml

# Use it
storageClassName: longhorn
```

Longhorn replicates data across nodes over Tailscale!

---

## Part 10: Ingress with Traefik

### Default Ingress Controller
k3s includes Traefik by default.

### Create an Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  rules:
  - host: nginx.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx
            port:
              number: 80
```

### Access via Tailscale
```bash
# Add DNS entry in Tailscale or /etc/hosts
echo "100.64.0.1 nginx.example.com" | sudo tee -a /etc/hosts

# Access
curl http://nginx.example.com
```

---

## Part 11: Monitoring

### Install Metrics Server
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### View Resource Usage
```bash
kubectl top nodes
kubectl top pods
```

### Prometheus + Grafana
```bash
# Install kube-prometheus-stack via Helm
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack
```

---

## Part 12: Comparison: Docker Swarm vs k3s Setup

### Docker Swarm
```bash
# Manager
docker swarm init --advertise-addr 100.64.0.1

# Worker
docker swarm join --token <token> 100.64.0.1:2377

# Deploy
docker service create --name web --replicas 3 nginx
```

**Total commands**: 3

### k3s
```bash
# Server
curl -sfL https://get.k3s.io | sh -s - server \
  --node-ip 100.64.0.1 \
  --flannel-iface tailscale0

# Agent
curl -sfL https://get.k3s.io | K3S_URL=https://100.64.0.1:6443 \
  K3S_TOKEN=<token> sh -s - agent \
  --node-ip 100.64.0.2 \
  --flannel-iface tailscale0

# Deploy
kubectl create deployment web --image=nginx --replicas=3
kubectl expose deployment web --port=80
```

**Total commands**: 4

**Verdict**: Similar complexity, but k3s gives you Kubernetes ecosystem.

---

## Part 13: Troubleshooting

### Pods Not Starting
```bash
# Check pod status
kubectl get pods -A

# Describe pod
kubectl describe pod <pod-name>

# Check logs
kubectl logs <pod-name>
```

### Nodes Not Ready
```bash
# Check node status
kubectl get nodes

# Describe node
kubectl describe node <node-name>

# Check k3s logs
sudo journalctl -u k3s -f
```

### Network Issues
```bash
# Check Flannel
kubectl get pods -n kube-system | grep flannel

# Check if Tailscale interface is used
ip route | grep tailscale0
```

---

## Key Takeaways

1. **k3s = Lightweight Kubernetes** (~50 MB binary)
2. **Better VPN support** than Docker Swarm (flexible interface binding)
3. **`--flannel-iface tailscale0`** makes it work with Tailscale
4. **Kubernetes ecosystem** (Helm, operators, huge community)
5. **Similar complexity** to Docker Swarm for basic use cases
6. **Better for long-term** if you want to learn industry-standard tools

---

## Further Reading
- [k3s Documentation](https://docs.k3s.io/)
- [k3s + Tailscale Guide](https://tailscale.com/kb/1185/kubernetes/)
- [Kubernetes Basics](https://kubernetes.io/docs/tutorials/kubernetes-basics/)
