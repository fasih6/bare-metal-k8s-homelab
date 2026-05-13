# bare-metal-k8s-homelab

A two-node bare metal Kubernetes cluster built from scratch using k3s, Longhorn distributed storage, MetalLB, and a self-authored Helm chart. Deployed a stateful PostgreSQL instance and verified data persistence across a simulated node failure.

Built to close the gap between managed Kubernetes (EKS) and real bare metal operations — no cloud control plane, no cloud storage backend, no cloud load balancer.

---

## Architecture

```
                        ┌──────────────────────────────────────────┐
                        │              AWS VPC (172.31.0.0/16)     │
                        │                                          │
                        │  ┌──────────────────┐  ┌───────────────┐ │
                        │  │   k3s-master     │  │  k3s-worker   │ │
                        │  │  172.31.84.x     │  │  172.31.88.x  │ │
                        │  │                  │  │               │ │
                        │  │  control-plane   │  │  worker       │ │
                        │  │  CoreDNS         │  │               │ │
                        │  │  MetalLB ctrl    │  │  MetalLB spkr │ │
                        │  │  Traefik         │  │               │ │
                        │  │  Longhorn mgr    │  │  Longhorn mgr │ │
                        │  │  Longhorn UI     │  │               │ │
                        │  │  PostgreSQL ──── │──│──► Volume     │ │
                        │  └──────────────────┘  └───────────────┘ │
                        │           │                    │         │
                        │      Flannel VXLAN (UDP 8472)            │
                        │           │                    │         │
                        │      Longhorn iSCSI replication          │
                        └──────────────────────────────────────────┘
```

**Key design decisions:**

- **k3s** instead of kubeadm — single binary, faster to set up, Flannel CNI and Traefik bundled. Represents the bare metal reality where you own the control plane.
- **Longhorn** instead of cloud storage — dynamically provisions block volumes across nodes using iSCSI. Without it, PVCs stay `Pending` forever on bare metal.
- **MetalLB** instead of cloud load balancer — assigns real IPs to `LoadBalancer` services. Without it, `EXTERNAL-IP` stays `<pending>` forever.
- **Traefik** instead of NGINX Ingress — ships with k3s and avoids compatibility issues with newer Kubernetes versions.
- **Self-authored Helm chart** — every file written manually, including `_helpers.tpl` with named templates.

---

## Prerequisites

- Two Linux VMs (Ubuntu 22.04) with at least 1 vCPU and 2GB RAM each
- Both VMs in the same network with unrestricted traffic between them
- SSH access to both nodes
- `kubectl` and `helm` installed on your local machine or master node

> **If using AWS EC2:** Add an inbound + outbound rule of `All traffic` from/to `172.31.0.0/16` (VPC CIDR) to the security group attached to both instances **before doing anything else**. Without this, Flannel VXLAN is blocked and the worker node cannot reach any ClusterIP address, causing cascading failures across DNS, Longhorn, and CSI components.

---

## Stack

| Component | Version | Purpose |
|-----------|---------|---------|
| k3s | v1.29+ | Kubernetes distribution |
| Longhorn | v1.6+ | Distributed block storage |
| MetalLB | v0.14.3 | Bare metal load balancer |
| Traefik | v2.x (bundled) | Ingress controller |
| PostgreSQL | 15 | Stateful workload |
| Helm | v3.x | Package manager |

---

## Repository Structure

```
bare-metal-k8s-homelab/
├── README.md
├── docs/
│   ├── architecture.md          # Design decisions and component explanations
│   ├── longhorn-setup.md        # Longhorn architecture and interview Q&A
│   └── troubleshooting.md       # Every issue hit during the build, with root cause and fix
├── infra/
│   ├── metallb-config.yaml      # IPAddressPool and L2Advertisement
│   ├── test-pvc.yaml            # Longhorn PVC smoke test
│   └── postgres-manual/         # Manual deployment (Phase 6 — before Helm)
│       ├── postgres-pvc.yaml
│       └── postgres-deployment.yaml
└── helm/
    └── postgres-chart/
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
            ├── _helpers.tpl     # Named templates for labels and selectors
            ├── secret.yaml
            ├── pvc.yaml
            ├── deployment.yaml
            └── service.yaml
```

---

## Setup

### 1. Bootstrap both nodes

```bash
# On both nodes
apt update && apt upgrade -y
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab
apt install -y curl open-iscsi nfs-common
systemctl enable --now iscsid
```

### 2. Install k3s master

```bash
curl -sfL https://get.k3s.io | sh -s - \
  --disable servicelb \
  --write-kubeconfig-mode 644
```

### 3. Join worker node

```bash
# Get token from master
cat /var/lib/rancher/k3s/server/node-token

# On worker
curl -sfL https://get.k3s.io | \
  K3S_URL=https://<MASTER_PRIVATE_IP>:6443 \
  K3S_TOKEN=<NODE_TOKEN> sh -
```

### 4. Verify cluster

```bash
kubectl get nodes
# NAME          STATUS   ROLES                  AGE
# k3s-master    Ready    control-plane,master   5m
# k3s-worker    Ready    <none>                 1m
```

### 5. Install MetalLB

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.3/config/manifests/metallb-native.yaml

# AWS/k3s webhook fix — see docs/troubleshooting.md
kubectl delete validatingwebhookconfigurations metallb-webhook-configuration 2>/dev/null || true
kubectl delete mutatingwebhookconfigurations metallb-webhook-configuration 2>/dev/null || true

kubectl apply -f infra/metallb-config.yaml
```

### 6. Install Longhorn

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update

helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace \
  --set defaultSettings.defaultReplicaCount=1

# AWS/k3s webhook fix — patch to Ignore instead of deleting
# (Longhorn recreates webhooks on every manager restart)
kubectl patch validatingwebhookconfiguration longhorn-webhook-validator \
  --type='json' \
  -p='[{"op":"replace","path":"/webhooks/0/failurePolicy","value":"Ignore"}]'

kubectl patch mutatingwebhookconfiguration longhorn-webhook-mutator \
  --type='json' \
  -p='[{"op":"replace","path":"/webhooks/0/failurePolicy","value":"Ignore"}]'
```

### 7. Verify Longhorn storage

```bash
kubectl get storageclass
# longhorn (default)   driver.longhorn.io

kubectl apply -f infra/test-pvc.yaml
kubectl get pvc
# test-pvc   Bound   — Longhorn provisioned the volume

kubectl delete -f infra/test-pvc.yaml
```

---

## Deploy PostgreSQL via Helm

```bash
helm install postgres-release helm/postgres-chart/ \
  --namespace postgres \
  --create-namespace

kubectl get all -n postgres
kubectl get pvc -n postgres
```

Override values at install time:

```bash
helm install postgres-release helm/postgres-chart/ \
  --namespace postgres \
  --create-namespace \
  --set postgres.password=mypassword \
  --set storage.size=5Gi
```

---

## Verify Storage Persistence (Node Failure Simulation)

```bash
# Write data
kubectl exec -it -n postgres \
  $(kubectl get pod -n postgres -l app.kubernetes.io/name=postgres-chart -o jsonpath='{.items[0].metadata.name}') \
  -- psql -U admin -d appdb \
  -c "CREATE TABLE test (id SERIAL, name TEXT); INSERT INTO test (name) VALUES ('data survived');"

# Simulate node failure — drain the worker
kubectl cordon k3s-worker
kubectl delete poddisruptionbudget -n longhorn-system --all
kubectl drain k3s-worker --ignore-daemonsets --delete-emptydir-data --force

# Watch pod reschedule on master
kubectl get pods -n postgres -o wide -w

# Verify data survived
kubectl exec -it -n postgres \
  $(kubectl get pod -n postgres -l app.kubernetes.io/name=postgres-chart -o jsonpath='{.items[0].metadata.name}') \
  -- psql -U admin -d appdb -c "SELECT * FROM test;"
# Returns: 1 | data survived

# Restore worker
kubectl uncordon k3s-worker
```

---

## Helm Operations

```bash
# Upgrade with inline value override
helm upgrade postgres-release helm/postgres-chart/ \
  --namespace postgres \
  --set storage.size=5Gi

# Upgrade with a values file
helm upgrade postgres-release helm/postgres-chart/ \
  --namespace postgres \
  --values custom-values.yaml

# View release history
helm history postgres-release -n postgres

# Rollback to revision 1
helm rollback postgres-release 1 -n postgres

# Uninstall
helm uninstall postgres-release -n postgres
```

---

## What I Learned

**Bare metal vs managed Kubernetes is not just infrastructure — it changes how you think about every component.**

On EKS, a PVC binding, a LoadBalancer IP, and an Ingress just work. On bare metal, each one requires understanding and a deliberate solution: Longhorn for storage, MetalLB for load balancing, Traefik for ingress. You realize how much work cloud providers abstract away — and you appreciate it more once you've done it yourself.

**The hardest problem was networking, not Kubernetes.** Every mysterious pod failure traced back to a single missing AWS security group rule. Flannel VXLAN requires UDP 8472 between nodes — without it, the worker node cannot reach any ClusterIP address, which breaks DNS, which breaks every service discovery call in the cluster. The lesson: always verify inter-node connectivity first.

**Admission webhooks are a recurring failure mode on bare metal.** MetalLB, Longhorn, and any component with `failurePolicy: Fail` webhooks will break on setups where the API server cannot reach ClusterIP services. Understanding why — and knowing the fix (`failurePolicy: Ignore` or webhook deletion) — is the kind of operational depth that separates someone who has run Kubernetes from someone who has only read about it.

**Writing a Helm chart from scratch is different from using one.** Understanding `_helpers.tpl`, named templates, and the `.Release` / `.Chart` / `.Values` object hierarchy makes chart debugging intuitive rather than confusing. The `subPath` fix for PostgreSQL's `lost+found` issue is a good example — knowing why it's needed (ext4 formats with `lost+found` at volume root) makes it something you apply confidently, not cargo-cult.

---

## Troubleshooting

See [`docs/troubleshooting.md`](docs/troubleshooting.md) for documented issues including:

- AWS security group blocking inter-node Flannel traffic
- Admission webhook timeouts on k3s + AWS EC2
- PostgreSQL refusing to start due to `lost+found` in Longhorn volume
- Longhorn manager race condition on first startup
- PodDisruptionBudget blocking node drain
