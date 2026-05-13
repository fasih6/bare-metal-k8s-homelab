# MetalLB — Comprehensive Notes
## Bare Metal Load Balancer for Kubernetes

> Part of `fasih6/My_Devops_Notes` — Bare Metal Kubernetes Series

---

## Table of Contents

1. [What is MetalLB?](#1-what-is-metallb)
2. [The Problem it Solves](#2-the-problem-it-solves)
3. [How MetalLB Works](#3-how-metallb-works)
4. [Operating Modes](#4-operating-modes)
5. [Architecture](#5-architecture)
6. [Installation](#6-installation)
7. [Configuration](#7-configuration)
8. [Using MetalLB with Services](#8-using-metallb-with-services)
9. [MetalLB with NGINX Ingress](#9-metallb-with-nginx-ingress)
10. [Advanced Configuration](#10-advanced-configuration)
11. [Troubleshooting](#11-troubleshooting)
12. [Interview Q&A](#12-interview-qa)

---

## 1. What is MetalLB?

MetalLB is an **open-source load balancer implementation for bare metal Kubernetes clusters**. It gives Kubernetes a way to create `LoadBalancer` type Services without relying on a cloud provider.

- **Website:** [metallb.universe.tf](https://metallb.universe.tf)
- **License:** Apache 2.0
- **CNCF Status:** Sandbox project
- **Language:** Go

### In One Sentence

MetalLB watches for Services of type `LoadBalancer` and assigns them real, routable IP addresses from a pool you define — making them accessible from outside the cluster.

---

## 2. The Problem it Solves

### What Happens Without MetalLB

On managed Kubernetes (EKS, AKS, GKE), when you create a Service of type `LoadBalancer`:

```
Cloud Provider detects the Service
→ Provisions a cloud load balancer (AWS ALB, Azure LB, GCP LB)
→ Assigns an external IP
→ Traffic flows: Internet → Cloud LB → Node → Pod
```

On bare metal Kubernetes, there is no cloud provider to do this. The result:

```bash
kubectl get svc my-service
# NAME         TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)
# my-service   LoadBalancer   10.96.45.123   <pending>     80:30080/TCP
```

The `EXTERNAL-IP` stays `<pending>` **forever**. No IP is ever assigned. The Service is unreachable from outside the cluster.

### Your Only Alternative Without MetalLB

Without MetalLB, you are limited to:

| Option | Limitation |
|--------|-----------|
| `NodePort` | Exposes a random high port (30000-32767) on every node — ugly, not production-grade |
| `hostNetwork: true` | Pod binds directly to node IP — no isolation, port conflicts |
| Manual iptables | Complex, fragile, not managed by Kubernetes |

MetalLB makes `LoadBalancer` Services work exactly as they do on the cloud — clean external IPs, standard ports.

---

## 3. How MetalLB Works

At a high level, MetalLB does two things:

1. **Address Allocation** — assigns an IP from your configured pool to a `LoadBalancer` Service
2. **Address Advertisement** — tells the network that this IP lives at your cluster nodes so traffic can reach them

The advertisement mechanism depends on which mode you use — L2 or BGP.

### The Full Flow (L2 Mode)

```
1. You create a Service of type LoadBalancer

2. MetalLB controller detects the new Service
   → picks an available IP from the IPAddressPool
   → assigns it to the Service as externalIP

3. MetalLB speaker on one node sends ARP responses
   → "Who has 192.168.1.200? I do — MAC: aa:bb:cc:dd:ee:ff"
   → neighbour devices on the LAN now know: send 192.168.1.200 to that node

4. External traffic arrives at the node
   → kube-proxy forwards it to the correct pod via iptables/ipvs
   → pod receives the request
```

```bash
kubectl get svc my-service
# NAME         TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)
# my-service   LoadBalancer   10.96.45.123   192.168.1.200   80:30080/TCP
```

The `EXTERNAL-IP` is now assigned and routable.

---

## 4. Operating Modes

MetalLB supports two fundamentally different advertisement modes.

### Mode 1: L2 Mode (Layer 2 / ARP)

**How it works:**

MetalLB uses **ARP** (Address Resolution Protocol) for IPv4 or **NDP** (Neighbor Discovery Protocol) for IPv6. One node in the cluster — the **leader** — responds to ARP queries for the assigned IP, claiming it owns that IP on the local network.

```
External client: "Who has 10.0.0.100?"
MetalLB speaker (leader node): "I do — here's my MAC address"
Client sends traffic to that MAC → reaches the node → kube-proxy forwards to pod
```

**Characteristics:**
- Simple to configure — works on any network, no router support needed
- Single node handles all traffic for a given IP (the ARP leader)
- No true load balancing at the network level — all traffic enters via one node
- Kubernetes then distributes to pods via kube-proxy
- Failover: if the leader node goes down, another speaker takes over (takes 10–30 seconds)

**When to use:**
- Homelabs, small clusters, simple setups
- When you do not control your router or network switches
- Your Hetzner project → use L2 mode

**Limitation:**
- Single node ingress point — that node is a bottleneck
- Not suitable for very high traffic clusters

### Mode 2: BGP Mode (Border Gateway Protocol)

**How it works:**

MetalLB establishes **BGP peering sessions** with your network routers. It advertises routes for the assigned IP addresses. Routers distribute traffic across all nodes using ECMP (Equal-Cost Multi-Path routing).

```
MetalLB speaker on Node 1 → BGP peer → Router: "10.0.0.100 reachable via Node 1"
MetalLB speaker on Node 2 → BGP peer → Router: "10.0.0.100 reachable via Node 2"
MetalLB speaker on Node 3 → BGP peer → Router: "10.0.0.100 reachable via Node 3"

Router: ECMP — distribute traffic across Node 1, 2, and 3
```

**Characteristics:**
- True load balancing at the network layer across all nodes
- Instant failover — BGP withdraws route when a node goes down
- Requires router support for BGP (enterprise routers, cloud VMs with BGP, etc.)
- More complex to configure

**When to use:**
- Production environments with BGP-capable routers
- High traffic clusters needing true network-level load balancing
- Data centres with proper networking infrastructure

**Hetzner note:** Hetzner's basic cloud networking does not support BGP peering with customer VMs. Use L2 mode for your homelab project.

---

## 5. Architecture

### Components

```
┌─────────────────────────────────────────────────────────┐
│                   Kubernetes Cluster                     │
│                                                          │
│  ┌─────────────────────────────────────┐                 │
│  │         metallb-system namespace    │                 │
│  │                                     │                 │
│  │  ┌──────────────────────────────┐   │                 │
│  │  │    Controller (Deployment)   │   │                 │
│  │  │                              │   │                 │
│  │  │  - Watches for LB Services   │   │                 │
│  │  │  - Assigns IPs from pool     │   │                 │
│  │  │  - Updates Service status    │   │                 │
│  │  └──────────────────────────────┘   │                 │
│  │                                     │                 │
│  │  ┌──────────────────────────────┐   │                 │
│  │  │    Speaker (DaemonSet)       │   │                 │
│  │  │    one pod per node          │   │                 │
│  │  │                              │   │                 │
│  │  │  - Advertises IPs via ARP    │   │                 │
│  │  │    (L2) or BGP               │   │                 │
│  │  │  - Leader election per IP    │   │                 │
│  │  │  - Handles failover          │   │                 │
│  │  └──────────────────────────────┘   │                 │
│  └─────────────────────────────────────┘                 │
└─────────────────────────────────────────────────────────┘
```

### Controller

- Runs as a single **Deployment** (one replica)
- Watches the Kubernetes API for Services of type `LoadBalancer`
- When a new LoadBalancer Service appears: allocates an IP from the configured pool
- Updates the Service's `status.loadBalancer.ingress` field with the assigned IP
- Handles IP deallocation when Services are deleted

### Speaker

- Runs as a **DaemonSet** — one pod per node
- Responsible for advertising the assigned IP to the network
- In L2 mode: runs ARP/NDP responder, leader election determines which speaker responds
- In BGP mode: all speakers establish BGP sessions with routers and advertise routes
- Uses **memberlist** for cluster membership and leader election between speakers

### Leader Election (L2 Mode)

For each assigned IP, MetalLB elects one speaker as the leader. Only the leader responds to ARP queries for that IP. If the leader node goes down:

```
Leader node fails
→ Remaining speakers detect via memberlist (gossip protocol)
→ New leader election occurs (seconds)
→ New leader starts responding to ARP for that IP
→ External clients re-learn the MAC → failover complete
```

Failover typically takes **10–30 seconds** in L2 mode, limited by ARP cache expiry on network devices.

---

## 6. Installation

### Prerequisites

- Kubernetes cluster (k3s, kubeadm, etc.)
- kube-proxy in `iptables` or `ipvs` mode (NOT `nftables`)
- If using kube-proxy in IPVS mode: `strictARP` must be enabled
- No other network components claiming LoadBalancer IPs (e.g. disable k3s's built-in servicelb)

### Important: Disable k3s Built-in Load Balancer

k3s ships with its own load balancer called **ServiceLB (klipper-lb)**. You must disable it before installing MetalLB, otherwise they conflict.

```bash
# If installing k3s fresh:
curl -sfL https://get.k3s.io | sh -s - --disable servicelb

# If k3s is already running, edit the config:
# Add to /etc/rancher/k3s/config.yaml:
echo "disable: servicelb" >> /etc/rancher/k3s/config.yaml
systemctl restart k3s
```

### Install via Helm (Recommended)

```bash
# Add MetalLB Helm repo
helm repo add metallb https://metallb.github.io/metallb
helm repo update

# Install MetalLB
helm install metallb metallb/metallb \
  --namespace metallb-system \
  --create-namespace

# Wait for pods to be ready
kubectl wait --namespace metallb-system \
  --for=condition=ready pod \
  --selector=app=metallb \
  --timeout=90s

# Verify pods
kubectl get pods -n metallb-system
# NAME                                  READY   STATUS
# metallb-controller-xxx                1/1     Running
# metallb-speaker-xxx (per node)        1/1     Running
```

### Install via kubectl (Alternative)

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.3/config/manifests/metallb-native.yaml
```

---

## 7. Configuration

MetalLB v0.13+ uses **CRDs (Custom Resource Definitions)** for configuration. No more ConfigMaps.

### Step 1: Create an IP Address Pool

The IP pool defines which addresses MetalLB can assign to Services.

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.200-192.168.1.250   # Range of IPs
```

Other address formats:

```yaml
spec:
  addresses:
  - 192.168.1.200-192.168.1.210   # Range
  - 192.168.1.100/32              # Single IP (CIDR notation)
  - 10.0.0.0/24                   # Entire subnet
```

**For Hetzner single-server homelab** — use your server's public IP:

```yaml
spec:
  addresses:
  - YOUR_SERVER_PUBLIC_IP/32      # Just the one IP you have
```

### Step 2: Create an Advertisement

The advertisement tells MetalLB how to announce the IPs.

**L2 Advertisement** (for homelab/simple setups):

```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - default-pool              # References the pool by name
```

**L2 Advertisement without pool reference** (advertises ALL pools):

```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default-l2
  namespace: metallb-system
```

**BGP Advertisement** (for production with BGP routers):

```yaml
apiVersion: metallb.io/v1beta1
kind: BGPAdvertisement
metadata:
  name: default-bgp
  namespace: metallb-system
spec:
  ipAddressPools:
  - default-pool
  communities:
  - 65000:1                   # BGP community tag
```

### Step 3: Apply Configuration

```bash
kubectl apply -f metallb-config.yaml

# Verify pool was created
kubectl get ipaddresspool -n metallb-system
# NAME           AUTO ASSIGN   AVOID BUGGY IPS   ADDRESSES
# default-pool   true          false             ["192.168.1.200-192.168.1.250"]

# Verify advertisement was created
kubectl get l2advertisement -n metallb-system
# NAME        IPADDRESSPOOLS   IPADDRESSPOOL SELECTORS   INTERFACES
# default-l2  ["default-pool"]
```

### Multiple Pools

You can create multiple pools for different purposes:

```yaml
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: production-pool
  namespace: metallb-system
spec:
  addresses:
  - 10.0.0.100-10.0.0.150
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: staging-pool
  namespace: metallb-system
spec:
  addresses:
  - 10.0.0.200-10.0.0.220
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: production-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - production-pool
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: staging-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - staging-pool
```

---

## 8. Using MetalLB with Services

### Basic LoadBalancer Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: default
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
```

```bash
kubectl apply -f service.yaml

kubectl get svc my-app
# NAME     TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)
# my-app   LoadBalancer   10.96.1.100   192.168.1.200   80:31234/TCP
```

MetalLB automatically assigns `192.168.1.200` from the pool.

### Request a Specific IP

```yaml
spec:
  type: LoadBalancer
  loadBalancerIP: 192.168.1.205   # Must be within a configured pool
```

### Select a Specific Pool

Use an annotation to direct MetalLB to use a particular pool:

```yaml
metadata:
  annotations:
    metallb.universe.io/address-pool: staging-pool
spec:
  type: LoadBalancer
```

### Sharing an IP Between Services (Different Ports)

By default, each Service gets its own IP. To share one IP across multiple Services:

```yaml
# Service 1 — HTTP
metadata:
  annotations:
    metallb.universe.io/allow-shared-ip: "shared-key"
spec:
  type: LoadBalancer
  loadBalancerIP: 192.168.1.200
  ports:
  - port: 80
    protocol: TCP
---
# Service 2 — HTTPS
metadata:
  annotations:
    metallb.universe.io/allow-shared-ip: "shared-key"
spec:
  type: LoadBalancer
  loadBalancerIP: 192.168.1.200
  ports:
  - port: 443
    protocol: TCP
```

Both Services share `192.168.1.200` — useful when you have limited IPs.

### ExternalTrafficPolicy

Controls how traffic is routed after reaching the node:

```yaml
spec:
  type: LoadBalancer
  externalTrafficPolicy: Cluster   # default — any node can receive, kube-proxy forwards
  # OR
  externalTrafficPolicy: Local     # only nodes running the pod receive traffic
```

| Policy | Pros | Cons |
|--------|------|------|
| `Cluster` (default) | Traffic distributed across all nodes | Source IP is lost (SNAT applied) |
| `Local` | Source IP preserved | Traffic only reaches nodes with matching pods |

> Use `Local` when you need the real client IP (e.g. for rate limiting, geo-blocking, logging).

---

## 9. MetalLB with NGINX Ingress

In a typical bare metal setup, MetalLB and NGINX Ingress work together:

```
Internet
    ↓
MetalLB IP (e.g. 10.0.0.100)
    ↓
NGINX Ingress Controller Service (LoadBalancer type)
    ↓
NGINX Ingress Controller Pod
    ↓  (routes by hostname/path)
    ├── Service A → Pod A
    ├── Service B → Pod B
    └── Service C → Pod C
```

MetalLB assigns the external IP to the NGINX Ingress Service. NGINX then handles all HTTP/HTTPS routing via Ingress rules. You only need **one MetalLB IP** for any number of services — NGINX differentiates by hostname.

### Install NGINX Ingress (Gets IP from MetalLB)

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/cloud/deploy.yaml

# Check — MetalLB should assign an IP
kubectl get svc -n ingress-nginx
# NAME                       TYPE           EXTERNAL-IP
# ingress-nginx-controller   LoadBalancer   192.168.1.200   ← assigned by MetalLB
```

### Example Ingress Rule

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app
            port:
              number: 80
```

Traffic flow:
```
curl http://app.example.com
  → DNS resolves to 192.168.1.200 (MetalLB IP)
  → reaches NGINX Ingress pod
  → NGINX matches host: app.example.com
  → forwards to my-app Service
  → reaches pod
```

---

## 10. Advanced Configuration

### BGP Configuration (Reference)

For environments with BGP-capable routers:

```yaml
# Define BGP peer (your router)
apiVersion: metallb.io/v1beta2
kind: BGPPeer
metadata:
  name: router
  namespace: metallb-system
spec:
  myASN: 64500          # Your cluster's AS number
  peerASN: 64501        # Your router's AS number
  peerAddress: 10.0.0.1 # Router IP
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: bgp-pool
  namespace: metallb-system
spec:
  addresses:
  - 203.0.113.0/24
---
apiVersion: metallb.io/v1beta1
kind: BGPAdvertisement
metadata:
  name: bgp-advert
  namespace: metallb-system
spec:
  ipAddressPools:
  - bgp-pool
  peers:
  - router
```

### Interface Selector (L2 Mode)

Restrict which network interface MetalLB uses for L2 advertisement:

```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-eth0
  namespace: metallb-system
spec:
  ipAddressPools:
  - default-pool
  interfaces:
  - eth0        # Only advertise on this interface
```

Useful when nodes have multiple NICs and you want to control which network carries the traffic.

### Node Selector (L2 Mode)

Restrict which nodes can be elected as the ARP leader:

```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-selected-nodes
  namespace: metallb-system
spec:
  ipAddressPools:
  - default-pool
  nodeSelectors:
  - matchLabels:
      kubernetes.io/hostname: k3s-master   # Only master can be leader
```

### Avoid Buggy IPs

Some network switches have issues with certain IPs (e.g. the first/last IP in a range). Tell MetalLB to skip them:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.200-192.168.1.250
  avoidBuggyIPs: true    # Skips .0 and .255 addresses
```

---

## 11. Troubleshooting

### Service Still Shows `<pending>`

```bash
# Check if MetalLB controller is running
kubectl get pods -n metallb-system

# Check controller logs
kubectl logs -n metallb-system -l app=metallb,component=controller

# Common causes:
# 1. No IPAddressPool configured
# 2. Pool has no available IPs (all assigned)
# 3. L2Advertisement not created
# 4. k3s servicelb still running and conflicting
```

### Check IP Pool Availability

```bash
kubectl get ipaddresspool -n metallb-system -o yaml
# Look at: spec.addresses vs how many services are using IPs

# See which IPs are allocated
kubectl get services --all-namespaces | grep LoadBalancer
```

### Check Speaker Logs

```bash
# Speaker handles the actual ARP/BGP advertisement
kubectl logs -n metallb-system -l app=metallb,component=speaker --tail=50

# Common messages to look for:
# "triggering new leader election" — failover happening
# "not sending ARP" — node not elected as leader
# "sending ARP" — this node is the leader for that IP
```

### ARP Not Working (L2 Mode)

```bash
# On your local machine or router, check ARP table
arp -n
# The MetalLB IP should map to one of your node MACs

# If not there, the ARP announcement is not reaching your network
# Check: firewall rules, network interface, VLAN configuration
```

### Conflict with k3s ServiceLB

```bash
# Check if klipper-lb is running (k3s built-in)
kubectl get pods --all-namespaces | grep klipper

# If running, disable it
# Edit /etc/rancher/k3s/config.yaml and add: disable: servicelb
# Then restart: systemctl restart k3s
```

### IP Already in Use on Network

```bash
# If the IP you configured is already used by another device:
# MetalLB will assign it but traffic will be unpredictable (ARP conflict)

# Check before configuring:
ping 192.168.1.200   # Should be unreachable if IP is free
arping 192.168.1.200 # Check for ARP responses
```

### Verify MetalLB CRDs are Installed

```bash
kubectl get crd | grep metallb
# Should show:
# bfdprofiles.metallb.io
# bgpadvertisements.metallb.io
# bgppeers.metallb.io
# communities.metallb.io
# ipaddresspools.metallb.io
# l2advertisements.metallb.io
```

---

## 12. Interview Q&A

**Q: What is MetalLB and why is it needed on bare metal Kubernetes?**

On cloud Kubernetes, Services of type LoadBalancer automatically get an external IP from the cloud provider's load balancer service. On bare metal, no such provider exists — the Service stays in pending state with no IP ever assigned. MetalLB fills this gap by acting as a network load balancer implementation. It assigns IPs from a user-defined pool and advertises them to the network using ARP (L2 mode) or BGP, making the Services reachable from outside the cluster.

---

**Q: What is the difference between L2 mode and BGP mode in MetalLB?**

L2 mode uses ARP to advertise IPs. One node — the elected leader — responds to ARP queries claiming it owns the IP. All external traffic enters the cluster through that single node. It is simple and works on any network without special router support. BGP mode establishes BGP sessions with routers and advertises routes for the IPs. Routers then distribute traffic across all nodes using ECMP, providing true network-level load balancing. BGP requires BGP-capable routers but gives better traffic distribution and faster failover.

---

**Q: What is a limitation of L2 mode?**

In L2 mode, only one node handles all incoming traffic for a given IP — the ARP leader. This node becomes a single ingress point and can be a bandwidth bottleneck for high-traffic services. It is not true load balancing at the network level. Kubernetes still distributes traffic across pods via kube-proxy, but all traffic must first pass through the one leader node. Failover when the leader goes down takes 10–30 seconds, limited by ARP cache expiry on network devices.

---

**Q: How does MetalLB integrate with NGINX Ingress Controller?**

MetalLB assigns an external IP to the NGINX Ingress Controller's Service of type LoadBalancer. NGINX then handles all HTTP/HTTPS routing via Ingress rules — matching requests by hostname and path and forwarding them to the correct backend Services. This means you need only one MetalLB IP for potentially dozens of services. MetalLB handles the external IP assignment; NGINX handles the application-level routing.

---

**Q: What are IPAddressPool and L2Advertisement in MetalLB's CRD-based config?**

IPAddressPool defines a range of IP addresses that MetalLB is allowed to assign to LoadBalancer Services. L2Advertisement tells MetalLB to advertise those IPs using Layer 2 ARP/NDP and optionally specifies which pools, nodes, or interfaces to use. Both are Kubernetes custom resources. Before MetalLB v0.13, this was configured via a ConfigMap — the CRD approach is more flexible and Kubernetes-native.

---

**Q: What is ExternalTrafficPolicy and when would you use Local vs Cluster?**

ExternalTrafficPolicy controls how a node handles traffic arriving for a LoadBalancer Service. `Cluster` (default) allows any node to receive the traffic and uses kube-proxy to forward it to any pod in the cluster, but this applies SNAT so the original client IP is lost. `Local` only forwards traffic on nodes that are actually running a matching pod, and preserves the original client IP since no SNAT is needed. You would use `Local` when you need the real client IP for logging, rate limiting, or geo-blocking.

---

**Q: How does MetalLB handle failover in L2 mode?**

MetalLB speakers use a gossip protocol called memberlist to monitor each other. When the leader node for an IP goes down, the remaining speakers detect this via memberlist and run a new leader election. The new leader begins responding to ARP queries for that IP. External clients update their ARP caches and resume sending traffic to the new leader node. The full failover process typically takes 10–30 seconds, the main delay being ARP cache expiry on routers and switches.

---

**Q: Why did you disable k3s's built-in ServiceLB before installing MetalLB?**

k3s ships with a built-in load balancer called ServiceLB (klipper-lb) that also handles LoadBalancer Services. Running both MetalLB and ServiceLB simultaneously causes conflicts — both try to assign IPs and handle the same Services, leading to unpredictable behaviour. MetalLB must be the sole controller responsible for LoadBalancer Services, so ServiceLB must be disabled first using the `--disable servicelb` flag when installing k3s.

---

**Q: Can multiple Services share the same MetalLB IP?**

Yes, using the `metallb.universe.io/allow-shared-ip` annotation with the same key on multiple Services, combined with specifying the same `loadBalancerIP`. The Services must use different ports to avoid conflicts. This is useful when you have a limited IP pool — for example, sharing one IP between an HTTP service on port 80 and an HTTPS service on port 443.

---

*Notes by Fasih-Ur-Rehman Shahid | github.com/fasih6/My_Devops_Notes*
