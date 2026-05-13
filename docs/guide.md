# Bare Metal Kubernetes Project Guide
## k3s + Longhorn + Helm — Portfolio Project for Mittwald Application

---

## Overview

**Goal:** Deploy a stateful application on a bare metal Kubernetes cluster using k3s, provision persistent storage with Longhorn, and package everything with a self-authored Helm chart.

**Infrastructure:** 2 EC2 instances on AWS (or 2 VMs on Hetzner Cloud ~€6/month total)
**Time estimate:** 4–5 weeks alongside other work
**GitHub repo:** Create a dedicated repo, e.g. `bare-metal-k8s-homelab`

**Stack:**
- k3s (lightweight Kubernetes, no managed control plane)
- Longhorn (distributed block storage for bare metal)
- Helm (self-authored chart)
- PostgreSQL (stateful app to deploy)
- MetalLB (load balancer without cloud provider)
- Traefik Ingress Controller (bundled with k3s)

---

## ⚠️ Critical: AWS Security Group Configuration (Do This First)

> **If you skip this, you will spend hours debugging mysterious pod failures on the worker node. This is the most important prerequisite.**

When running a k3s cluster on AWS EC2, both instances must be able to communicate freely with each other on all internal ports. Without this, the worker node cannot reach any ClusterIP address — which means DNS is broken, service discovery fails, and every pod that tries to connect to another service will time out.

**The root cause chain:**
- Missing security group rule → Flannel VXLAN (UDP 8472) blocked → worker cannot route to ClusterIP addresses → CoreDNS unreachable from worker → all DNS lookups time out → every pod on the worker that tries to resolve a service name fails → cascading failures across Longhorn, CSI, and any other component

**Fix:** Go to AWS Console → EC2 → Security Groups → find the group attached to both instances → Edit inbound rules → Add:

| Type | Protocol | Port | Source |
|------|----------|------|--------|
| All traffic | All | All | `172.31.0.0/16` |

Also add to outbound rules:

| Type | Protocol | Port | Destination |
|------|----------|------|-------------|
| All traffic | All | All | `172.31.0.0/16` |

`172.31.0.0/16` is the default AWS VPC CIDR — it covers all instances in your VPC. If you used a custom VPC, substitute your actual VPC CIDR.

**Verify connectivity from the worker node before proceeding:**
```bash
# Run on the WORKER node — "Empty reply from server" means success (connection reached)
curl --max-time 5 http://<kube-dns-clusterip>:53
# Get the kube-dns ClusterIP with: kubectl get svc -n kube-system kube-dns
```

If it times out instead of returning "Empty reply", the security group is still blocking traffic.

---

## Phase 1 — Infrastructure Setup (Week 1)

### Step 1: Create VMs

**Option A: AWS EC2**
- Launch 2 Ubuntu 22.04 instances (t2.micro or t3.small)
- Place both in the same VPC and security group
- Configure the security group as described above before doing anything else
- Note both private IPs (172.31.x.x)

**Option B: Hetzner Cloud (~€6/month total)**
1. Sign up at [hetzner.com/cloud](https://hetzner.com/cloud)
2. Create two servers:
   - **Type:** CX11 (1 vCPU, 2GB RAM)
   - **OS:** Ubuntu 22.04 LTS
   - **Location:** Nuremberg or Falkenstein (closest to Heilbronn)
   - **Names:** `k3s-master` and `k3s-worker`
3. Add your SSH public key during creation
4. Note both public and private IPs

> **Why not EKS?** EKS manages the control plane, storage, and load balancers for you. Using k3s on raw VMs means you own everything — CNI networking, storage provisioning, ingress. That is what "bare metal" means in this context and exactly what Mittwald was testing for.

### Step 2: Initial Server Configuration

Run on **both** servers:

```bash
apt update && apt upgrade -y

# Disable swap (required for Kubernetes)
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab

# On master:
hostnamectl set-hostname k3s-master
# On worker:
hostnamectl set-hostname k3s-worker

apt install -y curl wget vim htop open-iscsi nfs-common
systemctl enable iscsid
systemctl start iscsid
```

> `open-iscsi` is required by Longhorn. Install it now on both nodes — if you skip it, Longhorn will install but volumes will fail to attach.

### Step 3: Configure /etc/hosts

On **both** servers:

```bash
echo "YOUR_MASTER_PRIVATE_IP  k3s-master" >> /etc/hosts
echo "YOUR_WORKER_PRIVATE_IP  k3s-worker" >> /etc/hosts
```

Use private IPs, not public ones.

---

## Phase 2 — Install k3s Cluster (Week 1)

### Step 4: Install k3s on Master Node

```bash
curl -sfL https://get.k3s.io | sh -s - \
  --disable servicelb \
  --write-kubeconfig-mode 644
```

> `--disable servicelb` — disables k3s's built-in load balancer so MetalLB can manage it instead.
> `--write-kubeconfig-mode 644` — lets non-root users read the kubeconfig.
>
> Note: Unlike the original plan, we keep Traefik enabled (do not pass `--disable traefik`). The NGINX Ingress Controller's upstream manifest uses a deprecated `RuntimeClass` that is removed in newer Kubernetes versions and will fail to install. Traefik ships with k3s and works out of the box.

Verify after 30 seconds:

```bash
kubectl get nodes
# k3s-master   Ready   control-plane,master
```

Get the node token:

```bash
cat /var/lib/rancher/k3s/server/node-token
```

### Step 5: Join Worker Node

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://YOUR_MASTER_PRIVATE_IP:6443 \
  K3S_TOKEN=YOUR_NODE_TOKEN sh -
```

### Step 6: Verify Cluster

```bash
kubectl get nodes
# NAME          STATUS   ROLES                  AGE
# k3s-master    Ready    control-plane,master   5m
# k3s-worker    Ready    <none>                 1m
```

---

## Phase 3 — Install MetalLB (Week 2)

MetalLB provides LoadBalancer-type Services without a cloud provider. Without it, Services of type LoadBalancer stay in `<pending>` forever.

### Step 7: Install MetalLB

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.3/config/manifests/metallb-native.yaml

kubectl wait --namespace metallb-system \
  --for=condition=ready pod \
  --selector=app=metallb \
  --timeout=90s
```

### ⚠️ MetalLB Webhook Fix (AWS/k3s Issue)

MetalLB installs admission webhooks. On AWS EC2 with k3s + Flannel CNI, the API server cannot reach webhook endpoints over ClusterIP, causing webhook calls to time out. This is a known limitation of Flannel's VXLAN routing on EC2.

**Symptom:** MetalLB pods stuck, resources fail to apply with webhook timeout errors.

**Fix — delete the webhook configurations after install:**

```bash
kubectl delete validatingwebhookconfigurations metallb-webhook-configuration 2>/dev/null || true
kubectl delete mutatingwebhookconfigurations metallb-webhook-configuration 2>/dev/null || true
```

> This is safe for a homelab/portfolio cluster. The webhooks validate MetalLB resource configuration — without them, misconfigured resources will fail at runtime rather than at apply time, which is acceptable here.

### Step 8: Configure IP Address Pool

Create `metallb-config.yaml`:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - YOUR_MASTER_PUBLIC_IP/32
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
```

```bash
kubectl apply -f metallb-config.yaml
```

---

## Phase 4 — Ingress Controller (Week 2)

Traefik is already running — k3s installs it by default. Verify:

```bash
kubectl get pods -n traefik
kubectl get svc -n traefik
# traefik   LoadBalancer   <pending or IP>
```

MetalLB should assign a public IP to the Traefik service. Verify:

```bash
kubectl get svc -n traefik
# TYPE: LoadBalancer, EXTERNAL-IP: YOUR_MASTER_PUBLIC_IP
```

> **Why Traefik instead of NGINX?** The NGINX Ingress Controller upstream manifest references `RuntimeClass` which was removed in Kubernetes 1.30+. k3s ships with a recent Kubernetes version, so the NGINX manifest fails. Traefik works identically for ingress purposes and avoids this compatibility issue entirely.

---

## Phase 5 — Install Longhorn (Week 2)

Longhorn provides distributed block storage — the bare metal equivalent of EBS or EFS. Without it, PVCs on bare metal stay in `Pending` forever.

### Step 9: Install Longhorn via Helm

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update

helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace \
  --set defaultSettings.defaultReplicaCount=1
```

> `defaultReplicaCount=1` — use 1 replica since we have only 2 nodes. In production you would use 3.

### ⚠️ Longhorn Webhook Fix (AWS/k3s Issue)

Same Flannel routing issue as MetalLB. Longhorn installs admission webhooks that time out on AWS EC2 + k3s. The `longhorn-manager` pod enters a fatal loop waiting for its own webhook endpoint.

**Symptom:** `longhorn-manager` pod logs show:
```
level=fatal msg="Error starting webhooks: admission webhook service is not accessible
on cluster after 2m0s sec: timed out waiting for endpoint
https://longhorn-admission-webhook.longhorn-system.svc:9502/v1/healthz"
```

**Fix — patch webhooks to `Ignore` failure policy instead of deleting them:**

```bash
# Check webhook names
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations

# Patch to Ignore (safer than deleting — Longhorn recreates them on manager restart)
kubectl patch validatingwebhookconfiguration longhorn-webhook-validator \
  --type='json' \
  -p='[{"op":"replace","path":"/webhooks/0/failurePolicy","value":"Ignore"}]'

kubectl patch mutatingwebhookconfiguration longhorn-webhook-mutator \
  --type='json' \
  -p='[{"op":"replace","path":"/webhooks/0/failurePolicy","value":"Ignore"}]'
```

> **Why patch instead of delete?** Longhorn recreates its webhooks every time the manager pod restarts. Deleting them starts a race — the webhooks come back before the manager is ready, blocking the next start. Patching to `Ignore` is permanent: when the webhook times out, Kubernetes ignores the failure rather than blocking the request.

After patching, delete any stuck manager pods:

```bash
kubectl delete pod -n longhorn-system -l app=longhorn-manager
```

### ⚠️ Longhorn Manager Race Condition on First Start

**Symptom:** `longhorn-manager` pod crashes with:
```
level=fatal msg="Error starting manager: Operation cannot be fulfilled on
settings.longhorn.io \"default-replica-count\": the object has been modified;
please apply your changes to the latest version and try again"
```

**Cause:** Both manager pods (one per node) try to initialize the same Longhorn settings simultaneously. One loses the write conflict and exits fatally.

**Fix:** Simply delete the crashing pod. On the next restart the setting already exists and the conflict does not occur:

```bash
kubectl delete pod <crashing-manager-pod> -n longhorn-system
```

### Step 10: Verify Longhorn

```bash
kubectl get storageclass
# NAME                 PROVISIONER
# longhorn (default)   driver.longhorn.io

kubectl get pods -n longhorn-system
# All pods should be Running
```

### Step 11: Test PVC Provisioning

```yaml
# test-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 1Gi
```

```bash
kubectl apply -f test-pvc.yaml
kubectl get pvc
# STATUS: Bound — Longhorn provisioned the volume
kubectl delete -f test-pvc.yaml
```

---

## Phase 6 — Deploy PostgreSQL Manually (Week 3)

Deploy PostgreSQL manually before packaging with Helm. This builds understanding of every component before abstracting it.

### Step 12: Create Namespace and Secret

```bash
kubectl create namespace postgres

kubectl create secret generic postgres-secret \
  --from-literal=POSTGRES_USER=admin \
  --from-literal=POSTGRES_PASSWORD=securepassword123 \
  --from-literal=POSTGRES_DB=appdb \
  -n postgres
```

### Step 13: Create PVC

```yaml
# postgres-pvc.yaml
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
kubectl apply -f postgres-pvc.yaml
kubectl get pvc -n postgres
# postgres-pvc   Bound
```

### Step 14: Deploy PostgreSQL

```yaml
# postgres-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        envFrom:
        - secretRef:
            name: postgres-secret
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
          subPath: pgdata                        # ← Critical: see note below
      volumes:
      - name: postgres-data
        persistentVolumeClaim:
          claimName: postgres-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: postgres
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
```

> **Why `subPath: pgdata`?** When Longhorn provisions a volume, it formats it with ext4. The ext4 filesystem automatically creates a `lost+found` directory at the root of the volume. PostgreSQL's `initdb` checks that its data directory is empty before initializing — it refuses to start if it finds any existing files or directories, including `lost+found`. The `subPath: pgdata` tells Kubernetes to mount the volume at `<volume_root>/pgdata` instead of `<volume_root>/` — a subdirectory that does not exist yet and is created clean, with no `lost+found` inside it. This issue affects any database that checks for an empty data directory on init (MySQL, MariaDB, etc.) — `subPath` is the standard fix.

```bash
kubectl apply -f postgres-deployment.yaml
kubectl get pods -n postgres
# postgres-xxxx   1/1   Running
```

### Step 15: Simulate Node Failure

This is what Mittwald will ask about.

```bash
# Write data
kubectl exec -it -n postgres \
  $(kubectl get pod -n postgres -l app=postgres -o jsonpath='{.items[0].metadata.name}') \
  -- psql -U admin -d appdb \
  -c "CREATE TABLE test (id SERIAL, name TEXT); INSERT INTO test (name) VALUES ('data survived');"

# Cordon and drain the worker node
kubectl cordon k3s-worker
kubectl drain k3s-worker --ignore-daemonsets --delete-emptydir-data --force
```

> **Note on PodDisruptionBudgets:** Longhorn sets a PDB on its `instance-manager` pod to prevent eviction. The drain command will retry and eventually time out on this pod. Fix:
> ```bash
> kubectl delete poddisruptionbudget -n longhorn-system --all
> kubectl drain k3s-worker --ignore-daemonsets --delete-emptydir-data --force
> ```

```bash
# Watch postgres reschedule on master
kubectl get pods -n postgres -o wide -w

# Verify data survived
kubectl exec -it -n postgres \
  $(kubectl get pod -n postgres -l app=postgres -o jsonpath='{.items[0].metadata.name}') \
  -- psql -U admin -d appdb -c "SELECT * FROM test;"
# Returns: 1 | data survived

# Restore the worker
kubectl uncordon k3s-worker
```

> **Important note on replica count:** With `defaultReplicaCount=1`, the Longhorn volume has only one replica — on whichever node it was first provisioned. If you drain that node, the pod will stay `Pending` until the node comes back, because the data physically exists only there. With `replicaCount=2`, one replica lives on each node and the pod reschedules immediately. This is a key interview talking point — it illustrates exactly why replica count matters in production.

---

## Phase 7 — Write a Helm Chart from Scratch (Week 4)

Package everything into a reusable Helm chart. Do NOT copy an existing chart — write every file yourself.

### Step 16: Create Chart Skeleton

```bash
kubectl delete namespace postgres

mkdir -p postgres-chart/templates
cd postgres-chart
```

### `Chart.yaml`

```yaml
apiVersion: v2
name: postgres-chart
description: A Helm chart for deploying PostgreSQL with Longhorn persistent storage on bare metal Kubernetes
type: application
version: 0.1.0
appVersion: "15.0"
```

### `values.yaml`

```yaml
replicaCount: 1

image:
  repository: postgres
  tag: "15"
  pullPolicy: IfNotPresent

postgres:
  user: admin
  password: securepassword123
  database: appdb

storage:
  storageClass: longhorn
  size: 2Gi

service:
  type: ClusterIP
  port: 5432

resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"

namespace: postgres
```

### `templates/_helpers.tpl`

This is the file most candidates skip. Write it yourself — it defines reusable named templates that every other template file references.

```
{{/*
Expand the name of the chart.
*/}}
{{- define "postgres-chart.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "postgres-chart.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "postgres-chart.labels" -}}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
app.kubernetes.io/name: {{ include "postgres-chart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "postgres-chart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "postgres-chart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

### `templates/secret.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "postgres-chart.fullname" . }}-secret
  namespace: {{ .Values.namespace }}
  labels:
    {{- include "postgres-chart.labels" . | nindent 4 }}
type: Opaque
stringData:
  POSTGRES_USER: {{ .Values.postgres.user | quote }}
  POSTGRES_PASSWORD: {{ .Values.postgres.password | quote }}
  POSTGRES_DB: {{ .Values.postgres.database | quote }}
```

### `templates/pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: {{ include "postgres-chart.fullname" . }}-pvc
  namespace: {{ .Values.namespace }}
  labels:
    {{- include "postgres-chart.labels" . | nindent 4 }}
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: {{ .Values.storage.storageClass }}
  resources:
    requests:
      storage: {{ .Values.storage.size }}
```

### `templates/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "postgres-chart.fullname" . }}
  namespace: {{ .Values.namespace }}
  labels:
    {{- include "postgres-chart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "postgres-chart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "postgres-chart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: postgres
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        envFrom:
        - secretRef:
            name: {{ include "postgres-chart.fullname" . }}-secret
        ports:
        - name: postgres
          containerPort: 5432
          protocol: TCP
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
          subPath: pgdata                        # ← Required: prevents lost+found conflict
        livenessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - {{ .Values.postgres.user }}
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - {{ .Values.postgres.user }}
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: postgres-data
        persistentVolumeClaim:
          claimName: {{ include "postgres-chart.fullname" . }}-pvc
```

### `templates/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "postgres-chart.fullname" . }}
  namespace: {{ .Values.namespace }}
  labels:
    {{- include "postgres-chart.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  selector:
    {{- include "postgres-chart.selectorLabels" . | nindent 4 }}
  ports:
  - port: {{ .Values.service.port }}
    targetPort: postgres
    protocol: TCP
```

### Step 17: Validate and Deploy

```bash
# Lint — catch errors before deploying
helm lint postgres-chart/
# Expected: 1 chart(s) linted, 0 chart(s) failed
# The [INFO] icon warning is cosmetic — ignore it

# Dry run — see rendered output without applying
helm install postgres-release postgres-chart/ \
  --namespace postgres \
  --create-namespace \
  --dry-run

# Deploy
helm install postgres-release postgres-chart/ \
  --namespace postgres \
  --create-namespace

kubectl get all -n postgres
kubectl get pvc -n postgres
```

### Step 18: Practice Upgrades and Rollbacks

```bash
# Upgrade with inline override
helm upgrade postgres-release postgres-chart/ \
  --namespace postgres \
  --set storage.size=5Gi

# Upgrade with a custom values file
helm upgrade postgres-release postgres-chart/ \
  --namespace postgres \
  --values custom-values.yaml

# Check release history
helm history postgres-release -n postgres

# Rollback to revision 1
helm rollback postgres-release 1 -n postgres

# Uninstall
helm uninstall postgres-release -n postgres
```

---

## Phase 8 — GitHub Repository Structure (Week 4)

```
bare-metal-k8s-homelab/
├── README.md
├── docs/
│   ├── architecture.md
│   ├── longhorn-setup.md
│   └── troubleshooting.md        # Document everything below
├── infra/
│   ├── metallb-config.yaml
│   ├── test-pvc.yaml
│   └── postgres-manual/
│       ├── postgres-pvc.yaml
│       └── postgres-deployment.yaml
└── helm/
    └── postgres-chart/
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
            ├── _helpers.tpl
            ├── secret.yaml
            ├── pvc.yaml
            ├── deployment.yaml
            └── service.yaml
```

## Troubleshooting Reference

This section documents every real issue encountered during the build. Each one is an interview talking point.

### Issue 1: AWS Security Group Blocking Inter-Node Traffic

**Symptom:** Pods on the worker node crash with DNS timeout errors. Pods on the master node work fine. Pattern: exactly one pod per deployment is healthy (on master), the other crashes (on worker).

**Root cause:** AWS security group had no rule allowing traffic between the two instances on internal ports. Flannel's VXLAN tunnel (UDP 8472) was silently dropped.

**Diagnostic:**
```bash
# From the worker node — if this times out, networking is broken
curl --max-time 5 http://<kube-dns-clusterip>:53
# Correct result: "Empty reply from server" (connection succeeded, port doesn't speak HTTP)
# Broken result: "Connection timed out"
```

**Fix:** Add inbound + outbound rule: `All traffic` from/to `172.31.0.0/16` (VPC CIDR).

**Lesson:** Always verify inter-node connectivity first when one node's pods work and the other's don't. Test ClusterIP reachability from the worker before debugging Kubernetes internals.

---

### Issue 2: Admission Webhook Timeouts (MetalLB and Longhorn)

**Symptom:** After installing MetalLB or Longhorn, pods enter CrashLoopBackOff with logs showing webhook endpoint timeouts.

**Root cause:** k3s uses Flannel CNI. On AWS EC2, the k3s API server cannot reach ClusterIP-backed webhook services over the Flannel VXLAN tunnel. Webhook calls time out, causing fatal errors in the manager pods.

**This affects any component that installs admission webhooks**, including MetalLB and Longhorn. The pattern repeats every time.

**Fix for MetalLB:** Delete the webhook configurations (MetalLB does not recreate them aggressively):
```bash
kubectl delete validatingwebhookconfigurations metallb-webhook-configuration 2>/dev/null || true
kubectl delete mutatingwebhookconfigurations metallb-webhook-configuration 2>/dev/null || true
```

**Fix for Longhorn:** Patch to `Ignore` failure policy (Longhorn recreates webhooks on every manager restart, so deletion starts a loop):
```bash
kubectl patch validatingwebhookconfiguration longhorn-webhook-validator \
  --type='json' \
  -p='[{"op":"replace","path":"/webhooks/0/failurePolicy","value":"Ignore"}]'

kubectl patch mutatingwebhookconfiguration longhorn-webhook-mutator \
  --type='json' \
  -p='[{"op":"replace","path":"/webhooks/0/failurePolicy","value":"Ignore"}]'
```

**Lesson:** `failurePolicy: Fail` (the default) means a webhook timeout blocks the entire operation. `failurePolicy: Ignore` means the operation proceeds even if the webhook is unreachable. In a homelab without reliable webhook connectivity, `Ignore` is appropriate.

---

### Issue 3: PostgreSQL Refuses to Start — `lost+found` in Data Directory

**Symptom:**
```
initdb: error: directory "/var/lib/postgresql/data" exists but is not empty
initdb: detail: It contains a lost+found directory, perhaps due to it being a mount point.
```

**Root cause:** When Longhorn provisions a volume, it formats it with ext4. The ext4 filesystem automatically creates a `lost+found` directory at the volume root. PostgreSQL's `initdb` requires a completely empty directory and refuses to initialize if any files or directories exist.

**Fix:** Add `subPath: pgdata` to the volume mount:
```yaml
volumeMounts:
- name: postgres-data
  mountPath: /var/lib/postgresql/data
  subPath: pgdata
```

This mounts `<volume>/pgdata` instead of `<volume>/` — a subdirectory that is created fresh with no `lost+found`.

**Lesson:** This affects any database that validates an empty data directory on init. `subPath` is the standard Kubernetes solution. Applies to MySQL, MariaDB, and others.

---

### Issue 4: Longhorn Manager Race Condition on Startup

**Symptom:** One `longhorn-manager` pod crashes with:
```
Error starting manager: Operation cannot be fulfilled on settings.longhorn.io
"default-replica-count": the object has been modified; please apply your changes
to the latest version and try again
```

**Root cause:** Both manager pods (one per node) start simultaneously and race to initialize the same Longhorn settings. One loses the optimistic concurrency check and exits.

**Fix:** Delete the crashing pod. On next restart the setting already exists and the conflict does not occur:
```bash
kubectl delete pod <crashing-manager-pod> -n longhorn-system
```

---

### Issue 5: k3s-Agent CNI Not Routing ClusterIP Traffic

**Symptom:** After fixing the security group, ClusterIP routing still fails intermittently from the worker node. Restarting pods doesn't help.

**Fix:** Restart the k3s-agent on the worker node to reinitialize the Flannel CNI interface:
```bash
# On the WORKER node
sudo systemctl restart k3s-agent
```

---

### Issue 6: Longhorn PodDisruptionBudget Blocks Node Drain

**Symptom:** `kubectl drain k3s-worker` hangs indefinitely with:
```
error when evicting pods/"instance-manager-xxx": Cannot evict pod as it would
violate the pod's disruption budget.
```

**Root cause:** Longhorn sets a PDB on `instance-manager` to prevent eviction, since evicting it would interrupt running volume operations.

**Fix:**
```bash
kubectl delete poddisruptionbudget -n longhorn-system --all
kubectl drain k3s-worker --ignore-daemonsets --delete-emptydir-data --force
```
