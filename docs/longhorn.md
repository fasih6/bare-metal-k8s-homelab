# Longhorn — Comprehensive Notes
## Distributed Block Storage for Kubernetes

> Part of `fasih6/My_Devops_Notes` — Bare Metal Kubernetes Series

---

## Table of Contents

1. [What is Longhorn?](#1-what-is-longhorn)
2. [Why Longhorn on Bare Metal?](#2-why-longhorn-on-bare-metal)
3. [Core Concepts](#3-core-concepts)
4. [Architecture Deep Dive](#4-architecture-deep-dive)
5. [How a Volume is Provisioned](#5-how-a-volume-is-provisioned)
6. [How Replication Works](#6-how-replication-works)
7. [Installation](#7-installation)
8. [Storage Classes](#8-storage-classes)
9. [Working with Volumes](#9-working-with-volumes)
10. [Snapshots and Backups](#10-snapshots-and-backups)
11. [Node Failure and Recovery](#11-node-failure-and-recovery)
12. [Longhorn UI](#12-longhorn-ui)
13. [Key Settings and Tuning](#13-key-settings-and-tuning)
14. [Troubleshooting](#14-troubleshooting)
15. [Interview Q&A](#15-interview-qa)

---

## 1. What is Longhorn?

Longhorn is an **open-source, cloud-native distributed block storage system** for Kubernetes. It was originally built by Rancher Labs and is now a **CNCF incubating project**.

It solves a fundamental problem on bare metal Kubernetes: **where do PersistentVolumes come from?**

On managed Kubernetes (EKS, AKS, GKE), cloud providers handle this automatically:
- AWS EKS → Amazon EBS
- Azure AKS → Azure Disk
- GKE → Google Persistent Disk

On bare metal, there is no cloud provider. Without a storage backend, any PVC you create will stay in `Pending` state forever. Longhorn fills this gap.

### What Longhorn Does

- Dynamically provisions PersistentVolumes on bare metal nodes
- Replicates volume data across multiple nodes for high availability
- Provides snapshots, backups, and restore functionality
- Exposes a web UI for storage management
- Integrates natively with Kubernetes via a StorageClass and CSI driver

### Key Facts

| Property | Value |
|----------|-------|
| License | Apache 2.0 |
| CNCF Status | Incubating |
| Storage type | Block storage |
| Protocol | iSCSI / NVMe-oF |
| Minimum nodes | 1 (1 replica), recommended 3 (3 replicas) |
| Kubernetes requirement | v1.21+ |

---

## 2. Why Longhorn on Bare Metal?

### The Problem Without Longhorn

```
kubectl apply -f pvc.yaml

kubectl get pvc
NAME       STATUS    VOLUME   CAPACITY   STORAGECLASS
my-pvc     Pending   —        —          standard
```

Status stays `Pending` indefinitely. No storage provisioner exists to fulfil the claim.

### The Problem With Local Storage

You could use `hostPath` or `local` volumes — but these bind a pod to a specific node permanently. If that node fails, the pod cannot be rescheduled. The data is stuck on the dead node.

### Why Longhorn Solves Both

Longhorn:
1. Acts as a **dynamic storage provisioner** — creates PVs automatically when a PVC is created
2. **Replicates** the volume data across multiple nodes
3. When a node fails, the replica on another node becomes the new primary — the pod reschedules and the volume reattaches on the new node

This is what real distributed storage looks like on bare metal.

---

## 3. Core Concepts

### Volume

A Longhorn Volume is a virtual block device. It appears to a pod as a regular disk at a mountPath. Internally it is stored as a collection of files on the host nodes' disks.

### Replica

Each volume has one or more **replicas** — full copies of the volume data stored on different nodes. The number of replicas is defined by the `numberOfReplicas` setting (default: 3).

```
Volume: my-pvc (3 replicas)
├── Replica A → stored on node-1 at /var/lib/longhorn/
├── Replica B → stored on node-2 at /var/lib/longhorn/
└── Replica C → stored on node-3 at /var/lib/longhorn/
```

If node-2 fails, replicas A and C still have the full data. The volume continues operating.

### Engine

The **Longhorn Engine** is the controller process for a volume. It sits between the pod and the replicas, handling all reads and writes. Each volume has exactly one engine running on the same node as the pod using the volume.

```
Pod → Engine (on node where pod runs) → Replica A, Replica B, Replica C
```

### Frontend

The frontend is how the volume is exposed to the pod. Longhorn supports:
- **tgt-blockdev** — iSCSI-based, most common
- **NVMe-oF** — newer, lower latency (experimental)

### StorageClass

A Kubernetes StorageClass that references Longhorn as the provisioner. When a PVC references this StorageClass, Longhorn automatically creates and manages the volume.

### CSI Driver

Longhorn integrates with Kubernetes via the **Container Storage Interface (CSI)**. CSI is a standardised API that lets storage vendors plug into Kubernetes without modifying core code.

When you create a PVC:
1. Kubernetes calls the CSI driver
2. The CSI driver calls the Longhorn manager
3. Longhorn creates the volume and replicas
4. Kubernetes binds the PVC to the new PV

---

## 4. Architecture Deep Dive

### Components

```
┌─────────────────────────────────────────────┐
│              Kubernetes Cluster              │
│                                             │
│  ┌──────────────┐    ┌──────────────────┐   │
│  │  Longhorn    │    │  Longhorn        │   │
│  │  Manager     │    │  UI              │   │
│  │  (DaemonSet) │    │  (Deployment)    │   │
│  └──────┬───────┘    └──────────────────┘   │
│         │                                   │
│  ┌──────▼───────────────────────────────┐   │
│  │           Node 1                     │   │
│  │  ┌─────────────┐  ┌───────────────┐  │   │
│  │  │   Engine    │  │   Replica     │  │   │
│  │  │  (per vol)  │  │  /var/lib/    │  │   │
│  │  └─────────────┘  │   longhorn/   │  │   │
│  │                   └───────────────┘  │   │
│  └──────────────────────────────────────┘   │
│                                             │
│  ┌──────────────────────────────────────┐   │
│  │           Node 2                     │   │
│  │                   ┌───────────────┐  │   │
│  │                   │   Replica     │  │   │
│  │                   │  /var/lib/    │  │   │
│  │                   │   longhorn/   │  │   │
│  │                   └───────────────┘  │   │
│  └──────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

### Longhorn Manager

- Runs as a **DaemonSet** — one pod per node
- Handles volume lifecycle: create, attach, detach, delete
- Synchronises state with the Kubernetes API
- Monitors replica health and triggers rebuilding if a replica is degraded

### Longhorn Engine

- Runs as a **separate process** on the node where the volume is attached
- Handles all I/O between the pod and the replicas
- Replicates writes synchronously to all replicas
- If a replica is slow or unavailable, the engine marks it as failed and continues with remaining replicas

### Instance Manager

- Manages the lifecycle of engine and replica processes on each node
- There is one instance manager pod per node

### CSI Plugin

- Runs as a DaemonSet
- Implements the CSI spec so Kubernetes can call Longhorn for volume operations
- Handles: CreateVolume, DeleteVolume, ControllerPublishVolume (attach), NodeStageVolume (mount)

---

## 5. How a Volume is Provisioned

Step-by-step flow when you apply a PVC:

```
1. You apply a PVC referencing storageClassName: longhorn

2. Kubernetes PersistentVolume controller sees an unbound PVC
   → calls Longhorn CSI provisioner

3. Longhorn Manager creates a Volume object in the Longhorn CRD store
   → decides how many replicas (default: 3)
   → selects nodes for replica placement (avoids same node twice)

4. Longhorn Manager creates Replica processes on the selected nodes
   → each replica creates a sparse file at /var/lib/longhorn/replicas/

5. Longhorn Manager creates an Engine process on the scheduling node
   → Engine connects to all replicas

6. Longhorn CSI driver creates a Kubernetes PersistentVolume object
   → binds it to your PVC

7. PVC status changes from Pending → Bound

8. When your pod starts and mounts the PVC:
   → CSI NodeStageVolume is called
   → volume is formatted (ext4 by default) and mounted at the pod's mountPath
```

### What You See

```bash
kubectl get pvc
NAME           STATUS   VOLUME                                     CAPACITY   STORAGECLASS
postgres-pvc   Bound    pvc-3f2a1b4c-...                          2Gi        longhorn

kubectl get pv
NAME                       CAPACITY   RECLAIM POLICY   STATUS   STORAGECLASS
pvc-3f2a1b4c-...           2Gi        Delete           Bound    longhorn
```

---

## 6. How Replication Works

### Write Path

When a pod writes data to a Longhorn volume:

```
Pod
 └─→ Engine (synchronous write to ALL replicas)
       ├─→ Replica A (Node 1) ✓
       ├─→ Replica B (Node 2) ✓
       └─→ Replica C (Node 3) ✓
            All must acknowledge → write confirmed to pod
```

Writes are **synchronous** — the engine waits for ALL healthy replicas to confirm before acknowledging the write to the pod. This guarantees consistency.

### Read Path

Reads are served from the local replica if available, otherwise from any healthy replica.

### What Happens When a Replica Fails

```
Node 2 goes down → Replica B is unreachable

Engine marks Replica B as ERR
Engine continues with Replica A and Replica C (quorum maintained)
Volume stays healthy with 2/3 replicas

Longhorn Manager detects degraded volume
→ schedules a new Replica on Node 4 (or Node 1/3 if no Node 4)
→ rebuilds from an existing replica

Once rebuilt: volume returns to fully healthy (3/3 replicas)
```

The pod **never stops running** during replica failure if quorum is maintained.

### Quorum Rules

| Total Replicas | Max Failed Replicas (still healthy) |
|---------------|--------------------------------------|
| 1 | 0 — any failure stops the volume |
| 2 | 0 — any failure makes volume read-only |
| 3 | 1 — one failure tolerated |
| 5 | 2 — two failures tolerated |

> For a 2-node homelab like ours, use `numberOfReplicas: 1`. Replication across 2 nodes means either node failure stops the volume anyway since you can't maintain quorum.

---

## 7. Installation

### Prerequisites

On **every node** in the cluster:

```bash
# iSCSI initiator — required for Longhorn to attach volumes
apt install -y open-iscsi
systemctl enable iscsid
systemctl start iscsid

# NFS client — required for RWX volumes and backups
apt install -y nfs-common

# Check all prerequisites
curl -sSfL https://raw.githubusercontent.com/longhorn/longhorn/v1.6.0/scripts/environment_check.sh | bash
```

### Install via Helm (Recommended)

```bash
# Add repo
helm repo add longhorn https://charts.longhorn.io
helm repo update

# Install with default settings
helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace

# Install with custom replica count (for 2-node homelab)
helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace \
  --set defaultSettings.defaultReplicaCount=1

# Verify all pods are running
kubectl get pods -n longhorn-system
```

### Expected Pods After Installation

```
NAME                                        READY   STATUS
longhorn-manager-xxxxx                      1/1     Running   # DaemonSet — one per node
longhorn-driver-deployer-xxxxx              1/1     Running
longhorn-ui-xxxxx                           1/1     Running
csi-attacher-xxxxx                          1/1     Running
csi-provisioner-xxxxx                       1/1     Running
csi-resizer-xxxxx                           1/1     Running
csi-snapshotter-xxxxx                       1/1     Running
engine-image-ei-xxxxx                       1/1     Running   # DaemonSet — one per node
instance-manager-xxxxx                      1/1     Running   # DaemonSet — one per node
```

### Install via kubectl (Alternative)

```bash
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.6.0/deploy/longhorn.yaml
```

---

## 8. Storage Classes

### Default Longhorn StorageClass

After installation, Longhorn creates a default StorageClass:

```bash
kubectl get storageclass
NAME                 PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE
longhorn (default)   driver.longhorn.io   Delete          Immediate
```

### Custom StorageClass

You can create additional StorageClasses with different settings:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-fast
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Retain          # Keep PV after PVC is deleted
volumeBindingMode: Immediate
parameters:
  numberOfReplicas: "3"
  staleReplicaTimeout: "2880"  # Minutes before a stale replica is replaced
  fromBackup: ""               # Restore from backup URL if set
  diskSelector: "ssd"          # Only use nodes/disks tagged with "ssd"
  nodeSelector: "storage"      # Only use nodes tagged with "storage"
  recurringJobSelector: '[{"name":"snap","isGroup":true}]'
```

### Reclaim Policies

| Policy | Behaviour when PVC is deleted |
|--------|-------------------------------|
| `Delete` (default) | PV and volume data are permanently deleted |
| `Retain` | PV remains, data preserved, manual cleanup needed |

> Use `Retain` for production databases. Use `Delete` for stateless or test workloads.

### Volume Access Modes

| Mode | Description | Longhorn Support |
|------|-------------|-----------------|
| `ReadWriteOnce` (RWO) | One node can read/write | ✅ Full support |
| `ReadWriteMany` (RWX) | Multiple nodes read/write | ✅ Via NFS share layer |
| `ReadOnlyMany` (ROX) | Multiple nodes read-only | ✅ |

---

## 9. Working with Volumes

### Create a PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: postgres
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 2Gi
```

```bash
kubectl apply -f pvc.yaml

kubectl get pvc -n postgres
# NAME           STATUS   VOLUME         CAPACITY   STORAGECLASS
# postgres-pvc   Bound    pvc-xxx...     2Gi        longhorn
```

### Expand a Volume

Longhorn supports **online volume expansion** — no downtime required.

```bash
# Edit the PVC and increase the storage request
kubectl edit pvc postgres-pvc -n postgres
# Change: storage: 2Gi → storage: 5Gi

# Verify expansion
kubectl get pvc postgres-pvc -n postgres
# CAPACITY shows 5Gi once complete
```

The StorageClass must have `allowVolumeExpansion: true` (default for Longhorn).

### Check Volume Details

```bash
# Via Longhorn CRD
kubectl get volumes.longhorn.io -n longhorn-system

# Detailed view of a specific volume
kubectl describe volumes.longhorn.io pvc-xxx -n longhorn-system
```

### Manually Attach/Detach

```bash
# List all Longhorn volumes
kubectl get volumes -n longhorn-system

# The Longhorn UI is easier for manual operations — see Section 12
```

---

## 10. Snapshots and Backups

### Snapshots

A snapshot captures the state of a volume at a point in time. Snapshots are stored locally on the same nodes as the volume replicas.

```yaml
# Create a snapshot manually
apiVersion: longhorn.io/v1beta2
kind: Snapshot
metadata:
  name: postgres-snap-manual
  namespace: longhorn-system
spec:
  volume: pvc-xxx-yyy-zzz
```

Or via the UI: Volumes → Select volume → Create Snapshot

**Recurring snapshots** (automated):

```yaml
apiVersion: longhorn.io/v1beta2
kind: RecurringJob
metadata:
  name: daily-snapshot
  namespace: longhorn-system
spec:
  cron: "0 2 * * *"    # Every day at 02:00
  task: snapshot
  retain: 7             # Keep last 7 snapshots
  concurrency: 1
```

### Backups

Backups copy snapshot data to an **external storage target** — S3, NFS, or Azure Blob. Unlike snapshots, backups survive if the entire cluster is lost.

#### Configure Backup Target (S3 Example)

```bash
# Create secret with S3 credentials
kubectl create secret generic longhorn-backup-secret \
  --from-literal=AWS_ACCESS_KEY_ID=AKIAXXXXXXXX \
  --from-literal=AWS_SECRET_ACCESS_KEY=xxxxxxxxxx \
  --from-literal=AWS_ENDPOINTS=https://s3.amazonaws.com \
  -n longhorn-system
```

In Longhorn UI → Settings → Backup Target:
```
s3://your-bucket-name@eu-central-1/
```

#### Trigger a Backup

```bash
# Via kubectl
kubectl -n longhorn-system create -f - <<EOF
apiVersion: longhorn.io/v1beta2
kind: Backup
metadata:
  name: postgres-backup-1
spec:
  snapshotName: postgres-snap-manual
  backupMode: full
EOF
```

#### Restore from Backup

Create a new StorageClass pointing to the backup:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-restore
provisioner: driver.longhorn.io
parameters:
  fromBackup: "s3://your-bucket/backups?volume=pvc-xxx&backup=backup-yyy"
  numberOfReplicas: "1"
```

Then create a PVC using this StorageClass — Longhorn restores the data automatically.

---

## 11. Node Failure and Recovery

This is the most important section for understanding what Longhorn actually does.

### Scenario: Worker Node Failure

Setup: 2-node cluster (master + worker), postgres pod running on worker, volume with 1 replica on worker.

```bash
# Step 1 — Check where everything is running
kubectl get pods -n postgres -o wide
# postgres-xxx   Running   worker-node

kubectl get volumes -n longhorn-system
# Status: healthy, replicas: 1/1
```

```bash
# Step 2 — Simulate worker node failure
# Option A: power off the VM in Hetzner dashboard
# Option B: cordon and drain
kubectl cordon k3s-worker
kubectl drain k3s-worker --ignore-daemonsets --delete-emptydir-data --force
```

```bash
# Step 3 — Watch what happens
kubectl get pods -n postgres -o wide --watch
# postgres-xxx   Terminating   worker-node
# postgres-yyy   Pending       <none>
# postgres-yyy   Running       k3s-master   ← rescheduled on master
```

```bash
# Step 4 — Verify data survived
kubectl exec -it -n postgres \
  $(kubectl get pod -n postgres -l app=postgres -o jsonpath='{.items[0].metadata.name}') \
  -- psql -U admin -d appdb -c "SELECT * FROM test;"
# Returns your test data — Longhorn reattached the volume on the new node
```

```bash
# Step 5 — Restore worker node
kubectl uncordon k3s-worker
```

### What Longhorn Did Internally

1. Node becomes unreachable → Longhorn manager detects replica is down
2. Volume enters **degraded** state (replicas: 0/1 in this single-replica setup)
3. Kubernetes reschedules pod on master node
4. Longhorn engine reattaches the volume on master
5. Pod mounts the volume — data is intact
6. Once worker comes back, Longhorn optionally rebuilds replica on worker

### Volume States

| State | Meaning |
|-------|---------|
| `healthy` | All replicas are in sync and available |
| `degraded` | Some replicas are down but volume still accessible |
| `faulted` | No replicas available — volume is inaccessible |
| `detached` | Volume exists but no pod is using it |

---

## 12. Longhorn UI

The Longhorn UI provides a dashboard for managing volumes, nodes, snapshots, and backups.

### Access the UI

By default the UI is only accessible inside the cluster. Expose it with port-forward:

```bash
kubectl port-forward -n longhorn-system svc/longhorn-frontend 8080:80
# Access at: http://localhost:8080
```

Or create an Ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: longhorn-ingress
  namespace: longhorn-system
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: longhorn.your-domain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: longhorn-frontend
            port:
              number: 80
```

### UI Overview

- **Dashboard** — cluster-wide storage health, capacity, replica status
- **Volumes** — list all volumes, create/delete/attach/detach, create snapshots
- **Nodes** — view disk usage per node, add/remove disks, set scheduling tags
- **Recurring Jobs** — configure automated snapshots and backups
- **Backups** — manage backup targets, browse backup history, trigger restores
- **Settings** — configure defaults (replica count, data path, backup target)

---

## 13. Key Settings and Tuning

Access settings via: Longhorn UI → Settings, or by editing the ConfigMap.

### Most Important Settings

| Setting | Default | Description |
|---------|---------|-------------|
| `default-replica-count` | 3 | Number of replicas for new volumes |
| `storage-over-provisioning-percentage` | 200 | How much more storage Longhorn can allocate vs physical disk |
| `storage-minimal-available-percentage` | 25 | Longhorn stops scheduling replicas if free disk < this % |
| `node-down-pod-deletion-policy` | do-nothing | What to do with pods on a failed node |
| `auto-salvage` | true | Automatically recover faulted volumes if a replica is available |
| `replica-auto-balance` | disabled | Automatically rebalance replicas across nodes |
| `backup-target` | — | External backup destination (S3/NFS URL) |
| `concurrent-replica-rebuild-per-node-limit` | 5 | Max simultaneous replica rebuilds per node |

### Configure via Helm

```bash
helm upgrade longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --set defaultSettings.defaultReplicaCount=1 \
  --set defaultSettings.storageOverProvisioningPercentage=150 \
  --set defaultSettings.nodeDownPodDeletionPolicy=delete-both-statefulset-and-deployment-pod
```

### Data Path

By default Longhorn stores replica data at:
```
/var/lib/longhorn/
```

You can change this per node in the UI under Nodes → Edit Node → Disk. Useful if you have a separate data disk mounted at `/data`.

---

## 14. Troubleshooting

### PVC Stuck in Pending

```bash
kubectl describe pvc my-pvc -n my-namespace
# Look for Events at the bottom

# Common causes:
# 1. No nodes have enough disk space
# 2. iSCSI not installed on nodes
# 3. Longhorn manager pod not running on target node
```

```bash
# Check Longhorn manager logs
kubectl logs -n longhorn-system -l app=longhorn-manager --tail=50
```

### Volume Stuck in Attaching

```bash
# Check instance manager logs
kubectl logs -n longhorn-system -l app=longhorn-instance-manager --tail=50

# Check if iSCSI is running on the node
systemctl status iscsid
```

### Volume Shows Degraded

```bash
kubectl get volumes -n longhorn-system
# Find your volume name, check replica count

kubectl describe volumes.longhorn.io <volume-name> -n longhorn-system
# Shows which replicas are healthy and which are failed
```

Degraded is not an emergency — volume is still accessible. Longhorn will rebuild the failed replica automatically.

### Node Not Schedulable for Replicas

```bash
# Check node status in Longhorn
kubectl get nodes.longhorn.io -n longhorn-system

# A node may be excluded if:
# - Disk usage exceeds storage-minimal-available-percentage
# - Node is cordoned in Kubernetes
# - Node scheduling is disabled in Longhorn UI
```

### Useful Debugging Commands

```bash
# All Longhorn custom resources
kubectl get volumes,replicas,engines,nodes -n longhorn-system

# Longhorn events
kubectl get events -n longhorn-system --sort-by='.lastTimestamp'

# Check CSI driver is registered
kubectl get csidrivers
# Should show: driver.longhorn.io

# Check CSI node info
kubectl get csinodes
```

---

## 15. Interview Q&A

**Q: What is Longhorn and why do you need it on bare metal Kubernetes?**

On managed Kubernetes, cloud providers handle persistent storage via EBS, Azure Disk, etc. On bare metal there is no such backend — PVCs will stay Pending indefinitely. Longhorn solves this by acting as a dynamic storage provisioner and a distributed block storage system. It creates PersistentVolumes backed by replicated data across your nodes.

---

**Q: How does Longhorn replicate data?**

Each Longhorn volume has a configured number of replicas — by default 3. The Longhorn Engine synchronously writes every I/O operation to all healthy replicas before confirming the write to the pod. If a replica goes down, the engine continues with remaining replicas and marks the volume as degraded. Longhorn then automatically rebuilds a new replica on a healthy node to restore full redundancy.

---

**Q: What happens when a node fails in a Longhorn setup?**

The Longhorn Manager detects the node is unreachable and marks its replica as failed. The volume enters degraded state but remains accessible via surviving replicas. Kubernetes reschedules the pod onto a healthy node. Longhorn reattaches the volume on that node. Once the failed node recovers, Longhorn can rebuild the replica. Throughout this process, the pod's data is intact.

---

**Q: What is the Longhorn Engine?**

The Engine is a per-volume controller process that runs on the same node as the pod using the volume. It handles all I/O between the pod and the replicas, replicating writes synchronously. There is exactly one engine per volume, and it follows the pod if the pod is rescheduled.

---

**Q: What is the difference between a Longhorn snapshot and a backup?**

A snapshot captures the volume state at a point in time and is stored locally on the cluster nodes alongside the replica data. If the entire cluster is lost, snapshots are lost too. A backup copies snapshot data to an external target — S3, NFS, or Azure Blob — so it survives cluster loss. Backups are slower but provide true disaster recovery capability.

---

**Q: How does Longhorn integrate with Kubernetes?**

Via the Container Storage Interface (CSI). Longhorn implements the CSI spec and registers itself as a CSI driver (`driver.longhorn.io`). When a PVC is created with `storageClassName: longhorn`, Kubernetes calls Longhorn's CSI provisioner to create the volume, then calls the CSI node plugin to attach and mount it when a pod starts.

---

**Q: What replica count would you use on a 2-node cluster and why?**

I would use `numberOfReplicas: 1`. With only 2 nodes, replicating across both means that either node failure would still cause the volume to become inaccessible since you cannot maintain quorum with 1 of 2 replicas. The replica redundancy only provides real benefit with 3 or more nodes where losing one node still leaves a majority of replicas healthy.

---

**Q: How do you expand a Longhorn volume?**

Edit the PVC and increase the storage request. Longhorn supports online volume expansion — no pod restart is required. The StorageClass must have `allowVolumeExpansion: true`, which is the default for Longhorn. The filesystem inside the volume is also expanded automatically.

---

**Q: How would you back up a Longhorn volume to S3?**

Create a Kubernetes secret with the AWS credentials, configure the S3 bucket URL as the backup target in Longhorn settings, then either create a Backup resource manually or configure a RecurringJob to automate it on a schedule. Restoring is done by creating a StorageClass with `fromBackup` pointing to the backup URL, then creating a PVC that references that StorageClass.

---

**Q: What is storage over-provisioning in Longhorn?**

Longhorn stores replica data as sparse files — the file is allocated at the full volume size but only uses actual disk space for written data. This means you can provision more total volume capacity than your physical disk has. The `storage-over-provisioning-percentage` setting controls this — at 200% (default), on a 100GB disk you can provision up to 200GB of volumes. This works safely as long as actual data written stays within physical limits.

---

*Notes by Fasih-Ur-Rehman Shahid | github.com/fasih6/My_Devops_Notes*
