# Talos HA Kubernetes Cluster and Canine and OpenCost – Production Ready

Complete production guide for building a high-availability Talos Linux Kubernetes cluster with Longhorn distributed storage, MetalLB load balancing, Canine PaaS with strong tenant isolation, and OpenCost cost monitoring — API-first, copy-paste ready.

---

## Cluster Specifications

| Component | Version / Value |
|-----------|----------------|
| Network | `192.168.29.0/24` |
| Talos Linux | v1.11.6 with VMware Tools |
| Kubernetes | v1.30.8 |
| Control Planes | 2 nodes with VIP failover |
| Worker Nodes | 3 nodes with dedicated storage disks |
| Calico CNI | v3.30.5 |
| MetalLB | v0.14.9 |
| Longhorn | v1.10.1 |
| Metrics Server | latest stable |
| Canine PaaS | latest (Apache 2.0) |
| OpenCost | latest stable (CNCF) |

---

## Table of Contents

1. [Cluster Topology](#cluster-topology)
2. [Architecture Overview](#architecture-overview)
3. [Prerequisites](#prerequisites)
4. [Installation Steps](#installation-steps)
5. [Network Components](#network-components)
6. [Storage Configuration](#storage-configuration)
7. [Canine PaaS – Isolated Multi-Tenant](#canine-paas--isolated-multi-tenant)
8. [OpenCost – Cost Monitoring](#opencost--cost-monitoring)
9. [Admin Access Control – IP Allowlist](#admin-access-control--ip-allowlist)
10. [Verification & Testing](#verification--testing)
11. [Operations & Maintenance](#operations--maintenance)
12. [Optional Hardening & Improvements](#optional-hardening--improvements)
13. [Troubleshooting](#troubleshooting)

---

## Cluster Topology

### Network Configuration

| Item | Value |
|------|-------|
| Network | 192.168.29.0/24 |
| Gateway | 192.168.29.1 |
| VIP | 192.168.29.10 |
| VIP Hostname | 29.talos.vaheed.net |

### Control Plane Nodes

| Name | IP | Hostname | Role |
|------|----|----------|------|
| cp01 | 192.168.29.11 | cp01 | Control Plane + etcd |
| cp02 | 192.168.29.12 | cp02 | Control Plane + etcd |

### Worker Nodes

| Name | IP | Hostname | Storage Device |
|------|----|----------|----------------|
| w01 | 192.168.29.21 | w01 | /dev/sdb |
| w02 | 192.168.29.22 | w02 | /dev/sdb |
| w03 | 192.168.29.23 | w03 | /dev/sdb |

### Service Ranges

| Service | Range / Address |
|---------|----------------|
| Kubernetes VIP | 192.168.29.10 |
| MetalLB Pool | 192.168.29.100 – 192.168.29.199 |
| Longhorn UI | 192.168.29.100 |
| Canine PaaS | 192.168.29.101 |
| OpenCost UI / API | 192.168.29.102 |
| Pod Network | 10.244.0.0/16 |
| Service Network | 10.96.0.0/12 |

### Admin Source IPs (Replace with your actual IPs)

| Purpose | IP / CIDR |
|---------|-----------|
| Admin workstation | 192.168.1.10/32 |
| VPN / jump host | 10.10.0.0/24 |
| CI/CD server | 192.168.1.20/32 |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                   Gateway (192.168.29.1)                    │
└───────────────────────────────┬─────────────────────────────┘
                                │
┌───────────────────────────────┴─────────────────────────────┐
│         VIP: 192.168.29.10 (29.talos.vaheed.net)           │
│         HA Kubernetes API — Admin IPs only                   │
└──────────────────┬────────────────────────┬─────────────────┘
                   │                        │
     ┌─────────────┴──────┐    ┌────────────┴───────────┐
     │  cp01 (.11)        │    │  cp02 (.12)            │
     │  hostname: cp01    │◄──►│  hostname: cp02        │
     │  Control Plane     │    │  Control Plane         │
     │  + etcd            │    │  + etcd                │
     └─────────────┬──────┘    └────────────┬───────────┘
                   └──────────┬─────────────┘
          ┌───────────────────┼───────────────────┐
          │                   │                   │
┌─────────┴────────┐ ┌────────┴─────────┐ ┌──────┴──────────┐
│  w01 (.21)       │ │  w02 (.22)       │ │  w03 (.23)      │
│  hostname: w01   │ │  hostname: w02   │ │  hostname: w03  │
│  Worker          │ │  Worker          │ │  Worker         │
│  /dev/sdb 100GB  │ │  /dev/sdb 100GB  │ │  /dev/sdb 100GB │
└──────────────────┘ └──────────────────┘ └─────────────────┘

Access control layers:
  ┌─────────────────────────────────────────────────────────┐
  │  Admin plane (.10, .101 /admin, .102)                   │
  │  → Calico GlobalNetworkPolicy: allow admin IPs only     │
  └─────────────────────────────────────────────────────────┘
  ┌─────────────────────────────────────────────────────────┐
  │  Tenant plane (Canine user namespaces)                  │
  │  → Isolated: NetworkPolicy, ResourceQuota, LimitRange   │
  │  → NO access to admin services                         │
  └─────────────────────────────────────────────────────────┘
```

---

## Prerequisites

### Hardware Requirements

**Control Plane Nodes (each):** 2+ vCPU, 4 GB+ RAM, 50 GB+ OS disk, `ens192`

**Worker Nodes (each):** 4+ vCPU, 8 GB+ RAM, 50 GB+ OS disk (`/dev/sda`), 100 GB+ data disk (`/dev/sdb`), `ens192`

### Software Requirements

- VMware vSphere environment
- `talosctl` v1.11.6, `kubectl` v1.30.8+, `helm` v3.x
- DNS: `29.talos.vaheed.net` → `192.168.29.10`
- DNS: `canine.29.talos.vaheed.net` → `192.168.29.101`
- DNS: `opencost.29.talos.vaheed.net` → `192.168.29.102`

### Network Requirements

- Subnet `192.168.29.0/24`, Gateway `192.168.29.1`
- Open ports: `6443` (Kubernetes API), `50000–51000` (Talos API), `80/443` (ingress)

---

## Installation Steps

### Step 1: Download Talos with VMware Tools

```bash
wget https://factory.talos.dev/image/dfd1ac9abdf529ca644694b17af0ce1a2ae23a5cccdff39439aa7f0774901e90/v1.11.6/vmware-amd64.iso \
  -O talos-vmware-1.11.6.iso
```

### Step 2: Install talosctl

```bash
# macOS (Apple Silicon)
wget https://github.com/siderolabs/talos/releases/download/v1.11.6/talosctl-darwin-arm64
chmod +x talosctl-darwin-arm64 && sudo mv talosctl-darwin-arm64 /usr/local/bin/talosctl

# macOS (Intel)
wget https://github.com/siderolabs/talos/releases/download/v1.11.6/talosctl-darwin-amd64
chmod +x talosctl-darwin-amd64 && sudo mv talosctl-darwin-amd64 /usr/local/bin/talosctl

# Linux
wget https://github.com/siderolabs/talos/releases/download/v1.11.6/talosctl-linux-amd64
chmod +x talosctl-linux-amd64 && sudo mv talosctl-linux-amd64 /usr/local/bin/talosctl

talosctl version --client
```

### Step 3: Create VMs in vSphere

1. Upload `talos-vmware-1.11.6.iso` to your datastore.
2. Create 5 VMs:
   - **cp01, cp02**: 2 vCPU, 4 GB RAM, 50 GB disk
   - **w01, w02, w03**: 4 vCPU, 8 GB RAM, 50 GB OS disk + 100 GB data disk (`/dev/sdb`)
3. Mount ISO and boot. Nodes start in maintenance mode automatically.

### Step 4: Generate Base Configuration

```bash
talosctl gen config "talos29" "https://29.talos.vaheed.net:6443" \
  --kubernetes-version v1.30.8 \
  --install-image factory.talos.dev/installer/dfd1ac9abdf529ca644694b17af0ce1a2ae23a5cccdff39439aa7f0774901e90:v1.11.6
```

Files created: `controlplane.yaml`, `worker.yaml`, `talosconfig`

### Step 5: Create Control Plane Patch

The `hostname` field under `machine.network` sets a **static, deterministic hostname** so nodes never get random cloud-init names. This is applied per-node via the IP patch in Step 7.

Create `cp-patch.yaml`:

```yaml
machine:
  network:
    interfaces:
      - interface: ens192
        vip:
          ip: 192.168.29.10
    extraHostEntries:
      - ip: 192.168.29.10
        aliases: ["29.talos.vaheed.net"]
      - ip: 192.168.29.11
        aliases: ["cp01"]
      - ip: 192.168.29.12
        aliases: ["cp02"]
      - ip: 192.168.29.21
        aliases: ["w01"]
      - ip: 192.168.29.22
        aliases: ["w02"]
      - ip: 192.168.29.23
        aliases: ["w03"]

  registries:
    mirrors:
      docker.io:
        endpoints: ["https://registry.vaheed.net:2053"]
      gcr.io:
        endpoints: ["https://registry.vaheed.net:2083"]
      ghcr.io:
        endpoints: ["https://registry.vaheed.net:2087"]
      quay.io:
        endpoints: ["https://registry.vaheed.net:8443"]
      registry.k8s.io:
        endpoints: ["https://registry.vaheed.net:2096"]

  certSANs:
    - 192.168.29.10
    - 29.talos.vaheed.net
    - 192.168.29.11
    - 192.168.29.12

  kubelet:
    nodeIP:
      validSubnets:
        - 192.168.29.0/24

cluster:
  network:
    cni:
      name: none
    dnsDomain: cluster.local
  allowSchedulingOnControlPlanes: false
```

### Step 6: Create Worker Patch

Create `worker-patch.yaml`:

```yaml
machine:
  network:
    extraHostEntries:
      - ip: 192.168.29.10
        aliases: ["29.talos.vaheed.net"]
      - ip: 192.168.29.11
        aliases: ["cp01"]
      - ip: 192.168.29.12
        aliases: ["cp02"]
      - ip: 192.168.29.21
        aliases: ["w01"]
      - ip: 192.168.29.22
        aliases: ["w02"]
      - ip: 192.168.29.23
        aliases: ["w03"]

  registries:
    mirrors:
      docker.io:
        endpoints: ["https://registry.vaheed.net:2053"]
      gcr.io:
        endpoints: ["https://registry.vaheed.net:2083"]
      ghcr.io:
        endpoints: ["https://registry.vaheed.net:2087"]
      quay.io:
        endpoints: ["https://registry.vaheed.net:8443"]
      registry.k8s.io:
        endpoints: ["https://registry.vaheed.net:2096"]

  kubelet:
    nodeIP:
      validSubnets:
        - 192.168.29.0/24
    extraMounts:
      - destination: /var/lib/longhorn
        type: bind
        source: /var/mnt/longhorn
        options:
          - bind
          - rshared
          - rw

  disks:
    - device: /dev/sdb
      partitions:
        - mountpoint: /var/mnt/longhorn

  sysctls:
    vm.max_map_count: "262144"

cluster:
  network:
    dnsDomain: cluster.local
```

### Step 7: Generate Per-Node Configurations

Each node patch sets **both** a static IP and a **static hostname** via `machine.network.hostname`. This prevents Talos from falling back to a random or DHCP-assigned name.

#### cp01 (192.168.29.11, hostname: cp01)

```bash
talosctl machineconfig patch controlplane.yaml --patch @cp-patch.yaml --output cp01.yaml

talosctl machineconfig patch cp01.yaml \
  --patch '[
    {"op":"replace","path":"/machine/network/interfaces","value":[{"interface":"ens192","dhcp":false,"addresses":["192.168.29.11/24"],"routes":[{"network":"0.0.0.0/0","gateway":"192.168.29.1"}],"vip":{"ip":"192.168.29.10"}}]},
    {"op":"add","path":"/machine/network/hostname","value":"cp01"}
  ]' \
  --output cp01.yaml
```

#### cp02 (192.168.29.12, hostname: cp02)

```bash
talosctl machineconfig patch controlplane.yaml --patch @cp-patch.yaml --output cp02.yaml

talosctl machineconfig patch cp02.yaml \
  --patch '[
    {"op":"replace","path":"/machine/network/interfaces","value":[{"interface":"ens192","dhcp":false,"addresses":["192.168.29.12/24"],"routes":[{"network":"0.0.0.0/0","gateway":"192.168.29.1"}],"vip":{"ip":"192.168.29.10"}}]},
    {"op":"add","path":"/machine/network/hostname","value":"cp02"}
  ]' \
  --output cp02.yaml
```

#### w01 (192.168.29.21, hostname: w01)

```bash
talosctl machineconfig patch worker.yaml --patch @worker-patch.yaml --output w01.yaml

talosctl machineconfig patch w01.yaml \
  --patch '[
    {"op":"replace","path":"/machine/network/interfaces","value":[{"interface":"ens192","dhcp":false,"addresses":["192.168.29.21/24"],"routes":[{"network":"0.0.0.0/0","gateway":"192.168.29.1"}]}]},
    {"op":"add","path":"/machine/network/hostname","value":"w01"}
  ]' \
  --output w01.yaml
```

#### w02 (192.168.29.22, hostname: w02)

```bash
talosctl machineconfig patch worker.yaml --patch @worker-patch.yaml --output w02.yaml

talosctl machineconfig patch w02.yaml \
  --patch '[
    {"op":"replace","path":"/machine/network/interfaces","value":[{"interface":"ens192","dhcp":false,"addresses":["192.168.29.22/24"],"routes":[{"network":"0.0.0.0/0","gateway":"192.168.29.1"}]}]},
    {"op":"add","path":"/machine/network/hostname","value":"w02"}
  ]' \
  --output w02.yaml
```

#### w03 (192.168.29.23, hostname: w03)

```bash
talosctl machineconfig patch worker.yaml --patch @worker-patch.yaml --output w03.yaml

talosctl machineconfig patch w03.yaml \
  --patch '[
    {"op":"replace","path":"/machine/network/interfaces","value":[{"interface":"ens192","dhcp":false,"addresses":["192.168.29.23/24"],"routes":[{"network":"0.0.0.0/0","gateway":"192.168.29.1"}]}]},
    {"op":"add","path":"/machine/network/hostname","value":"w03"}
  ]' \
  --output w03.yaml
```

### Step 8: Apply Configurations

```bash
talosctl apply-config --insecure --nodes 192.168.29.11 --file cp01.yaml
talosctl apply-config --insecure --nodes 192.168.29.12 --file cp02.yaml
talosctl apply-config --insecure --nodes 192.168.29.21 --file w01.yaml
talosctl apply-config --insecure --nodes 192.168.29.22 --file w02.yaml
talosctl apply-config --insecure --nodes 192.168.29.23 --file w03.yaml

echo "Waiting 10 minutes for nodes to initialize..."
sleep 600
```

### Step 9: Bootstrap Cluster

```bash
# Bootstrap on first control plane only — run exactly once
talosctl --talosconfig talosconfig bootstrap \
  --endpoints 192.168.29.11 \
  --nodes 192.168.29.11

echo "Waiting 5 minutes for cluster initialization..."
sleep 300

ping -c 4 192.168.29.10

talosctl --talosconfig talosconfig kubeconfig . \
  --nodes 192.168.29.11 \
  --endpoints 192.168.29.11 \
  --force

kubectl --kubeconfig=kubeconfig get nodes
# Expected: cp01, cp02, w01, w02, w03 — all with correct static hostnames
```

### Step 10: Verify Static Hostnames

```bash
kubectl --kubeconfig=kubeconfig get nodes -o custom-columns=NAME:.metadata.name,IP:.status.addresses[0].address

# Verify via talosctl
for node in 192.168.29.11 192.168.29.12 192.168.29.21 192.168.29.22 192.168.29.23; do
  echo -n "$node → hostname: "
  talosctl --talosconfig talosconfig --nodes $node get hostname 2>/dev/null | awk 'NR==2{print $2}'
done

# Expected output:
# 192.168.29.11 → hostname: cp01
# 192.168.29.12 → hostname: cp02
# 192.168.29.21 → hostname: w01
# 192.168.29.22 → hostname: w02
# 192.168.29.23 → hostname: w03
```

---

## Network Components

### Install Calico CNI

```bash
kubectl --kubeconfig=kubeconfig apply -f \
  https://raw.githubusercontent.com/projectcalico/calico/v3.30.5/manifests/tigera-operator.yaml

kubectl --kubeconfig=kubeconfig apply -f \
  https://raw.githubusercontent.com/projectcalico/calico/v3.30.5/manifests/custom-resources.yaml

kubectl --kubeconfig=kubeconfig wait --for=condition=ready pod \
  -l k8s-app=calico-node -n calico-system --timeout=300s

kubectl --kubeconfig=kubeconfig get nodes
```

### Install MetalLB

```bash
kubectl --kubeconfig=kubeconfig apply -f \
  https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/metallb-native.yaml

kubectl --kubeconfig=kubeconfig wait --for=condition=ready pod \
  -l app=metallb -n metallb-system --timeout=300s

kubectl --kubeconfig=kubeconfig apply -f - <<'EOF'
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.29.100-192.168.29.199
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-advert
  namespace: metallb-system
spec:
  ipAddressPools:
    - default-pool
EOF
```

### Install Metrics Server

```bash
kubectl --kubeconfig=kubeconfig apply -f \
  https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

kubectl --kubeconfig=kubeconfig patch deployment metrics-server -n kube-system --type='json' \
  -p='[
    {"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-preferred-address-types=InternalIP"},
    {"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}
  ]'

kubectl --kubeconfig=kubeconfig wait --for=condition=ready pod \
  -l k8s-app=metrics-server -n kube-system --timeout=300s

sleep 30 && kubectl --kubeconfig=kubeconfig top nodes
```

---

## Storage Configuration

### Prepare Worker Disks for Longhorn

Create `longhorn-disk.yaml`:

```yaml
machine:
  disks:
    - device: /dev/sdb
      partitions:
        - mountpoint: /var/lib/longhorn
  kubelet:
    extraMounts:
      - destination: /var/lib/longhorn
        type: bind
        source: /var/lib/longhorn
        options:
          - bind
          - rshared
          - rw
```

```bash
for node in 192.168.29.21 192.168.29.22 192.168.29.23; do
  echo "Patching disk config on $node..."
  talosctl --talosconfig talosconfig patch machineconfig \
    -n $node -e $node --patch @longhorn-disk.yaml
done
```

### Install Longhorn

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update

kubectl --kubeconfig=kubeconfig create namespace longhorn-system

kubectl --kubeconfig=kubeconfig label namespace longhorn-system \
  pod-security.kubernetes.io/enforce=privileged --overwrite

helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --kubeconfig=kubeconfig \
  --version 1.10.1 \
  --set defaultSettings.defaultDataPath="/var/lib/longhorn" \
  --set defaultSettings.replicaCount=3 \
  --set persistence.defaultClass=true \
  --set persistence.defaultFsType=xfs \
  --set persistence.defaultClassReplicaCount=3

echo "Waiting for Longhorn (up to 10 minutes)..."
kubectl --kubeconfig=kubeconfig wait --for=condition=ready pod \
  -l app=longhorn-manager -n longhorn-system --timeout=600s

kubectl --kubeconfig=kubeconfig get pods -n longhorn-system
kubectl --kubeconfig=kubeconfig get storageclass
```

### Expose Longhorn UI (Admin only — see IP Allowlist section)

```bash
kubectl --kubeconfig=kubeconfig apply -f - <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: longhorn-frontend-lb
  namespace: longhorn-system
spec:
  type: LoadBalancer
  loadBalancerIP: 192.168.29.100
  selector:
    app: longhorn-ui
  ports:
    - name: http
      port: 80
      targetPort: 8000
      protocol: TCP
EOF

sleep 15
kubectl --kubeconfig=kubeconfig get svc -n longhorn-system longhorn-frontend-lb
echo "Longhorn UI: http://192.168.29.100 (admin IPs only — see GlobalNetworkPolicy)"
```

### Verify Longhorn Storage

```bash
kubectl --kubeconfig=kubeconfig apply -f - <<'EOF'
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
      storage: 5Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
    - name: test
      image: nginx
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: test-pvc
EOF

sleep 60
kubectl --kubeconfig=kubeconfig exec test-pod -- sh -c "echo 'Longhorn OK' > /data/test.txt"
kubectl --kubeconfig=kubeconfig exec test-pod -- cat /data/test.txt

kubectl --kubeconfig=kubeconfig delete pod test-pod
kubectl --kubeconfig=kubeconfig delete pvc test-pvc
```

---

## Canine PaaS – Isolated Multi-Tenant

> **Canine** is a developer-friendly, self-hosted PaaS for Kubernetes. Git-push deploys, in-cluster Docker builds via BuildKit, service management, and environment secrets — without writing YAML. Apache 2.0 licensed.
>
> **Project**: https://github.com/CanineHQ/canine | **Docs**: https://docs.canine.sh

### Tenant Isolation Model

Every account/team in Canine maps to a dedicated Kubernetes **namespace**. Isolation is enforced at the Kubernetes layer — not just the application layer — via four independently effective controls:

| Layer | Mechanism | What it prevents |
|-------|-----------|-----------------|
| Network | Calico `NetworkPolicy` (deny-all default) | Cross-namespace pod traffic |
| Compute | `ResourceQuota` | Runaway CPU/memory/storage usage |
| Defaults | `LimitRange` | Pods starting without resource constraints |
| Auth | `Role` + `RoleBinding` (namespace-scoped) | Tenant access to other namespaces or cluster API |

Admin control surfaces (Canine dashboard, OpenCost API, Longhorn UI, Kubernetes API) are **completely separated** from tenant namespaces via Calico `GlobalNetworkPolicy` — tenant pods cannot reach admin services regardless of NetworkPolicy within the tenant namespace.

### Step 1: Create Canine Namespace

```bash
kubectl --kubeconfig=kubeconfig create namespace canine

kubectl --kubeconfig=kubeconfig label namespace canine \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/audit=privileged \
  pod-security.kubernetes.io/warn=privileged \
  --overwrite
```

### Step 2: Add Canine Helm Repository

```bash
helm repo add canine https://canine.sh/charts
helm repo update
helm search repo canine
```

### Step 3: Generate Secret Key

```bash
# Copy the output and paste into secretKeyBase in canine-values.yaml
openssl rand -hex 64
```

### Step 4: Create Canine Values File

Create `canine-values.yaml`:

```yaml
global:
  domain: canine.29.talos.vaheed.net

service:
  type: LoadBalancer
  loadBalancerIP: "192.168.29.101"
  port: 80

ingress:
  enabled: false   # Using MetalLB LoadBalancer directly

persistence:
  enabled: true
  storageClass: longhorn
  accessMode: ReadWriteOnce
  size: 20Gi

database:
  internal: true
  persistence:
    enabled: true
    storageClass: longhorn
    size: 10Gi

buildkit:
  enabled: true
  persistence:
    enabled: true
    storageClass: longhorn
    size: 50Gi

resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "2"
    memory: "2Gi"

# REQUIRED: replace with output of: openssl rand -hex 64
secretKeyBase: "REPLACE_WITH_OUTPUT_OF_openssl_rand_-hex_64"

adminEmail: "admin@vaheed.net"
# REQUIRED: set a strong password (min 20 chars, mixed case + symbols)
adminPassword: "REPLACE_WITH_STRONG_PASSWORD"

# Per-tenant namespace enforcement
tenantIsolation:
  networkPolicy:
    enabled: true
  resourceQuota:
    enabled: true
    defaultCPU: "4"
    defaultMemory: "8Gi"
    defaultPVCCount: "10"
    defaultStorage: "50Gi"
  limitRange:
    enabled: true
    defaultCPURequest: "100m"
    defaultCPULimit: "1"
    defaultMemoryRequest: "128Mi"
    defaultMemoryLimit: "512Mi"
```

### Step 5: Install Canine

```bash
helm install canine canine/canine \
  --namespace canine \
  --kubeconfig=kubeconfig \
  -f canine-values.yaml \
  --wait \
  --timeout=15m

kubectl --kubeconfig=kubeconfig wait --for=condition=ready pod \
  -l app.kubernetes.io/name=canine -n canine --timeout=600s

kubectl --kubeconfig=kubeconfig get pods -n canine
kubectl --kubeconfig=kubeconfig get svc -n canine
echo "Canine Dashboard: http://canine.29.talos.vaheed.net (admin IPs only)"
```

### Step 6: Tenant Namespace Isolation Template

Apply this for every Canine user namespace. Canine creates namespaces automatically; run this script after each new user/team is provisioned to harden the namespace.

Create `apply-tenant-isolation.sh`:

```bash
#!/usr/bin/env bash
# Usage: NS=canine-alice ./apply-tenant-isolation.sh
set -euo pipefail

NS="${NS:?Set NS to the tenant namespace, e.g. NS=canine-alice}"

echo "Applying isolation policies to namespace: ${NS}"

kubectl --kubeconfig=kubeconfig apply -f - <<EOF
# ── NetworkPolicy: deny all by default; allow only intra-namespace + DNS ──
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-default
  namespace: ${NS}
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector: {}
  egress:
    # Same-namespace pods only
    - to:
        - podSelector: {}
    # kube-dns
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
        - podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
---
# ── Explicit deny: block tenant pods from reaching admin namespaces ──
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-admin-namespaces
  namespace: ${NS}
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector: {}
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
        - podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
    # Explicitly: NOT allowed to canine, opencost, longhorn-system, prometheus-system
---
# ── ResourceQuota: hard cap on compute and storage ──
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-quota
  namespace: ${NS}
spec:
  hard:
    requests.cpu: "2"
    requests.memory: "4Gi"
    limits.cpu: "4"
    limits.memory: "8Gi"
    persistentvolumeclaims: "10"
    requests.storage: "50Gi"
    longhorn.storageclass.storage.k8s.io/requests.storage: "50Gi"
    count/secrets: "30"
    count/configmaps: "30"
    count/services: "20"
    count/pods: "50"
---
# ── LimitRange: auto-inject defaults so all pods are constrained ──
apiVersion: v1
kind: LimitRange
metadata:
  name: tenant-limits
  namespace: ${NS}
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "2"
        memory: "4Gi"
      min:
        cpu: "50m"
        memory: "64Mi"
    - type: PersistentVolumeClaim
      max:
        storage: "20Gi"
      min:
        storage: "1Gi"
---
# ── RBAC: tenant service account scoped to its own namespace only ──
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tenant-sa
  namespace: ${NS}
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: tenant-role
  namespace: ${NS}
rules:
  - apiGroups: [""]
    resources: ["pods","services","configmaps","secrets","persistentvolumeclaims","pods/log","pods/exec"]
    verbs: ["get","list","watch","create","update","patch","delete"]
  - apiGroups: ["apps"]
    resources: ["deployments","replicasets","statefulsets"]
    verbs: ["get","list","watch","create","update","patch","delete"]
  - apiGroups: ["batch"]
    resources: ["jobs","cronjobs"]
    verbs: ["get","list","watch","create","update","patch","delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: tenant-rolebinding
  namespace: ${NS}
subjects:
  - kind: ServiceAccount
    name: tenant-sa
    namespace: ${NS}
roleRef:
  kind: Role
  name: tenant-role
  apiGroup: rbac.authorization.k8s.io
EOF

echo "Done: ${NS} is isolated."
```

```bash
chmod +x apply-tenant-isolation.sh
```

### Step 7: Canine API – Complete Operations Reference

All Canine management is API-first. Obtain your admin token from Settings → API Keys in the dashboard, or via the bootstrap secret.

```bash
CANINE_URL="http://canine.29.talos.vaheed.net"
CANINE_TOKEN="your-admin-api-token-here"

# ── Health check ──
curl -s $CANINE_URL/api/v1/health | jq .

# ── List all users (tenants) ──
curl -s -H "Authorization: Bearer $CANINE_TOKEN" \
  $CANINE_URL/api/v1/users | jq .

# ── Create a new user (tenant) ──
curl -s -X POST \
  -H "Authorization: Bearer $CANINE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@example.com","password":"SecurePass!123","name":"Alice"}' \
  $CANINE_URL/api/v1/users | jq .

# After creating a user, apply isolation to their namespace:
# NS=canine-alice ./apply-tenant-isolation.sh

# ── Get a specific user ──
curl -s -H "Authorization: Bearer $CANINE_TOKEN" \
  $CANINE_URL/api/v1/users/2 | jq .

# ── Update user (e.g. reset password) ──
curl -s -X PATCH \
  -H "Authorization: Bearer $CANINE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"password":"NewSecurePass!456"}' \
  $CANINE_URL/api/v1/users/2 | jq .

# ── Suspend a user (disable login) ──
curl -s -X PATCH \
  -H "Authorization: Bearer $CANINE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"suspended":true}' \
  $CANINE_URL/api/v1/users/2 | jq .

# ── List all applications ──
curl -s -H "Authorization: Bearer $CANINE_TOKEN" \
  $CANINE_URL/api/v1/applications | jq .

# ── Create an application for a user ──
curl -s -X POST \
  -H "Authorization: Bearer $CANINE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "my-api",
    "repository": "https://github.com/alice/my-api",
    "branch": "main",
    "user_id": 2
  }' \
  $CANINE_URL/api/v1/applications | jq .

# ── Trigger a deployment ──
curl -s -X POST \
  -H "Authorization: Bearer $CANINE_TOKEN" \
  $CANINE_URL/api/v1/applications/1/deploy | jq .

# ── List deployments for an application ──
curl -s -H "Authorization: Bearer $CANINE_TOKEN" \
  $CANINE_URL/api/v1/applications/1/deployments | jq .

# ── Get deployment status ──
curl -s -H "Authorization: Bearer $CANINE_TOKEN" \
  $CANINE_URL/api/v1/applications/1/deployments/latest | jq '.status'

# ── Set environment variable (secret) ──
curl -s -X POST \
  -H "Authorization: Bearer $CANINE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"key":"DATABASE_URL","value":"postgres://user:pass@host/db"}' \
  $CANINE_URL/api/v1/applications/1/environment_variables | jq .

# ── List environment variables ──
curl -s -H "Authorization: Bearer $CANINE_TOKEN" \
  $CANINE_URL/api/v1/applications/1/environment_variables | jq .

# ── Delete an environment variable ──
curl -s -X DELETE \
  -H "Authorization: Bearer $CANINE_TOKEN" \
  $CANINE_URL/api/v1/applications/1/environment_variables/DATABASE_URL | jq .

# ── Scale an application (set replica count) ──
curl -s -X PATCH \
  -H "Authorization: Bearer $CANINE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"replicas": 3}' \
  $CANINE_URL/api/v1/applications/1 | jq .

# ── Add a custom domain ──
curl -s -X POST \
  -H "Authorization: Bearer $CANINE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"domain":"api.alice.com"}' \
  $CANINE_URL/api/v1/applications/1/domains | jq .

# ── Delete an application ──
curl -s -X DELETE \
  -H "Authorization: Bearer $CANINE_TOKEN" \
  $CANINE_URL/api/v1/applications/1 | jq .
```

### Step 8: Automate Tenant Provisioning (API + Isolation Script)

Create `provision-tenant.sh` — full end-to-end: create user via API, then apply Kubernetes isolation:

```bash
#!/usr/bin/env bash
# provision-tenant.sh — provision an isolated Canine tenant
# Usage: ./provision-tenant.sh alice alice@example.com "SecurePass!123"
set -euo pipefail

NAME="${1:?Usage: $0 <name> <email> <password>}"
EMAIL="${2:?}"
PASSWORD="${3:?}"

CANINE_URL="http://canine.29.talos.vaheed.net"
CANINE_TOKEN="${CANINE_TOKEN:?Set CANINE_TOKEN env var}"

echo "=== Provisioning tenant: $NAME ($EMAIL) ==="

# 1. Create user via Canine API
RESPONSE=$(curl -s -X POST \
  -H "Authorization: Bearer $CANINE_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"name\":\"$NAME\",\"email\":\"$EMAIL\",\"password\":\"$PASSWORD\"}" \
  $CANINE_URL/api/v1/users)

USER_ID=$(echo "$RESPONSE" | jq -r '.id')
NS=$(echo "$RESPONSE" | jq -r '.namespace // "canine-'"$(echo $NAME | tr '[:upper:]' '[:lower:]')"'"')

echo "User created: ID=$USER_ID, Namespace=$NS"

# 2. Wait for Canine to create the namespace
echo "Waiting for namespace $NS..."
for i in $(seq 1 30); do
  kubectl --kubeconfig=kubeconfig get namespace "$NS" &>/dev/null && break
  sleep 2
done

# 3. Apply Kubernetes isolation policies
NS="$NS" ./apply-tenant-isolation.sh

echo ""
echo "✅ Tenant provisioned:"
echo "   User ID:   $USER_ID"
echo "   Email:     $EMAIL"
echo "   Namespace: $NS"
echo "   Dashboard: $CANINE_URL"
```

```bash
chmod +x provision-tenant.sh
export CANINE_TOKEN="your-admin-api-token-here"
./provision-tenant.sh alice alice@example.com "SecurePass!123"
```

---

## OpenCost – Cost Monitoring

> **OpenCost** is a CNCF incubating project for real-time Kubernetes cost allocation. REST API and web UI for costs by namespace, pod, label, and deployment.
>
> **Project**: https://github.com/opencost/opencost | **Docs**: https://opencost.io/docs

### Step 1: Install Prometheus

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/prometheus \
  --namespace prometheus-system \
  --create-namespace \
  --kubeconfig=kubeconfig \
  --set prometheus-pushgateway.enabled=false \
  --set alertmanager.enabled=false \
  --set server.persistentVolume.enabled=true \
  --set server.persistentVolume.storageClass=longhorn \
  --set server.persistentVolume.size=20Gi \
  -f https://raw.githubusercontent.com/opencost/opencost/develop/kubernetes/prometheus/extraScrapeConfigs.yaml

kubectl --kubeconfig=kubeconfig wait --for=condition=ready pod \
  -l app.kubernetes.io/name=prometheus -n prometheus-system --timeout=300s

kubectl --kubeconfig=kubeconfig get pods -n prometheus-system
```

### Step 2: Create OpenCost Namespace

```bash
kubectl --kubeconfig=kubeconfig create namespace opencost

kubectl --kubeconfig=kubeconfig label namespace opencost \
  pod-security.kubernetes.io/enforce=privileged --overwrite
```

### Step 3: Create OpenCost Values File

Create `opencost-values.yaml`:

```yaml
opencost:
  exporter:
    customPricing:
      enabled: true
      costModel:
        description: "On-prem VMware cluster pricing"
        CPU: "0.030000"       # USD per vCPU-hour — adjust to your hardware cost
        RAM: "0.004000"       # USD per GiB-hour
        storage: "0.000050"   # USD per GiB-hour
        GPU: "0.000000"

    env:
      - name: PROMETHEUS_SERVER_ENDPOINT
        value: "http://prometheus-server.prometheus-system.svc.cluster.local"

    persistence:
      enabled: true
      storageClass: longhorn
      accessMode: ReadWriteOnce
      size: 5Gi

    resources:
      requests:
        cpu: "100m"
        memory: "256Mi"
      limits:
        cpu: "1"
        memory: "1Gi"

  ui:
    enabled: true
    resources:
      requests:
        cpu: "50m"
        memory: "64Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"

  mcp:
    enabled: true
    port: 8081
```

### Step 4: Install OpenCost

```bash
helm repo add opencost-charts https://opencost.github.io/opencost-helm-chart
helm repo update

helm install opencost opencost-charts/opencost \
  --namespace opencost \
  --kubeconfig=kubeconfig \
  -f opencost-values.yaml \
  --wait \
  --timeout=10m

kubectl --kubeconfig=kubeconfig wait --for=condition=ready pod \
  -l app.kubernetes.io/name=opencost -n opencost --timeout=300s

kubectl --kubeconfig=kubeconfig get pods -n opencost
```

### Step 5: Expose OpenCost (Admin only)

```bash
kubectl --kubeconfig=kubeconfig apply -f - <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: opencost-ui-lb
  namespace: opencost
spec:
  type: LoadBalancer
  loadBalancerIP: 192.168.29.102
  selector:
    app.kubernetes.io/name: opencost
    app.kubernetes.io/component: ui
  ports:
    - name: http-ui
      port: 80
      targetPort: 9090
      protocol: TCP
    - name: http-api
      port: 9003
      targetPort: 9003
      protocol: TCP
EOF

sleep 15
kubectl --kubeconfig=kubeconfig get svc -n opencost opencost-ui-lb
echo "OpenCost UI:  http://192.168.29.102 (admin IPs only)"
echo "OpenCost API: http://192.168.29.102:9003 (admin IPs only)"
```

### Step 6: OpenCost API – Complete Operations Reference

```bash
OPENCOST_API="http://192.168.29.102:9003"

# ── Health check ──
curl -s $OPENCOST_API/healthz

# ── Total allocation: last 7 days by namespace ──
curl -sG $OPENCOST_API/allocation \
  -d window=7d \
  -d aggregate=namespace \
  -d accumulate=true \
  -d resolution=1m | jq '.data[0] | to_entries[] | {namespace: .key, totalCost: .value.totalCost}'

# ── Hourly breakdown: last 24h by namespace ──
curl -sG $OPENCOST_API/allocation \
  -d window=24h \
  -d step=1h \
  -d aggregate=namespace \
  -d resolution=1m | jq '.data[] | to_entries[] | {namespace: .key, cost: .value.totalCost}'

# ── Per-tenant cost (replace canine-alice with actual tenant namespace) ──
curl -sG $OPENCOST_API/allocation \
  -d window=30d \
  -d aggregate=namespace \
  -d accumulate=true \
  -d 'filter=namespace:"canine-alice"' | jq '.data[0]'

# ── Deployment-level drill-down within a tenant namespace ──
curl -sG $OPENCOST_API/allocation \
  -d window=7d \
  -d aggregate=deployment \
  -d accumulate=true \
  -d 'filter=namespace:"canine-alice"' | \
  jq '.data[0] | to_entries[] | {deployment: .key, totalCost: .value.totalCost}'

# ── Asset costs (nodes, disks, load balancers) ──
curl -sG $OPENCOST_API/assets \
  -d window=7d \
  -d aggregate=type \
  -d accumulate=true | \
  jq '.data[0] | to_entries[] | {type: .key, totalCost: .value.totalCost}'

# ── Cluster-wide total: last 30 days ──
curl -sG $OPENCOST_API/allocation \
  -d window=30d \
  -d aggregate=cluster \
  -d accumulate=true | jq '.data[0]'

# ── CPU + memory efficiency per namespace ──
curl -sG $OPENCOST_API/allocation \
  -d window=7d \
  -d aggregate=namespace \
  -d accumulate=true | jq '.data[0] | to_entries[] | {
    namespace: .key,
    cpuEfficiency: .value.cpuEfficiency,
    ramEfficiency: .value.ramEfficiency,
    totalCost: .value.totalCost
  }'

# ── Current month cost per namespace (calendar month) ──
START=$(date +%Y-%m-01T00:00:00Z)
END=$(date -u +%Y-%m-%dT%H:%M:%SZ)
curl -sG $OPENCOST_API/allocation \
  -d "window=${START},${END}" \
  -d aggregate=namespace \
  -d accumulate=true | \
  jq '.data[0] | to_entries[] | {namespace: .key, monthToDateCost: .value.totalCost}'

# ── Custom pricing config ──
curl -s $OPENCOST_API/costModel/customPricing | jq .
```

### Step 7: Automated Tenant Cost Report

Create `tenant-cost-report.sh`:

```bash
#!/usr/bin/env bash
# tenant-cost-report.sh — per-tenant cost for the last 30 days
set -euo pipefail

OPENCOST_API="http://192.168.29.102:9003"
WINDOW="30d"

echo "=== Tenant Cost Report — last ${WINDOW} ==="
echo "Generated: $(date -u)"
echo ""

COSTS=$(curl -sG "${OPENCOST_API}/allocation" \
  -d "window=${WINDOW}" \
  -d "aggregate=namespace" \
  -d "accumulate=true" \
  -d "resolution=1m")

printf "%-42s %12s %12s %12s %10s %10s\n" \
  "Namespace" "CPU Cost" "RAM Cost" "Total Cost" "CPU Eff%" "RAM Eff%"
printf "%-42s %12s %12s %12s %10s %10s\n" \
  "──────────────────────────────────────────" "────────────" "────────────" "────────────" "──────────" "──────────"

echo "$COSTS" | jq -r '.data[0] | to_entries[] |
  [.key, .value.cpuCost, .value.ramCost, .value.totalCost,
   (.value.cpuEfficiency * 100 | floor),
   (.value.ramEfficiency * 100 | floor)] | @tsv' \
  | sort -t$'\t' -k4 -rn \
  | while IFS=$'\t' read -r ns cpu ram total cpueff rameff; do
      printf "%-42s %12.4f %12.4f %12.4f %9d%% %9d%%\n" \
        "$ns" "$cpu" "$ram" "$total" "$cpueff" "$rameff"
    done

echo ""
echo "All costs in USD"
```

```bash
chmod +x tenant-cost-report.sh
./tenant-cost-report.sh
```

---

## Admin Access Control – IP Allowlist

This section enforces that **only admin IPs** can reach the control plane services: Canine dashboard, OpenCost UI/API, Longhorn UI, and the Kubernetes API. Tenant users can reach only their own application endpoints — they have zero visibility or network path to any admin surface.

**Replace the sample CIDRs below with your real admin/VPN IPs before applying.**

### Admin IP Reference

```
Admin workstation:  192.168.1.10/32
VPN / jump host:    10.10.0.0/24
CI/CD server:       192.168.1.20/32
```

### Step 1: Calico GlobalNetworkPolicy – Protect Admin Services

`GlobalNetworkPolicy` applies cluster-wide and takes effect before any namespace-scoped `NetworkPolicy`. This is the correct Calico resource for cross-namespace admin access control.

```bash
kubectl --kubeconfig=kubeconfig apply -f - <<'EOF'
# ── Allow admin IPs to reach Canine, OpenCost, Longhorn, Prometheus ──
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: allow-admin-to-platform-services
spec:
  order: 100
  selector: "projectcalico.org/namespace in {'canine','opencost','longhorn-system','prometheus-system'}"
  ingress:
    # Admin workstation
    - action: Allow
      source:
        nets:
          - 192.168.1.10/32
          - 10.10.0.0/24
          - 192.168.1.20/32
  egress:
    - action: Allow
  types:
    - Ingress
    - Egress
---
# ── Default: deny all other ingress to platform namespaces ──
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: deny-non-admin-to-platform-services
spec:
  order: 200
  selector: "projectcalico.org/namespace in {'canine','opencost','longhorn-system','prometheus-system'}"
  ingress:
    - action: Deny
  types:
    - Ingress
---
# ── Prevent tenant pods from egressing to platform namespaces ──
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: deny-tenant-to-platform-egress
spec:
  order: 150
  # Applies to all namespaces that start with "canine-" (tenant namespaces)
  namespaceSelector: "name starts with 'canine-'"
  egress:
    # Block egress to admin namespaces
    - action: Deny
      destination:
        namespaceSelector: "projectcalico.org/name in {'canine','opencost','longhorn-system','prometheus-system','kube-system'}"
    # Allow everything else (internet, intra-namespace, DNS)
    - action: Allow
  types:
    - Egress
EOF
```

### Step 2: Kubernetes API Server – Admin IP Allowlist

The Kubernetes API (VIP `192.168.29.10:6443`) should only be reachable from admin IPs. Enforce this via a Calico `GlobalNetworkPolicy` on the host network.

```bash
kubectl --kubeconfig=kubeconfig apply -f - <<'EOF'
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: allow-admin-to-k8s-api
spec:
  order: 50
  selector: all()
  applyOnForward: true
  preDNAT: true
  ingress:
    # Kubernetes API — admin IPs only
    - action: Allow
      protocol: TCP
      destination:
        nets:
          - 192.168.29.10/32
          - 192.168.29.11/32
          - 192.168.29.12/32
        ports: [6443]
      source:
        nets:
          - 192.168.1.10/32
          - 10.10.0.0/24
          - 192.168.1.20/32
    # Deny all other access to k8s API
    - action: Deny
      protocol: TCP
      destination:
        nets:
          - 192.168.29.10/32
          - 192.168.29.11/32
          - 192.168.29.12/32
        ports: [6443]
  types:
    - Ingress
EOF
```

### Step 3: Talos API – Admin IP Allowlist

Talos API (ports 50000–51000) should be locked to admin IPs only. Apply via Talos machine config.

Create `talos-firewall-patch.yaml`:

```yaml
machine:
  network:
    kubespan:
      enabled: false
  # Talos host-level firewall — block Talos API from non-admin IPs
  sysctls: {}
  # Use Talos firewall feature (available v1.6+)
  firewall:
    enabled: true
    rules:
      - name: allow-admin-talos-api
        protocol: TCP
        destinationPort: 50000-51000
        action: accept
        ingressSubnets:
          - 192.168.1.10/32
          - 10.10.0.0/24
          - 192.168.1.20/32
      - name: deny-all-talos-api
        protocol: TCP
        destinationPort: 50000-51000
        action: drop
```

```bash
for node in 192.168.29.11 192.168.29.12 192.168.29.21 192.168.29.22 192.168.29.23; do
  echo "Applying firewall to $node..."
  talosctl --talosconfig talosconfig patch machineconfig \
    -n $node -e $node \
    --patch @talos-firewall-patch.yaml
done
```

### Step 4: MetalLB – Restrict LoadBalancer IPs to Admin Pool

Create a **separate MetalLB IP pool** for admin services so they are never accidentally assigned to tenant workloads:

```bash
kubectl --kubeconfig=kubeconfig apply -f - <<'EOF'
# Admin pool: .100–.109 (Longhorn, Canine, OpenCost)
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: admin-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.29.100-192.168.29.109
  autoAssign: false   # Only assigned explicitly via loadBalancerIP annotation
---
# Tenant pool: .110–.199 (Canine app LoadBalancers)
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: tenant-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.29.110-192.168.29.199
  autoAssign: true
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-advert
  namespace: metallb-system
spec:
  ipAddressPools:
    - admin-pool
    - tenant-pool
EOF
```

### Step 5: Verify Admin Access Control

```bash
# Verify GlobalNetworkPolicies are applied
kubectl --kubeconfig=kubeconfig get globalnetworkpolicy

# From an ADMIN IP — should succeed (200)
curl -s -o /dev/null -w "%{http_code}" http://192.168.29.101
# Expected: 200

# From a NON-ADMIN IP / tenant pod — should be blocked
# Simulate from a tenant namespace pod:
kubectl --kubeconfig=kubeconfig run test-tenant \
  --image=busybox -n canine-alice --restart=Never \
  -- wget --timeout=5 -q -O- http://192.168.29.101 \
  || echo "EXPECTED: blocked — admin service unreachable from tenant"

kubectl --kubeconfig=kubeconfig delete pod test-tenant -n canine-alice --ignore-not-found

# Verify IP pools
kubectl --kubeconfig=kubeconfig get ipaddresspool -n metallb-system
```

---

## Verification & Testing

### Complete Cluster Health Check

```bash
echo "=== CLUSTER HEALTH CHECK ==="

echo -e "\n1. Nodes (verify static hostnames cp01/cp02/w01/w02/w03):"
kubectl --kubeconfig=kubeconfig get nodes -o wide

echo -e "\n2. Non-running pods:"
kubectl --kubeconfig=kubeconfig get pods -A | grep -v Running | grep -v Completed

echo -e "\n3. Storage Classes:"
kubectl --kubeconfig=kubeconfig get storageclass

echo -e "\n4. MetalLB Pools:"
kubectl --kubeconfig=kubeconfig get ipaddresspool -n metallb-system

echo -e "\n5. Longhorn:"
kubectl --kubeconfig=kubeconfig get pods -n longhorn-system | grep -v Running | head -5

echo -e "\n6. Canine:"
kubectl --kubeconfig=kubeconfig get pods -n canine
kubectl --kubeconfig=kubeconfig get svc -n canine

echo -e "\n7. Prometheus:"
kubectl --kubeconfig=kubeconfig get pods -n prometheus-system | grep -v Running | head -5

echo -e "\n8. OpenCost:"
kubectl --kubeconfig=kubeconfig get pods -n opencost

echo -e "\n9. LoadBalancer Services:"
kubectl --kubeconfig=kubeconfig get svc -A --field-selector spec.type=LoadBalancer

echo -e "\n10. Node Resources:"
kubectl --kubeconfig=kubeconfig top nodes

echo -e "\n11. GlobalNetworkPolicies:"
kubectl --kubeconfig=kubeconfig get globalnetworkpolicy

echo -e "\n12. VIP:"
ping -c 2 192.168.29.10

echo -e "\n13. OpenCost API health:"
curl -s http://192.168.29.102:9003/healthz
```

### Test Static Hostnames

```bash
kubectl --kubeconfig=kubeconfig get nodes \
  -o custom-columns='NAME:.metadata.name,IP:.status.addresses[0].address,STATUS:.status.conditions[-1].type'

# All nodes should show their correct short names: cp01, cp02, w01, w02, w03
```

### Test Tenant Isolation

```bash
# Provision two test tenants
export CANINE_TOKEN="your-admin-api-token-here"
./provision-tenant.sh alice alice@example.com "AlicePass!123"
./provision-tenant.sh bob bob@example.com "BobPass!456"

# Deploy test pods in each namespace
kubectl --kubeconfig=kubeconfig run pod-a --image=busybox -n canine-alice \
  --restart=Never -- sleep 3600
kubectl --kubeconfig=kubeconfig run pod-b --image=nginx -n canine-bob \
  --restart=Never

sleep 15
BOB_IP=$(kubectl --kubeconfig=kubeconfig get pod pod-b -n canine-bob -o jsonpath='{.status.podIP}')

# Alice's pod should NOT reach Bob's pod (cross-namespace blocked)
kubectl --kubeconfig=kubeconfig exec pod-a -n canine-alice -- \
  wget --timeout=5 -q -O- http://$BOB_IP \
  && echo "FAIL: isolation broken" \
  || echo "PASS: cross-tenant traffic blocked"

# Alice's pod should NOT reach Canine admin (admin isolation blocked)
kubectl --kubeconfig=kubeconfig exec pod-a -n canine-alice -- \
  wget --timeout=5 -q -O- http://192.168.29.101 \
  && echo "FAIL: admin reachable from tenant" \
  || echo "PASS: admin service unreachable from tenant"

# Alice's pod SHOULD reach the internet (DNS + external egress)
kubectl --kubeconfig=kubeconfig exec pod-a -n canine-alice -- \
  wget --timeout=10 -q -O- http://example.com \
  && echo "PASS: external internet reachable" \
  || echo "FAIL: external internet blocked"

# Cleanup
kubectl --kubeconfig=kubeconfig delete pod pod-a -n canine-alice --ignore-not-found
kubectl --kubeconfig=kubeconfig delete pod pod-b -n canine-bob --ignore-not-found
```

---

## Operations & Maintenance

### Daily Health Checks

```bash
kubectl --kubeconfig=kubeconfig get nodes
kubectl --kubeconfig=kubeconfig get pods -A | grep -v Running | grep -v Completed
kubectl --kubeconfig=kubeconfig top nodes
kubectl --kubeconfig=kubeconfig top pods -A --sort-by=memory | head -20

# Daily cost snapshot
curl -sG http://192.168.29.102:9003/allocation \
  -d window=1d -d aggregate=namespace -d accumulate=true | \
  jq '.data[0] | to_entries[] | {ns: .key, cost: .value.totalCost}' | head -20
```

### Backup Procedures

#### etcd Snapshot

```bash
talosctl --talosconfig talosconfig --nodes 192.168.29.11 \
  etcd snapshot /tmp/etcd-backup-$(date +%Y%m%d-%H%M%S).db

talosctl --talosconfig talosconfig --nodes 192.168.29.11 \
  cp /tmp/etcd-backup-*.db ./
```

#### Configuration Backup

```bash
BACKUP_DIR="cluster-backup-$(date +%Y%m%d)"
mkdir -p $BACKUP_DIR
cp *.yaml talosconfig kubeconfig canine-values.yaml opencost-values.yaml \
   apply-tenant-isolation.sh provision-tenant.sh tenant-cost-report.sh \
   $BACKUP_DIR/
tar -czf ${BACKUP_DIR}.tar.gz $BACKUP_DIR/
# Encrypt before storing off-cluster:
gpg --symmetric --cipher-algo AES256 ${BACKUP_DIR}.tar.gz
echo "Encrypted backup: ${BACKUP_DIR}.tar.gz.gpg"
```

### Kubernetes Upgrade

```bash
talosctl --talosconfig talosconfig --nodes 192.168.29.11 upgrade-k8s --to 1.31.0
kubectl --kubeconfig=kubeconfig get nodes
talosctl --talosconfig talosconfig --nodes 192.168.29.12 upgrade-k8s --to 1.31.0

for node in 192.168.29.21 192.168.29.22 192.168.29.23; do
  kubectl --kubeconfig=kubeconfig drain $node --ignore-daemonsets --delete-emptydir-data
  talosctl --talosconfig talosconfig --nodes $node upgrade-k8s --to 1.31.0
  sleep 120
  kubectl --kubeconfig=kubeconfig uncordon $node
done
```

### Talos OS Upgrade

```bash
SCHEMATIC_ID="dfd1ac9abdf529ca644694b17af0ce1a2ae23a5cccdff39439aa7f0774901e90"
NEW_VERSION="v1.12.0"

for node in 192.168.29.11 192.168.29.12; do
  talosctl --talosconfig talosconfig --nodes $node upgrade \
    --image factory.talos.dev/installer/${SCHEMATIC_ID}:${NEW_VERSION}
  sleep 180
done

for node in 192.168.29.21 192.168.29.22 192.168.29.23; do
  kubectl --kubeconfig=kubeconfig drain $node --ignore-daemonsets --delete-emptydir-data
  talosctl --talosconfig talosconfig --nodes $node upgrade \
    --image factory.talos.dev/installer/${SCHEMATIC_ID}:${NEW_VERSION}
  sleep 180
  kubectl --kubeconfig=kubeconfig uncordon $node
done
```

### Add a Worker Node

```bash
talosctl machineconfig patch worker.yaml --patch @worker-patch.yaml --output w04.yaml

talosctl machineconfig patch w04.yaml \
  --patch '[
    {"op":"replace","path":"/machine/network/interfaces","value":[{"interface":"ens192","dhcp":false,"addresses":["192.168.29.24/24"],"routes":[{"network":"0.0.0.0/0","gateway":"192.168.29.1"}]}]},
    {"op":"add","path":"/machine/network/hostname","value":"w04"}
  ]' \
  --output w04.yaml

talosctl apply-config --insecure --nodes 192.168.29.24 --file w04.yaml
sleep 300
kubectl --kubeconfig=kubeconfig get nodes
```

---

## Optional Hardening & Improvements

These are **not required** for basic operation but are strongly recommended for production deployments serving real end users.

### OPT-1: cert-manager – Automatic TLS for All Services

Eliminates manual certificate management. Issues free Let's Encrypt certificates automatically.

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --kubeconfig=kubeconfig \
  --set installCRDs=true \
  --wait

# Create a ClusterIssuer for Let's Encrypt production
kubectl --kubeconfig=kubeconfig apply -f - <<'EOF'
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@vaheed.net
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: ""
EOF

# Verify
kubectl --kubeconfig=kubeconfig get clusterissuer
```

**Why:** Tenant apps deployed via Canine can then get automatic HTTPS with zero configuration.

---

### OPT-2: NGINX Ingress Controller – Single Entry Point for Tenant Apps

Gives Canine tenant applications a shared, hostable HTTPS ingress instead of each app needing its own LoadBalancer IP.

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --kubeconfig=kubeconfig \
  --set controller.service.type=LoadBalancer \
  --set controller.service.loadBalancerIP=192.168.29.110 \
  --set controller.config.use-forwarded-headers="true" \
  --wait

kubectl --kubeconfig=kubeconfig get svc -n ingress-nginx
echo "Ingress entry point: 192.168.29.110"
```

**Why:** All tenant app domains (e.g. `*.apps.29.talos.vaheed.net`) resolve to a single IP. Canine creates Ingress objects automatically; cert-manager issues TLS.

---

### OPT-3: Velero – Backup and Disaster Recovery

Backs up Kubernetes resources and Longhorn PVC data to S3-compatible object storage (e.g. MinIO).

```bash
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
helm repo update

# First deploy MinIO in-cluster or use an external S3 endpoint
# Then install Velero:
helm install velero vmware-tanzu/velero \
  --namespace velero \
  --create-namespace \
  --kubeconfig=kubeconfig \
  --set configuration.backupStorageLocation[0].provider=aws \
  --set configuration.backupStorageLocation[0].bucket=velero-backups \
  --set configuration.backupStorageLocation[0].config.region=minio \
  --set configuration.backupStorageLocation[0].config.s3ForcePathStyle=true \
  --set configuration.backupStorageLocation[0].config.s3Url=http://minio.minio-system.svc.cluster.local:9000 \
  --set credentials.secretContents.cloud="[default]\naws_access_key_id=minio-access-key\naws_secret_access_key=minio-secret-key" \
  --set initContainers[0].name=velero-plugin-for-aws \
  --set initContainers[0].image=velero/velero-plugin-for-aws:v1.9.0 \
  --set initContainers[0].volumeMounts[0].mountPath=/target \
  --set initContainers[0].volumeMounts[0].name=plugins \
  --wait

# Create a daily backup schedule
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --ttl 720h \
  --include-namespaces '*'
```

**Why:** etcd snapshots protect cluster state; Velero protects tenant PVC data and all Kubernetes objects — enabling full disaster recovery.

---

### OPT-4: Prometheus Alerting – Cost and Resource Alerts

Alert when a tenant exceeds 80% of their quota or when cluster-wide cost exceeds a threshold.

```bash
kubectl --kubeconfig=kubeconfig apply -f - <<'EOF'
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: tenant-cost-alerts
  namespace: prometheus-system
spec:
  groups:
    - name: tenant-resource-usage
      rules:
        - alert: TenantCPUQuotaHigh
          expr: |
            (kube_resourcequota{resource="limits.cpu",type="used"}
            / kube_resourcequota{resource="limits.cpu",type="hard"}) > 0.80
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Tenant namespace {{ $labels.namespace }} is using >80% CPU quota"

        - alert: TenantMemoryQuotaHigh
          expr: |
            (kube_resourcequota{resource="limits.memory",type="used"}
            / kube_resourcequota{resource="limits.memory",type="hard"}) > 0.80
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Tenant namespace {{ $labels.namespace }} is using >80% memory quota"
EOF
```

**Why:** Proactively catch runaway workloads before they hit hard quota limits and cause disruption.

---

### OPT-5: Falco – Runtime Security Monitoring

Detects suspicious activity inside tenant containers (e.g. shell spawning, privilege escalation, unexpected network connections).

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update

helm install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  --kubeconfig=kubeconfig \
  --set driver.kind=modern_ebpf \
  --set falcosidekick.enabled=true \
  --set falcosidekick.config.slack.webhookurl="https://hooks.slack.com/your-webhook" \
  --wait

kubectl --kubeconfig=kubeconfig get pods -n falco
```

**Why:** NetworkPolicy prevents network paths but Falco catches what happens inside running containers — the last line of defense against compromised tenant workloads.

---

### OPT-6: OpenCost Showback via API – Automated Monthly Invoice

Run this monthly (via cron or CI) to generate a tenant-facing cost report:

```bash
# Add to crontab: 0 8 1 * * /path/to/monthly-invoice.sh
cat > monthly-invoice.sh <<'SCRIPT'
#!/usr/bin/env bash
set -euo pipefail

OPENCOST_API="http://192.168.29.102:9003"
MONTH=$(date -d "last month" +%Y-%m 2>/dev/null || date -v-1m +%Y-%m)
START="${MONTH}-01T00:00:00Z"
END=$(date -u +%Y-%m-%dT%H:%M:%SZ)
OUTPUT="invoice-${MONTH}.tsv"

echo "Generating invoice for $MONTH..."

curl -sG "${OPENCOST_API}/allocation" \
  -d "window=${START},${END}" \
  -d "aggregate=namespace" \
  -d "accumulate=true" | \
  jq -r '.data[0] | to_entries[] |
    select(.key | startswith("canine-")) |
    [.key, .value.cpuCost, .value.ramCost,
     .value.pvCost, .value.totalCost] | @tsv' \
  | sort -t$'\t' -k5 -rn > "$OUTPUT"

echo "Invoice saved: $OUTPUT"
cat "$OUTPUT"
SCRIPT
chmod +x monthly-invoice.sh
```

---

### OPT-7: Node Taints – Separate System and Tenant Workloads

Prevent tenant pods from landing on nodes that run platform components:

```bash
# Optional: dedicate w01 to platform workloads (Canine, OpenCost, Longhorn)
kubectl --kubeconfig=kubeconfig taint nodes w01 \
  dedicated=platform:NoSchedule

kubectl --kubeconfig=kubeconfig label nodes w01 \
  dedicated=platform

# w02 and w03 serve tenant workloads
kubectl --kubeconfig=kubeconfig label nodes w02 w03 \
  dedicated=tenants

# Add tolerations to platform Helm charts (Canine, OpenCost) via values:
# tolerations:
#   - key: dedicated
#     operator: Equal
#     value: platform
#     effect: NoSchedule
# nodeSelector:
#   dedicated: platform
```

**Why:** Ensures a noisy or misbehaving tenant cannot exhaust resources needed by the platform control plane.

---

## Troubleshooting

### Static Hostname Not Applied

```bash
# If a node still shows a random name after bootstrap:
talosctl --talosconfig talosconfig --nodes 192.168.29.21 get hostname

# Re-apply the node config with the hostname patch
talosctl apply-config --nodes 192.168.29.21 --file w01.yaml --mode reboot
sleep 120
kubectl --kubeconfig=kubeconfig get nodes
```

### Nodes Not Ready

```bash
talosctl --talosconfig talosconfig --nodes 192.168.29.21 service kubelet status
kubectl --kubeconfig=kubeconfig get pods -n calico-system
kubectl --kubeconfig=kubeconfig logs -n calico-system -l k8s-app=calico-node --tail=50
kubectl --kubeconfig=kubeconfig describe node w01
```

### VIP Not Responding

```bash
for node in 192.168.29.11 192.168.29.12; do
  echo "Checking $node..."
  talosctl --talosconfig talosconfig --nodes $node get links | grep -A 2 ens192
done
talosctl --talosconfig talosconfig --nodes 192.168.29.11 service etcd status
talosctl --talosconfig talosconfig --endpoints 192.168.29.10 get members
```

### Longhorn – Volumes Not Attaching

```bash
kubectl --kubeconfig=kubeconfig logs -n longhorn-system -l app=longhorn-manager --tail=100
kubectl --kubeconfig=kubeconfig get volumes -n longhorn-system
kubectl --kubeconfig=kubeconfig get nodes.longhorn.io -n longhorn-system

for node in 192.168.29.21 192.168.29.22 192.168.29.23; do
  echo "Disk on $node:"
  talosctl --talosconfig talosconfig --nodes $node df /dev/sdb
done
```

### MetalLB Not Assigning IPs

```bash
kubectl --kubeconfig=kubeconfig get pods -n metallb-system -l component=speaker
kubectl --kubeconfig=kubeconfig logs -n metallb-system -l component=controller --tail=100
kubectl --kubeconfig=kubeconfig get ipaddresspool,l2advertisement -n metallb-system -o yaml
```

### Canine – Deployment Failures

```bash
kubectl --kubeconfig=kubeconfig logs -n canine -l app.kubernetes.io/component=web --tail=100
kubectl --kubeconfig=kubeconfig logs -n canine -l app.kubernetes.io/component=buildkit --tail=100
kubectl --kubeconfig=kubeconfig get pvc -n canine
kubectl --kubeconfig=kubeconfig rollout restart deployment -n canine
```

### Canine – Tenant Isolation Issues

```bash
# Missing NetworkPolicy?
kubectl --kubeconfig=kubeconfig get networkpolicy -n <tenant-ns>

# Quota exhausted?
kubectl --kubeconfig=kubeconfig describe resourcequota -n <tenant-ns>

# LimitRange missing?
kubectl --kubeconfig=kubeconfig describe limitrange -n <tenant-ns>

# Leaked ClusterRoleBinding?
kubectl --kubeconfig=kubeconfig get clusterrolebinding | grep <tenant-ns>
# Expected: no output
```

### GlobalNetworkPolicy Not Blocking

```bash
# Verify Calico is running
kubectl --kubeconfig=kubeconfig get pods -n calico-system

# List all GlobalNetworkPolicies
kubectl --kubeconfig=kubeconfig get globalnetworkpolicy

# Check Calico Felix logs for policy errors
kubectl --kubeconfig=kubeconfig logs -n calico-system -l k8s-app=calico-node \
  -c calico-node --tail=100 | grep -i policy
```

### OpenCost – No Data / Zero Costs

```bash
kubectl --kubeconfig=kubeconfig logs -n opencost -l app.kubernetes.io/name=opencost --tail=100

# Verify Prometheus reachable from OpenCost pod
kubectl --kubeconfig=kubeconfig exec -n opencost \
  $(kubectl --kubeconfig=kubeconfig get pod -n opencost -o jsonpath='{.items[0].metadata.name}') \
  -- wget -qO- \
  "http://prometheus-server.prometheus-system.svc.cluster.local/api/v1/query?query=up" \
  | head -c 200

# Test API
curl -sG http://192.168.29.102:9003/allocation \
  -d window=1d -d aggregate=namespace | jq '.code'
# Expected: 200
```

### Storage Class Issues

```bash
kubectl --kubeconfig=kubeconfig get storageclass

# Ensure Longhorn is default
kubectl --kubeconfig=kubeconfig patch storageclass longhorn \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### Emergency Cluster Reset

> **⚠️ WARNING: Destroys all data permanently.**

```bash
kubectl --kubeconfig=kubeconfig delete pvc --all -A
kubectl --kubeconfig=kubeconfig delete deployments --all -A
kubectl --kubeconfig=kubeconfig delete services --all -A

for node in 192.168.29.11 192.168.29.12 192.168.29.21 192.168.29.22 192.168.29.23; do
  echo "Resetting $node..."
  talosctl --talosconfig talosconfig reset -n $node -e $node --graceful=false --reboot
done
```

---

## Quick Reference

### Essential Commands

```bash
# Cluster
kubectl --kubeconfig=kubeconfig get nodes -o wide    # verify static hostnames
kubectl --kubeconfig=kubeconfig get pods -A | grep -v Running | grep -v Completed
talosctl --talosconfig talosconfig health

# Resources
kubectl --kubeconfig=kubeconfig top nodes
kubectl --kubeconfig=kubeconfig top pods -A --sort-by=memory

# Storage
kubectl --kubeconfig=kubeconfig get storageclass,pvc -A

# Services
kubectl --kubeconfig=kubeconfig get svc -A --field-selector spec.type=LoadBalancer

# Canine
kubectl --kubeconfig=kubeconfig get pods -n canine
curl -s -H "Authorization: Bearer $CANINE_TOKEN" \
  http://canine.29.talos.vaheed.net/api/v1/users | jq .

# OpenCost
kubectl --kubeconfig=kubeconfig get pods -n opencost
curl -sG http://192.168.29.102:9003/allocation -d window=1d -d aggregate=namespace | jq '.data[0]'

# Security
kubectl --kubeconfig=kubeconfig get globalnetworkpolicy
kubectl --kubeconfig=kubeconfig get resourcequota -A
kubectl --kubeconfig=kubeconfig get networkpolicy -A
```

### Important Files

| File | Purpose |
|------|---------|
| `talosconfig` | Talos CLI authentication |
| `kubeconfig` | Kubernetes CLI authentication |
| `cp01.yaml`, `cp02.yaml` | Control plane node configs (with static hostname) |
| `w01.yaml` – `w03.yaml` | Worker node configs (with static hostname) |
| `cp-patch.yaml` | Control plane patch template |
| `worker-patch.yaml` | Worker patch template |
| `longhorn-disk.yaml` | Longhorn disk mount patch |
| `talos-firewall-patch.yaml` | Talos host firewall (admin IPs only) |
| `canine-values.yaml` | Canine Helm values |
| `opencost-values.yaml` | OpenCost Helm values |
| `apply-tenant-isolation.sh` | Apply isolation policies to a tenant namespace |
| `provision-tenant.sh` | End-to-end tenant provisioning (API + isolation) |
| `tenant-cost-report.sh` | Per-tenant monthly cost report |
| `monthly-invoice.sh` | Monthly cost invoice via OpenCost API |

### Network Reference

| Purpose | Address / Range |
|---------|----------------|
| Gateway | 192.168.29.1 |
| Kubernetes VIP | 192.168.29.10 |
| Control Planes | 192.168.29.11 – 12 |
| Workers | 192.168.29.21 – 23 |
| Admin MetalLB Pool | 192.168.29.100 – 109 |
| Tenant MetalLB Pool | 192.168.29.110 – 199 |
| Longhorn UI | 192.168.29.100 |
| Canine PaaS | 192.168.29.101 |
| OpenCost UI / API | 192.168.29.102 |
| Ingress (OPT-2) | 192.168.29.110 |

### Port Reference

| Port | Service | Access |
|------|---------|--------|
| 6443 | Kubernetes API | Admin IPs only |
| 50000–51000 | Talos API | Admin IPs only |
| 80 / 443 | HTTP/HTTPS | Ingress / MetalLB |
| 9003 | OpenCost REST API | Admin IPs only |
| 9090 | OpenCost UI | Admin IPs only |
| 8081 | OpenCost MCP | In-cluster |
| 8000 | Longhorn UI (internal) | Admin IPs only |

---

## Support Resources

- **Talos Linux**: https://www.talos.dev/
- **Kubernetes**: https://kubernetes.io/docs/
- **Calico**: https://docs.projectcalico.org/ | **GlobalNetworkPolicy**: https://docs.projectcalico.org/reference/resources/globalnetworkpolicy
- **Longhorn**: https://longhorn.io/docs/
- **MetalLB**: https://metallb.universe.tf/
- **Canine PaaS**: https://canine.sh | https://github.com/CanineHQ/canine | https://docs.canine.sh
- **OpenCost**: https://opencost.io | https://github.com/opencost/opencost | https://opencost.io/docs
- **OpenCost API**: https://opencost.io/docs/integrations/api-examples/
- **cert-manager**: https://cert-manager.io/docs/
- **Velero**: https://velero.io/docs/
- **Falco**: https://falco.org/docs/

---

## Summary

### What We Built

✅ **HA Control Plane** — 2 nodes, VIP `192.168.29.10`, static hostnames (`cp01`, `cp02`), automatic etcd failover

✅ **Worker Pool** — 3 workers with static hostnames (`w01–w03`), isolated from control plane, dedicated Longhorn disks

✅ **Networking** — Calico v3.30.5 CNI, MetalLB v0.14.9 with separate admin (`.100–.109`) and tenant (`.110–.199`) IP pools

✅ **Storage** — Longhorn v1.10.1, 3× replication, RWO + RWX, auto-provisioning, UI on `.100`

✅ **Canine PaaS** — Git-push deploys, BuildKit image builds, full REST API, dashboard on `.101` (admin only)

✅ **Tenant Isolation** — Namespace per tenant with: Calico deny-all NetworkPolicy, ResourceQuota (CPU/RAM/storage/object counts), LimitRange (auto-inject defaults + max caps), scoped RBAC (no cluster access), GlobalNetworkPolicy blocking tenant-to-admin egress

✅ **Admin Access Control** — Calico `GlobalNetworkPolicy` protecting all admin namespaces; Talos host firewall; MetalLB admin pool locked via `autoAssign: false`; Kubernetes API restricted to admin CIDRs

✅ **OpenCost** — Real-time cost allocation per namespace/tenant, on-prem pricing, REST API on `.102:9003`, UI on `.102`, MCP on `8081`

✅ **Automation Scripts** — `provision-tenant.sh`, `apply-tenant-isolation.sh`, `tenant-cost-report.sh`, `monthly-invoice.sh` — all API-first

✅ **Prometheus** — Dedicated instance in `prometheus-system`, Longhorn-backed, pre-wired for OpenCost

✅ **Optional Hardening** — cert-manager TLS, NGINX Ingress, Velero backup, Prometheus alerts, Falco runtime security, node taints, monthly invoice automation

---

**Access Points:**

| Service | URL / Endpoint | Access |
|---------|----------------|--------|
| Kubernetes API | https://29.talos.vaheed.net:6443 | Admin IPs only |
| Longhorn UI | http://192.168.29.100 | Admin IPs only |
| Canine Dashboard | http://canine.29.talos.vaheed.net | Admin IPs only |
| Canine REST API | http://canine.29.talos.vaheed.net/api/v1/ | Admin IPs only |
| OpenCost UI | http://192.168.29.102 | Admin IPs only |
| OpenCost REST API | http://192.168.29.102:9003 | Admin IPs only |
| OpenCost MCP | http://opencost.opencost.svc.cluster.local:8081 | In-cluster |
| Tenant Apps (Ingress) | http://192.168.29.110 (OPT-2) | Public |

---

*Last Updated: February 2026 — Version 7.0*
