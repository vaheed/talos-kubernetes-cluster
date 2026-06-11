# Talos HA Kubernetes Cluster — Production SaaS Architecture
## 5 Control Planes · 5 Workers · Rancher Multi-Tenancy · Velero · Full Observability
### Complete Production Guide — Version 8.0 — June 2026

> API-first · Copy-paste ready · Fully declarative · Multi-tenant SaaS

---

## Component Versions (Latest / LTS — June 2026)

| Component | Version | Notes |
|-----------|---------|-------|
| **Talos Linux** | **v1.13.0** | 2026-04-27 · Linux kernel 6.18.x |
| **Kubernetes** | **v1.34.x** | LTS — tested by all components |
| **etcd** | **v3.6.11** | Bundled in Talos |
| **containerd** | **v2.2.4** | Bundled in Talos |
| **Calico CNI** | **v3.31.5** | NetworkPolicy + GlobalNetworkPolicy |
| **MetalLB** | **v0.15.3** | L2 mode, split IP pools |
| **Traefik Ingress** | **v3.6.13** | Replaces deprecated NGINX Ingress |
| **cert-manager** | **v1.20.2** | Automatic TLS |
| **Longhorn CSI** | **v1.12.0** | Distributed block storage, v2 data engine |
| **Rancher** | **v2.14.2** | 2026-05-28 · Multi-cluster management |
| **Velero** | **v1.18.0** | CNCF Sandbox · CSI snapshot + concurrent backup |
| **kube-prometheus-stack** | **v86.2.2** | Prometheus + Grafana + Alertmanager |
| **Metrics Server** | **v0.8.1** | `kubectl top` |
| **OpenCost** | **latest** | CNCF · Real-time cost allocation |
| **Falco** | **latest** | Runtime security |
| **Helm** | **v3.20+** | Package manager |
| **talosctl** | **v1.13.0** | Must match Talos version |

---

## Table of Contents

1. [Cluster Topology](#1-cluster-topology)
2. [Architecture Diagram](#2-architecture-diagram)
3. [Prerequisites](#3-prerequisites)
4. [Installation — Talos Bootstrap](#4-installation--talos-bootstrap)
5. [Network Components](#5-network-components)
6. [Storage — Longhorn v1.12.0](#6-storage--longhorn-v1120)
7. [cert-manager & TLS](#7-cert-manager--tls)
8. [Rancher Installation](#8-rancher-installation)
9. [Multi-Tenancy Design](#9-multi-tenancy-design)
10. [Rancher API Automation — Tenant Onboarding](#10-rancher-api-automation--tenant-onboarding)
11. [SaaS App Deployment — WordPress per Tenant](#11-saas-app-deployment--wordpress-per-tenant)
12. [Velero — Backup & Restore per Tenant](#12-velero--backup--restore-per-tenant)
13. [Monitoring & Observability](#13-monitoring--observability)
14. [OpenCost — Cost Allocation per Namespace](#14-opencost--cost-allocation-per-namespace)
15. [Security Hardening](#15-security-hardening)
16. [Admin Access Control](#16-admin-access-control)
17. [Operations & Maintenance](#17-operations--maintenance)
18. [Scaling & HA Best Practices](#18-scaling--ha-best-practices)
19. [Troubleshooting](#19-troubleshooting)
20. [Quick Reference](#20-quick-reference)

---

## 1. Cluster Topology

### Network Configuration

| Item | Value |
|------|-------|
| Network | `192.168.29.0/24` |
| Gateway | `192.168.29.1` |
| VIP (Kubernetes API HA) | `192.168.29.10` |
| VIP Hostname | `k8s.prod.example.com` |
| Pod Network | `10.244.0.0/16` |
| Service Network | `10.96.0.0/12` |
| DNS Domain | `cluster.local` |

### Control Plane Nodes (5 — etcd quorum: 3, tolerates 2 failures)

| Name | IP | Hostname | Role | Specs |
|------|----|----------|------|-------|
| cp01 | 192.168.29.11 | cp01 | Control Plane + etcd | 4 vCPU · 8 GB RAM · 100 GB disk |
| cp02 | 192.168.29.12 | cp02 | Control Plane + etcd | 4 vCPU · 8 GB RAM · 100 GB disk |
| cp03 | 192.168.29.13 | cp03 | Control Plane + etcd | 4 vCPU · 8 GB RAM · 100 GB disk |
| cp04 | 192.168.29.14 | cp04 | Control Plane + etcd | 4 vCPU · 8 GB RAM · 100 GB disk |
| cp05 | 192.168.29.15 | cp05 | Control Plane + etcd | 4 vCPU · 8 GB RAM · 100 GB disk |

> **Why 5 control planes?** Five etcd members (quorum = 3) tolerate simultaneous failure of 2 nodes — safe for rolling upgrades without downtime. The VIP (`192.168.29.10`) always points to a healthy control plane leader.

### Worker Nodes (5 — dedicated Longhorn storage disks)

| Name | IP | Hostname | OS Disk | Data Disk | Role |
|------|----|----------|---------|-----------|------|
| w01 | 192.168.29.21 | w01 | /dev/sda 100 GB | /dev/sdb 200 GB | Workload + Longhorn |
| w02 | 192.168.29.22 | w02 | /dev/sda 100 GB | /dev/sdb 200 GB | Workload + Longhorn |
| w03 | 192.168.29.23 | w03 | /dev/sda 100 GB | /dev/sdb 200 GB | Workload + Longhorn |
| w04 | 192.168.29.24 | w04 | /dev/sda 100 GB | /dev/sdb 200 GB | Workload + Longhorn |
| w05 | 192.168.29.25 | w05 | /dev/sda 100 GB | /dev/sdb 200 GB | Workload + Longhorn |

### Service IP Allocation

| Service | Address / Range |
|---------|----------------|
| Kubernetes VIP | `192.168.29.10` |
| MetalLB — Admin Pool (`autoAssign: false`) | `192.168.29.100–109` |
| MetalLB — Tenant Ingress Pool | `192.168.29.110–149` |
| MetalLB — General Pool | `192.168.29.150–199` |
| Longhorn UI | `192.168.29.100` |
| Rancher UI | `192.168.29.101` |
| OpenCost UI / API | `192.168.29.102` |
| Grafana UI | `192.168.29.103` |
| Tenant Traefik Ingress | `192.168.29.110` |

### Admin Source IPs (replace before applying)

| Purpose | IP / CIDR |
|---------|-----------|
| Admin workstation | `192.168.1.10/32` |
| VPN / jump host | `10.10.0.0/24` |
| CI/CD server | `192.168.1.20/32` |

---

## 2. Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                         Gateway  192.168.29.1                                    │
└──────────────────────────────────────┬───────────────────────────────────────────┘
                                       │
┌──────────────────────────────────────┴───────────────────────────────────────────┐
│           VIP: 192.168.29.10  (k8s.prod.example.com)                            │
│           HA Kubernetes API — Admin IPs only                                     │
│           Active etcd leader rotates automatically across 5 CPs                  │
└───────┬──────────┬──────────┬──────────┬──────────┬───────────────────────────── ┘
        │          │          │          │          │
  ┌─────┴───┐ ┌────┴────┐ ┌──┴─────┐ ┌──┴─────┐ ┌──┴─────┐
  │  cp01   │ │  cp02   │ │  cp03  │ │  cp04  │ │  cp05  │
  │ .11     │ │ .12     │ │ .13    │ │ .14    │ │ .15    │
  │ Control │◄►│ Control │◄►│Control │◄►│Control │◄►│Control │
  │ Plane   │ │ Plane   │ │ Plane  │ │ Plane  │ │ Plane  │
  │ + etcd  │ │ + etcd  │ │ + etcd │ │ + etcd │ │ + etcd │
  └─────────┘ └─────────┘ └────────┘ └────────┘ └────────┘
        │             etcd quorum = 3/5 · tolerates 2 failures
        └──────────────────┬──────────────────────────────────
           ┌───────────────┼───────────────┬──────────────────┐
    ┌──────┴──────┐ ┌──────┴──────┐ ┌──────┴───────┐ ┌───────┴──────┐ ┌──────────────┐
    │   w01 .21   │ │   w02 .22   │ │   w03 .23    │ │   w04 .24    │ │   w05 .25    │
    │ Worker      │ │ Worker      │ │ Worker        │ │ Worker       │ │ Worker       │
    │ /dev/sdb    │ │ /dev/sdb    │ │ /dev/sdb      │ │ /dev/sdb     │ │ /dev/sdb     │
    │ 200GB       │ │ 200GB       │ │ 200GB         │ │ 200GB        │ │ 200GB        │
    │ Longhorn    │ │ Longhorn    │ │ Longhorn      │ │ Longhorn     │ │ Longhorn     │
    └─────────────┘ └─────────────┘ └───────────────┘ └──────────────┘ └──────────────┘

╔══════════════════════════════════════════════════════════════════════════════════╗
║                         PLATFORM LAYER (cattle-system)                          ║
║  Rancher v2.14.2 ─ Multi-cluster mgmt · Project/Namespace/RBAC automation      ║
╚══════════════════════════════════════════════════════════════════════════════════╝

╔══════════════════════════════════════════════════════════════════════════════════╗
║                       MULTI-TENANT NAMESPACES                                   ║
║  tenant-alice-ns  │  tenant-bob-ns  │  tenant-corp-ns  │  tenant-N-ns          ║
║  ResourceQuota    │  ResourceQuota  │  ResourceQuota   │  ResourceQuota         ║
║  NetworkPolicy    │  NetworkPolicy  │  NetworkPolicy   │  NetworkPolicy         ║
║  Velero Schedule  │  Velero Sched.  │  Velero Sched.   │  Velero Sched.        ║
║  WordPress+DB     │  WordPress+DB   │  WordPress+DB    │  App+DB               ║
╚══════════════════════════════════════════════════════════════════════════════════╝

╔══════════════════════════════════════════════════════════════════════════════════╗
║                         INFRASTRUCTURE LAYER                                    ║
║  Calico v3.31.5  │  MetalLB v0.15.3  │  Traefik v3.6.13  │  Longhorn v1.12.0  ║
║  CNI + NetPol    │  LoadBalancer IPs │  Ingress / TLS     │  Dist. Storage     ║
╚══════════════════════════════════════════════════════════════════════════════════╝

╔══════════════════════════════════════════════════════════════════════════════════╗
║                        OBSERVABILITY & BACKUP                                   ║
║  kube-prometheus-stack v86.2.2  │  OpenCost (latest)  │  Velero v1.18.0       ║
║  Prometheus + Grafana + Alerts  │  Cost per namespace  │  CSI snapshot backup  ║
╚══════════════════════════════════════════════════════════════════════════════════╝

╔══════════════════════════════════════════════════════════════════════════════════╗
║                          SECURITY LAYER                                          ║
║  Talos immutable OS  │  PodSecurity (restricted)  │  Falco runtime security    ║
║  Calico GlobalNetPol │  mTLS (Talos API)          │  cert-manager TLS         ║
╚══════════════════════════════════════════════════════════════════════════════════╝
```

---

## 3. Prerequisites

### Hardware Requirements

**Control Plane Nodes (×5):** 4 vCPU · 8 GB RAM · 100 GB OS disk · NIC `ens192`

**Worker Nodes (×5):** 8 vCPU · 16 GB RAM · 100 GB OS disk (`/dev/sda`) · 200 GB data disk (`/dev/sdb`) · NIC `ens192`

> For a production SaaS cluster hosting dozens of tenants, workers with 32 GB RAM and 500 GB data disks are recommended.

### Software Tools (Admin Workstation)

```bash
# talosctl v1.13.0
wget https://github.com/siderolabs/talos/releases/download/v1.13.0/talosctl-linux-amd64
chmod +x talosctl-linux-amd64 && sudo mv talosctl-linux-amd64 /usr/local/bin/talosctl

# kubectl v1.34.x
curl -LO "https://dl.k8s.io/release/v1.34.0/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/kubectl

# helm v3.20+
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# velero CLI v1.18.0
wget https://github.com/vmware-tanzu/velero/releases/download/v1.18.0/velero-v1.18.0-linux-amd64.tar.gz
tar xf velero-v1.18.0-linux-amd64.tar.gz
sudo mv velero-v1.18.0-linux-amd64/velero /usr/local/bin/velero

# Verify
talosctl version --client
kubectl version --client
helm version
velero version --client-only
```

### DNS Records (create before cluster bootstrap)

```
k8s.prod.example.com         → 192.168.29.10   (VIP — Kubernetes API)
rancher.prod.example.com     → 192.168.29.101  (Rancher UI)
grafana.prod.example.com     → 192.168.29.103  (Grafana UI)
*.tenant.prod.example.com    → 192.168.29.110  (Tenant Traefik wildcard)
```

### Open Ports

| Port | Service | Direction |
|------|---------|-----------|
| 6443 | Kubernetes API | Admin IPs → VIP |
| 50000–51000 | Talos API | Admin IPs → all nodes |
| 2379–2380 | etcd | CP nodes ↔ CP nodes |
| 80 / 443 | Ingress | Internet → Traefik LB |
| 4789 | Calico VXLAN | Nodes ↔ Nodes |

---

## 4. Installation — Talos Bootstrap

### Step 1 — Download Talos Image for VMware

```bash
# Get the factory image with VMware Tools + open-vm-tools extension
# Visit https://factory.talos.dev to generate a custom image
# Example schematic with VMware Tools + iscsi-tools (required for Longhorn v1.12 v2 engine):

SCHEMATIC_ID="a9d2d26b9f0b1b1c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7"
TALOS_VERSION="v1.13.0"

wget "https://factory.talos.dev/image/${SCHEMATIC_ID}/${TALOS_VERSION}/vmware-amd64.iso" \
  -O talos-vmware-v1.13.0.iso
```

> **Longhorn v1.12.0 Note:** The v2 data engine requires `iscsi_tcp` kernel module. Add `siderolabs/iscsi-tools` and `siderolabs/util-linux-tools` to your Talos schematic at factory.talos.dev.

### Step 2 — Create VMs in vSphere

1. Upload ISO to your datastore.
2. Create **10 VMs** (5 CP + 5 Worker):
   - CP nodes: 4 vCPU · 8 GB RAM · 100 GB disk
   - Worker nodes: 8 vCPU · 16 GB RAM · 100 GB OS disk + 200 GB data disk
3. Boot each VM from ISO — they enter **maintenance mode** automatically.

### Step 3 — Generate Base Cluster Configuration

```bash
# Generate PKI + base configs
talosctl gen config "prod-saas-cluster" "https://k8s.prod.example.com:6443" \
  --kubernetes-version v1.34.0 \
  --install-image factory.talos.dev/installer/${SCHEMATIC_ID}:v1.13.0 \
  --output-dir ./cluster-configs/

# Files created:
#   cluster-configs/controlplane.yaml
#   cluster-configs/worker.yaml
#   cluster-configs/talosconfig

ls -la cluster-configs/
```

### Step 4 — Control Plane Base Patch (`cp-patch.yaml`)

This patch is applied to ALL control plane nodes before per-node customisation.

```yaml
# cp-patch.yaml
machine:
  network:
    interfaces:
      - interface: ens192
        vip:
          ip: 192.168.29.10
    extraHostEntries:
      - ip: 192.168.29.10
        aliases: ["k8s.prod.example.com"]
      - ip: 192.168.29.11
        aliases: ["cp01"]
      - ip: 192.168.29.12
        aliases: ["cp02"]
      - ip: 192.168.29.13
        aliases: ["cp03"]
      - ip: 192.168.29.14
        aliases: ["cp04"]
      - ip: 192.168.29.15
        aliases: ["cp05"]
      - ip: 192.168.29.21
        aliases: ["w01"]
      - ip: 192.168.29.22
        aliases: ["w02"]
      - ip: 192.168.29.23
        aliases: ["w03"]
      - ip: 192.168.29.24
        aliases: ["w04"]
      - ip: 192.168.29.25
        aliases: ["w05"]

  certSANs:
    - 192.168.29.10
    - k8s.prod.example.com
    - 192.168.29.11
    - 192.168.29.12
    - 192.168.29.13
    - 192.168.29.14
    - 192.168.29.15

  kubelet:
    nodeIP:
      validSubnets:
        - 192.168.29.0/24

cluster:
  network:
    cni:
      name: none            # Calico is installed separately
    dnsDomain: cluster.local
    podSubnets:
      - 10.244.0.0/16
    serviceSubnets:
      - 10.96.0.0/12
  allowSchedulingOnControlPlanes: false   # Strict separation: CPs run only etcd+API
  etcd:
    advertisedSubnets:
      - 192.168.29.0/24
  apiServer:
    admissionPlugins:
      - name: PodSecurity
        configuration:
          apiVersion: pod-security.admission.config.k8s.io/v1
          kind: PodSecurityConfiguration
          defaults:
            enforce: "restricted"
            audit: "restricted"
            warn: "restricted"
          exemptions:
            namespaces:
              - kube-system
              - longhorn-system
              - cattle-system
              - monitoring
              - velero
              - opencost
              - falco
```

### Step 5 — Worker Base Patch (`worker-patch.yaml`)

```yaml
# worker-patch.yaml
machine:
  network:
    extraHostEntries:
      - ip: 192.168.29.10
        aliases: ["k8s.prod.example.com"]
      - ip: 192.168.29.11
        aliases: ["cp01"]
      - ip: 192.168.29.12
        aliases: ["cp02"]
      - ip: 192.168.29.13
        aliases: ["cp03"]
      - ip: 192.168.29.14
        aliases: ["cp04"]
      - ip: 192.168.29.15
        aliases: ["cp05"]
      - ip: 192.168.29.21
        aliases: ["w01"]
      - ip: 192.168.29.22
        aliases: ["w02"]
      - ip: 192.168.29.23
        aliases: ["w03"]
      - ip: 192.168.29.24
        aliases: ["w04"]
      - ip: 192.168.29.25
        aliases: ["w05"]

  kubelet:
    nodeIP:
      validSubnets:
        - 192.168.29.0/24
    # Longhorn v1.12 requires bind mount for volume data
    extraMounts:
      - destination: /var/lib/longhorn
        type: bind
        source: /var/mnt/longhorn
        options: [bind, rshared, rw]

  # Dedicated 200 GB data disk for Longhorn
  disks:
    - device: /dev/sdb
      partitions:
        - mountpoint: /var/mnt/longhorn

  sysctls:
    vm.max_map_count: "262144"      # Required by OpenSearch/Elasticsearch tenants
    fs.inotify.max_user_instances: "8192"
    fs.inotify.max_user_watches: "524288"

  # Longhorn v1.12 v2 Data Engine kernel modules
  kernel:
    modules:
      - name: iscsi_tcp
      - name: dm_crypt
      - name: nvme_tcp

cluster:
  network:
    dnsDomain: cluster.local
```

### Step 6 — Generate Per-Node Configurations

#### Control Plane Nodes

```bash
# Helper function for CP nodes
gen_cp() {
  local NAME=$1 IP=$2
  talosctl machineconfig patch cluster-configs/controlplane.yaml \
    --patch @cp-patch.yaml \
    --output cluster-configs/${NAME}-base.yaml

  talosctl machineconfig patch cluster-configs/${NAME}-base.yaml \
    --patch "[
      {\"op\":\"replace\",\"path\":\"/machine/network/interfaces\",\"value\":[
        {\"interface\":\"ens192\",\"dhcp\":false,
         \"addresses\":[\"${IP}/24\"],
         \"routes\":[{\"network\":\"0.0.0.0/0\",\"gateway\":\"192.168.29.1\"}],
         \"vip\":{\"ip\":\"192.168.29.10\"}}
      ]},
      {\"op\":\"add\",\"path\":\"/machine/network/hostname\",\"value\":\"${NAME}\"}
    ]" \
    --output cluster-configs/${NAME}.yaml

  echo "Generated: cluster-configs/${NAME}.yaml"
}

gen_cp cp01 192.168.29.11
gen_cp cp02 192.168.29.12
gen_cp cp03 192.168.29.13
gen_cp cp04 192.168.29.14
gen_cp cp05 192.168.29.15
```

#### Worker Nodes

```bash
# Helper function for worker nodes
gen_worker() {
  local NAME=$1 IP=$2
  talosctl machineconfig patch cluster-configs/worker.yaml \
    --patch @worker-patch.yaml \
    --output cluster-configs/${NAME}-base.yaml

  talosctl machineconfig patch cluster-configs/${NAME}-base.yaml \
    --patch "[
      {\"op\":\"replace\",\"path\":\"/machine/network/interfaces\",\"value\":[
        {\"interface\":\"ens192\",\"dhcp\":false,
         \"addresses\":[\"${IP}/24\"],
         \"routes\":[{\"network\":\"0.0.0.0/0\",\"gateway\":\"192.168.29.1\"}]}
      ]},
      {\"op\":\"add\",\"path\":\"/machine/network/hostname\",\"value\":\"${NAME}\"}
    ]" \
    --output cluster-configs/${NAME}.yaml

  echo "Generated: cluster-configs/${NAME}.yaml"
}

gen_worker w01 192.168.29.21
gen_worker w02 192.168.29.22
gen_worker w03 192.168.29.23
gen_worker w04 192.168.29.24
gen_worker w05 192.168.29.25
```

### Step 7 — Apply Configurations

```bash
# Apply all control plane nodes
for NODE in cp01 cp02 cp03 cp04 cp05; do
  IP=$(grep -A1 "hostname.*${NODE}" cluster-configs/${NODE}.yaml | \
       grep -oP '192\.168\.29\.\d+' | head -1)
  echo "Applying ${NODE} (${IP})..."
  talosctl apply-config --insecure \
    --nodes 192.168.29.1$(echo $NODE | tr -dc '0-9') \
    --file cluster-configs/${NODE}.yaml
done

# Apply all worker nodes
for NODE in w01 w02 w03 w04 w05; do
  NUM=$(echo $NODE | tr -dc '0-9')
  IP="192.168.29.2${NUM}"
  echo "Applying ${NODE} (${IP})..."
  talosctl apply-config --insecure --nodes ${IP} --file cluster-configs/${NODE}.yaml
done

# Simplified (manual IPs):
talosctl apply-config --insecure --nodes 192.168.29.11 --file cluster-configs/cp01.yaml
talosctl apply-config --insecure --nodes 192.168.29.12 --file cluster-configs/cp02.yaml
talosctl apply-config --insecure --nodes 192.168.29.13 --file cluster-configs/cp03.yaml
talosctl apply-config --insecure --nodes 192.168.29.14 --file cluster-configs/cp04.yaml
talosctl apply-config --insecure --nodes 192.168.29.15 --file cluster-configs/cp05.yaml
talosctl apply-config --insecure --nodes 192.168.29.21 --file cluster-configs/w01.yaml
talosctl apply-config --insecure --nodes 192.168.29.22 --file cluster-configs/w02.yaml
talosctl apply-config --insecure --nodes 192.168.29.23 --file cluster-configs/w03.yaml
talosctl apply-config --insecure --nodes 192.168.29.24 --file cluster-configs/w04.yaml
talosctl apply-config --insecure --nodes 192.168.29.25 --file cluster-configs/w05.yaml

echo "Waiting 10 minutes for all nodes to initialize..."
sleep 600
```

### Step 8 — Bootstrap the Cluster

```bash
# Bootstrap is run EXACTLY ONCE on cp01 only.
# This initialises the etcd cluster — the other 4 CPs join automatically.
talosctl --talosconfig cluster-configs/talosconfig bootstrap \
  --endpoints 192.168.29.11 \
  --nodes 192.168.29.11

echo "Waiting 5 minutes for etcd and Kubernetes API to stabilise..."
sleep 300

# Verify VIP is reachable
ping -c 4 192.168.29.10

# Retrieve kubeconfig
talosctl --talosconfig cluster-configs/talosconfig kubeconfig ./cluster-configs/kubeconfig \
  --nodes 192.168.29.11 \
  --endpoints 192.168.29.11 \
  --force

export KUBECONFIG=./cluster-configs/kubeconfig
kubectl get nodes
# All 10 nodes show: NotReady (CNI not yet installed — expected)
```

### Step 9 — Verify Static Hostnames & etcd Health

```bash
# Verify all node hostnames
kubectl get nodes \
  -o custom-columns='NAME:.metadata.name,IP:.status.addresses[0].address,STATUS:.status.conditions[-1].type'

# Expected output:
# NAME   IP              STATUS
# cp01   192.168.29.11   Ready (after CNI)
# cp02   192.168.29.12   Ready
# cp03   192.168.29.13   Ready
# cp04   192.168.29.14   Ready
# cp05   192.168.29.15   Ready
# w01    192.168.29.21   Ready
# w02    192.168.29.22   Ready
# w03    192.168.29.23   Ready
# w04    192.168.29.24   Ready
# w05    192.168.29.25   Ready

# Verify etcd member list (all 5 CPs)
talosctl --talosconfig cluster-configs/talosconfig \
  --endpoints 192.168.29.10 \
  etcd members

# Check hostname on each node directly
for NODE in 192.168.29.11 192.168.29.12 192.168.29.13 192.168.29.14 192.168.29.15 \
            192.168.29.21 192.168.29.22 192.168.29.23 192.168.29.24 192.168.29.25; do
  echo -n "${NODE} → hostname: "
  talosctl --talosconfig cluster-configs/talosconfig \
    --nodes ${NODE} get hostname 2>/dev/null | awk 'NR==2{print $2}'
done
```

---

## 5. Network Components

### 5.1 — Install Calico CNI v3.31.5

```bash
# Install Tigera Operator
kubectl apply -f \
  https://raw.githubusercontent.com/projectcalico/calico/v3.31.5/manifests/tigera-operator.yaml

# Wait for operator
kubectl wait --for=condition=ready pod \
  -l name=tigera-operator -n tigera-operator --timeout=120s

# Apply custom Calico Installation (VXLAN mode — compatible with Talos)
kubectl apply -f - <<'EOF'
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
      - blockSize: 26
        cidr: 10.244.0.0/16
        encapsulation: VXLANCrossSubnet   # VXLAN — no BGP required
        natOutgoing: Enabled
        nodeSelector: all()
  # CNI type: Calico
  cni:
    type: Calico
EOF

# Wait for all Calico pods
kubectl wait --for=condition=ready pod \
  -l k8s-app=calico-node -n calico-system --timeout=300s
kubectl wait --for=condition=ready pod \
  -l k8s-app=calico-kube-controllers -n calico-system --timeout=120s

kubectl get nodes
# All nodes should now show: Ready
```

### 5.2 — Install MetalLB v0.15.3

```bash
# Install MetalLB
kubectl apply -f \
  https://raw.githubusercontent.com/metallb/metallb/v0.15.3/config/manifests/metallb-native.yaml

kubectl wait --for=condition=ready pod \
  -l app=metallb -n metallb-system --timeout=300s

# Configure IP pools
kubectl apply -f - <<'EOF'
---
# Admin pool: platform services only, manual assignment required
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: admin-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.29.100-192.168.29.109
  autoAssign: false
---
# Tenant ingress pool: Traefik LB
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: tenant-ingress-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.29.110-192.168.29.149
  autoAssign: false
---
# General pool: other LoadBalancer services
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: general-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.29.150-192.168.29.199
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
    - tenant-ingress-pool
    - general-pool
EOF

kubectl get ipaddresspool -n metallb-system
```

### 5.3 — Install Traefik Ingress v3.6.13

> Traefik replaces the deprecated Ingress NGINX (retired March 2026 from upstream Kubernetes). It supports IngressRoute CRDs, middleware, and TLS termination natively.

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update

kubectl create namespace traefik

# traefik-values.yaml
cat > traefik-values.yaml <<'EOF'
deployment:
  replicas: 3   # One per worker — use anti-affinity for HA

podAntiAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app.kubernetes.io/name: traefik
      topologyKey: kubernetes.io/hostname

service:
  type: LoadBalancer
  annotations:
    metallb.universe.tf/address-pool: tenant-ingress-pool
    metallb.universe.tf/loadBalancerIPs: 192.168.29.110

# Enable Kubernetes Ingress provider (for legacy Ingress resources)
providers:
  kubernetesCRD:
    enabled: true
    allowCrossNamespace: true
  kubernetesIngress:
    enabled: true

# TLS — cert-manager handles certs; Traefik handles termination
ingressClass:
  enabled: true
  isDefaultClass: true

# Expose dashboard (admin only — protected by middleware)
ingressRoute:
  dashboard:
    enabled: false   # Disabled by default; enable with auth middleware if needed

# Prometheus metrics
metrics:
  prometheus:
    entryPoint: metrics
    addServicesLabels: true
    addEntryPointsLabels: true

# Access logs (tenant isolation visibility)
logs:
  access:
    enabled: true
    fields:
      headers:
        defaultMode: keep
        names:
          X-Tenant-ID: keep
EOF

helm install traefik traefik/traefik \
  --namespace traefik \
  --version 35.0.0 \
  --values traefik-values.yaml \
  --wait

kubectl get svc -n traefik
# EXTERNAL-IP should be 192.168.29.110
```

### 5.4 — Install Metrics Server v0.8.1

```bash
kubectl apply -f \
  https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.8.1/components.yaml

# Patch for Talos: kubelet uses InternalIP, self-signed cert
kubectl patch deployment metrics-server -n kube-system --type='json' \
  -p='[
    {"op":"add","path":"/spec/template/spec/containers/0/args/-",
     "value":"--kubelet-preferred-address-types=InternalIP"},
    {"op":"add","path":"/spec/template/spec/containers/0/args/-",
     "value":"--kubelet-insecure-tls"}
  ]'

kubectl wait --for=condition=ready pod \
  -l k8s-app=metrics-server -n kube-system --timeout=300s

sleep 30
kubectl top nodes   # All 10 nodes should show CPU/Memory
```

---

## 6. Storage — Longhorn v1.12.0

### 6.1 — Prepare Worker Disks

```bash
# Create disk preparation patch
cat > longhorn-disk-patch.yaml <<'EOF'
machine:
  disks:
    - device: /dev/sdb
      partitions:
        - mountpoint: /var/mnt/longhorn
  kubelet:
    extraMounts:
      - destination: /var/lib/longhorn
        type: bind
        source: /var/mnt/longhorn
        options: [bind, rshared, rw]
EOF

# Apply disk patch to all worker nodes
for NODE_IP in 192.168.29.21 192.168.29.22 192.168.29.23 192.168.29.24 192.168.29.25; do
  echo "Preparing disk on ${NODE_IP}..."
  talosctl --talosconfig cluster-configs/talosconfig \
    patch machineconfig -n ${NODE_IP} -e ${NODE_IP} \
    --patch @longhorn-disk-patch.yaml
done

echo "Waiting 3 minutes for disk mounts..."
sleep 180
```

### 6.2 — Install Longhorn v1.12.0

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update

kubectl create namespace longhorn-system
kubectl label namespace longhorn-system \
  pod-security.kubernetes.io/enforce=privileged --overwrite

# longhorn-values.yaml
cat > longhorn-values.yaml <<'EOF'
defaultSettings:
  defaultDataPath: "/var/lib/longhorn"
  replicaCount: 3                     # 3 replicas across 5 workers — survives 2 worker failures
  storageOverProvisioningPercentage: 150
  storageMinimalAvailablePercentage: 25
  # V2 Data Engine (new in v1.12) — high performance SPDK-based
  v2DataEngine: true
  # Automatic cleanup of failed replicas
  autoDeletePodWhenVolumeDetachedUnexpectedly: true
  # Recurring job for snapshots (default)
  recurringJobMaxRetain: 2

persistence:
  defaultClass: true
  defaultClassReplicaCount: 3
  defaultFsType: xfs
  defaultDataLocality: best-effort    # Prefer local scheduling when possible

csi:
  attacherReplicaCount: 3
  provisionerReplicaCount: 3
  resizerReplicaCount: 3
  snapshotterReplicaCount: 3          # Required for Velero CSI snapshots

ingress:
  enabled: false    # Exposed via MetalLB LoadBalancer below
EOF

helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --version 1.12.0 \
  --values longhorn-values.yaml \
  --wait \
  --timeout 10m

kubectl wait --for=condition=ready pod \
  -l app=longhorn-manager -n longhorn-system --timeout=600s

kubectl get storageclass
# longhorn (default)  driver.longhorn.io  ...
```

### 6.3 — Create VolumeSnapshotClass for Velero

```bash
# Required for Velero CSI snapshot integration
kubectl apply -f - <<'EOF'
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: longhorn-vsc
  labels:
    velero.io/csi-volumesnapshot-class: "true"   # Velero auto-discovers this label
driver: driver.longhorn.io
deletionPolicy: Delete
parameters:
  type: snap
EOF
```

### 6.4 — Expose Longhorn UI (Admin Only)

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: longhorn-frontend-lb
  namespace: longhorn-system
  annotations:
    metallb.universe.tf/address-pool: admin-pool
spec:
  type: LoadBalancer
  loadBalancerIP: 192.168.29.100
  selector:
    app: longhorn-ui
  ports:
    - name: http
      port: 80
      targetPort: 8000
EOF

echo "Longhorn UI: http://192.168.29.100 (admin IPs only)"
```

### 6.5 — Verify Storage

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: storage-test-pvc
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: longhorn
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: storage-test-pod
spec:
  containers:
    - name: test
      image: alpine:3.21
      command: [sh, -c, "echo 'Longhorn v1.12.0 OK' > /data/test.txt && cat /data/test.txt && sleep 60"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: storage-test-pvc
EOF

sleep 90
kubectl logs storage-test-pod
# Expected: Longhorn v1.12.0 OK

# Cleanup
kubectl delete pod storage-test-pod
kubectl delete pvc storage-test-pvc
```

---

## 7. cert-manager & TLS

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.20.2 \
  --set crds.enabled=true \
  --wait

# ClusterIssuer — Let's Encrypt production
kubectl apply -f - <<'EOF'
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            ingressClassName: traefik
EOF

# ClusterIssuer — Let's Encrypt staging (for testing)
kubectl apply -f - <<'EOF'
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-staging-key
    solvers:
      - http01:
          ingress:
            ingressClassName: traefik
EOF

kubectl get clusterissuer
```

---

## 8. Rancher Installation

### 8.1 — Install Rancher v2.14.2

```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update

kubectl create namespace cattle-system

# rancher-values.yaml
cat > rancher-values.yaml <<'EOF'
hostname: rancher.prod.example.com

replicas: 3       # 3 Rancher pods for HA

ingress:
  tls:
    source: letsEncrypt
  extraAnnotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod

letsEncrypt:
  email: admin@example.com
  ingress:
    class: traefik

resources:
  requests:
    cpu: 250m
    memory: 512Mi
  limits:
    cpu: 2
    memory: 2Gi

# Anti-affinity: spread Rancher pods across different workers
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: rancher
        topologyKey: kubernetes.io/hostname

# Audit logging for compliance
auditLog:
  level: 1
  destination: sidecar
  maxAge: 30
  maxBackup: 10
  maxSize: 100

# Private registry (optional — remove if using public registry)
# privateCA: true
# systemDefaultRegistry: registry.example.com
EOF

helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --version 2.14.2 \
  --values rancher-values.yaml \
  --wait \
  --timeout 15m

# Watch rollout
kubectl rollout status deployment/rancher -n cattle-system

echo ""
echo "Rancher UI: https://rancher.prod.example.com"
echo ""
echo "Get bootstrap password:"
kubectl get secret --namespace cattle-system bootstrap-secret \
  -o go-template='{{.data.bootstrapPassword|base64decode}}{{"\n"}}'
```

### 8.2 — Post-Install Rancher Configuration

```bash
# Wait for Rancher to become fully operational
kubectl wait --for=condition=ready pod \
  -l app=rancher -n cattle-system --timeout=600s

# Get the Rancher bootstrap password
RANCHER_PASSWORD=$(kubectl get secret --namespace cattle-system bootstrap-secret \
  -o go-template='{{.data.bootstrapPassword|base64decode}}')

# Set Rancher URL via API (after first login)
RANCHER_URL="https://rancher.prod.example.com"
RANCHER_TOKEN=""   # Populate after first login via UI

# Verify Rancher is healthy
curl -sk "${RANCHER_URL}/healthz"
# Expected: ok
```

---

## 9. Multi-Tenancy Design

### Architecture Principles

```
Rancher Project  ══════════════════════════════════════╗
│ Name: "Tenant-Alice"                                  ║
│ ID:   p-xxxx                                          ║
│                                                       ║
│  Namespace: tenant-alice-ns                           ║
│  ┌──────────────────────────────────────────────────┐ ║
│  │ ResourceQuota:  CPU 4, Mem 8Gi, Storage 50Gi     │ ║
│  │ LimitRange:     Container defaults + max limits   │ ║
│  │ NetworkPolicy:  Deny-all ingress except Traefik   │ ║
│  │ PodSecurityAdm: restricted profile                │ ║
│  │ Velero Label:   backup.velero.io/tenant=alice     │ ║
│  │                                                   │ ║
│  │  Workloads:  WordPress + MariaDB                  │ ║
│  │  Ingress:    alice.tenant.prod.example.com        │ ║
│  │  TLS:        cert-manager / Let's Encrypt         │ ║
│  └──────────────────────────────────────────────────┘ ║
│                                                       ║
│  ProjectRoleTemplateBinding:                          ║
│    alice@example.com → project-owner                  ║
╚══════════════════════════════════════════════════════╝
```

### Resource Quota Tiers

```yaml
# Three SaaS tiers — applied as ResourceQuota per namespace

# TIER: Starter ($10/mo)
starter_quota:
  requests.cpu: "1"
  limits.cpu: "2"
  requests.memory: 2Gi
  limits.memory: 4Gi
  requests.storage: 20Gi
  persistentvolumeclaims: "3"
  pods: "10"
  services: "5"
  services.loadbalancers: "0"

# TIER: Professional ($50/mo)
professional_quota:
  requests.cpu: "4"
  limits.cpu: "8"
  requests.memory: 8Gi
  limits.memory: 16Gi
  requests.storage: 100Gi
  persistentvolumeclaims: "10"
  pods: "50"
  services: "20"
  services.loadbalancers: "1"

# TIER: Enterprise ($200/mo)
enterprise_quota:
  requests.cpu: "16"
  limits.cpu: "32"
  requests.memory: 32Gi
  limits.memory: 64Gi
  requests.storage: 500Gi
  persistentvolumeclaims: "50"
  pods: "200"
  services: "100"
  services.loadbalancers: "5"
```

---

## 10. Rancher API Automation — Tenant Onboarding

Complete automated tenant onboarding script. Accepts a tenant name, email, and tier.

### 10.1 — Rancher API Configuration

```bash
# Set these once — store in CI/CD secrets, not in code
export RANCHER_URL="https://rancher.prod.example.com"
export RANCHER_TOKEN="token-xxxxx:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
# Create API token: Rancher UI → User Avatar → API & Keys → Add Key

# Cluster ID (the "local" cluster managed by Rancher)
CLUSTER_ID=$(curl -sk -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  "${RANCHER_URL}/v3/clusters?name=local" | jq -r '.data[0].id')
echo "Cluster ID: ${CLUSTER_ID}"
```

### 10.2 — Full Tenant Onboarding Script (`onboard-tenant.sh`)

```bash
#!/usr/bin/env bash
# onboard-tenant.sh — Provision a complete SaaS tenant
# Usage: ./onboard-tenant.sh <tenant-name> <tenant-email> <tier>
# Tiers: starter | professional | enterprise
# Example: ./onboard-tenant.sh alice alice@example.com professional

set -euo pipefail

TENANT_NAME="${1:?Usage: $0 <tenant-name> <tenant-email> <tier>}"
TENANT_EMAIL="${2:?Provide tenant email}"
TENANT_TIER="${3:-starter}"

RANCHER_URL="${RANCHER_URL:?Set RANCHER_URL}"
RANCHER_TOKEN="${RANCHER_TOKEN:?Set RANCHER_TOKEN}"

NS_NAME="tenant-${TENANT_NAME}-ns"
PROJECT_NAME="Tenant-${TENANT_NAME}"

echo "═══════════════════════════════════════════════════"
echo " Onboarding Tenant: ${TENANT_NAME}"
echo " Email:  ${TENANT_EMAIL}"
echo " Tier:   ${TENANT_TIER}"
echo " NS:     ${NS_NAME}"
echo "═══════════════════════════════════════════════════"

# ── Resource quota values by tier ──────────────────────
case "${TENANT_TIER}" in
  starter)
    CPU_REQ="1"; CPU_LIM="2"
    MEM_REQ="2Gi"; MEM_LIM="4Gi"
    STORAGE="20Gi"; PVCS="3"; PODS="10"
    ;;
  professional)
    CPU_REQ="4"; CPU_LIM="8"
    MEM_REQ="8Gi"; MEM_LIM="16Gi"
    STORAGE="100Gi"; PVCS="10"; PODS="50"
    ;;
  enterprise)
    CPU_REQ="16"; CPU_LIM="32"
    MEM_REQ="32Gi"; MEM_LIM="64Gi"
    STORAGE="500Gi"; PVCS="50"; PODS="200"
    ;;
  *)
    echo "ERROR: Unknown tier '${TENANT_TIER}'. Use starter|professional|enterprise"
    exit 1
    ;;
esac

# ── Step 1: Get Cluster ID ──────────────────────────────
echo "[1/8] Getting cluster ID..."
CLUSTER_ID=$(curl -sk -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  "${RANCHER_URL}/v3/clusters?name=local" | jq -r '.data[0].id')
echo "     Cluster ID: ${CLUSTER_ID}"

# ── Step 2: Create Rancher Project ─────────────────────
echo "[2/8] Creating Rancher project '${PROJECT_NAME}'..."
PROJECT_RESPONSE=$(curl -sk -X POST \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  -H "Content-Type: application/json" \
  "${RANCHER_URL}/v3/projects" \
  -d "{
    \"clusterId\": \"${CLUSTER_ID}\",
    \"name\": \"${PROJECT_NAME}\",
    \"description\": \"SaaS tenant project for ${TENANT_EMAIL} (${TENANT_TIER} tier)\",
    \"resourceQuota\": {
      \"limit\": {
        \"requestsCpu\": \"${CPU_REQ}\",
        \"limitsCpu\": \"${CPU_LIM}\",
        \"requestsMemory\": \"${MEM_REQ}\",
        \"limitsMemory\": \"${MEM_LIM}\",
        \"requestsStorage\": \"${STORAGE}\",
        \"persistentVolumeClaims\": \"${PVCS}\",
        \"pods\": \"${PODS}\"
      }
    },
    \"namespaceDefaultResourceQuota\": {
      \"limit\": {
        \"requestsCpu\": \"${CPU_REQ}\",
        \"limitsCpu\": \"${CPU_LIM}\",
        \"requestsMemory\": \"${MEM_REQ}\",
        \"limitsMemory\": \"${MEM_LIM}\",
        \"requestsStorage\": \"${STORAGE}\",
        \"persistentVolumeClaims\": \"${PVCS}\",
        \"pods\": \"${PODS}\"
      }
    },
    \"labels\": {
      \"tenant\": \"${TENANT_NAME}\",
      \"tier\": \"${TENANT_TIER}\"
    }
  }")

PROJECT_ID=$(echo "${PROJECT_RESPONSE}" | jq -r '.id')
echo "     Project ID: ${PROJECT_ID}"

# ── Step 3: Create Namespace inside Project ─────────────
echo "[3/8] Creating namespace '${NS_NAME}'..."
curl -sk -X POST \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  -H "Content-Type: application/json" \
  "${RANCHER_URL}/v3/cluster/${CLUSTER_ID}/namespaces" \
  -d "{
    \"name\": \"${NS_NAME}\",
    \"projectId\": \"${PROJECT_ID}\",
    \"annotations\": {
      \"tenant\": \"${TENANT_NAME}\",
      \"tier\": \"${TENANT_TIER}\",
      \"contact\": \"${TENANT_EMAIL}\"
    },
    \"labels\": {
      \"tenant\": \"${TENANT_NAME}\",
      \"tier\": \"${TENANT_TIER}\",
      \"pod-security.kubernetes.io/enforce\": \"restricted\"
    }
  }" > /dev/null
echo "     Namespace created."

# ── Step 4: Apply ResourceQuota via kubectl ─────────────
echo "[4/8] Applying ResourceQuota and LimitRange..."
kubectl apply -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-quota
  namespace: ${NS_NAME}
  labels:
    tenant: "${TENANT_NAME}"
    tier: "${TENANT_TIER}"
spec:
  hard:
    requests.cpu: "${CPU_REQ}"
    limits.cpu: "${CPU_LIM}"
    requests.memory: "${MEM_REQ}"
    limits.memory: "${MEM_LIM}"
    requests.storage: "${STORAGE}"
    persistentvolumeclaims: "${PVCS}"
    pods: "${PODS}"
    services: "20"
    secrets: "50"
    configmaps: "50"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: tenant-limitrange
  namespace: ${NS_NAME}
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
        cpu: "4"
        memory: "8Gi"
      min:
        cpu: "50m"
        memory: "64Mi"
    - type: PersistentVolumeClaim
      max:
        storage: "50Gi"
      min:
        storage: "1Gi"
EOF

# ── Step 5: Apply Network Policies ─────────────────────
echo "[5/8] Applying NetworkPolicies (tenant isolation)..."
kubectl apply -f - <<EOF
# Default deny all ingress and egress — whitelist only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: ${NS_NAME}
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
# Allow ingress from Traefik only (all tenants use the same Traefik instance)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-traefik-ingress
  namespace: ${NS_NAME}
spec:
  podSelector: {}
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: traefik
  policyTypes: [Ingress]
---
# Allow egress to DNS (kube-dns)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: ${NS_NAME}
spec:
  podSelector: {}
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
  policyTypes: [Egress]
---
# Allow intra-namespace communication (app ↔ database)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-intra-namespace
  namespace: ${NS_NAME}
spec:
  podSelector: {}
  ingress:
    - from:
        - podSelector: {}
  egress:
    - to:
        - podSelector: {}
  policyTypes: [Ingress, Egress]
---
# Allow egress to internet (for WordPress updates, plugins)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-internet-egress
  namespace: ${NS_NAME}
spec:
  podSelector: {}
  egress:
    - ports:
        - port: 80
          protocol: TCP
        - port: 443
          protocol: TCP
  policyTypes: [Egress]
EOF

# ── Step 6: Create Rancher User & Project Role Binding ──
echo "[6/8] Creating Rancher user and RBAC binding..."
# Create user (Rancher local auth)
USER_RESPONSE=$(curl -sk -X POST \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  -H "Content-Type: application/json" \
  "${RANCHER_URL}/v3/users" \
  -d "{
    \"username\": \"${TENANT_NAME}\",
    \"password\": \"ChangeMeNow123!\",
    \"name\": \"${TENANT_NAME^}\",
    \"description\": \"SaaS tenant user\",
    \"mustChangePassword\": true
  }")
USER_ID=$(echo "${USER_RESPONSE}" | jq -r '.id')
echo "     User ID: ${USER_ID}"

# Bind user to project as project-owner
curl -sk -X POST \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  -H "Content-Type: application/json" \
  "${RANCHER_URL}/v3/projectroletemplatebindings" \
  -d "{
    \"projectId\": \"${PROJECT_ID}\",
    \"userId\": \"${USER_ID}\",
    \"roleTemplateId\": \"project-owner\"
  }" > /dev/null
echo "     RBAC binding created: ${TENANT_NAME} → project-owner"

# ── Step 7: Create Velero Backup Schedule ──────────────
echo "[7/8] Creating Velero backup schedule..."
velero schedule create "backup-${TENANT_NAME}" \
  --schedule="0 3 * * *" \
  --ttl 720h \
  --include-namespaces "${NS_NAME}" \
  --snapshot-volumes=true \
  --volume-snapshot-locations longhorn-vsl \
  --labels "tenant=${TENANT_NAME},tier=${TENANT_TIER}" \
  --selector "tenant=${TENANT_NAME}" \
  || echo "     Note: Velero schedule may already exist or Velero not yet installed"

# ── Step 8: Summary ───────────────────────────────────
echo ""
echo "═══════════════════════════════════════════════════"
echo " Tenant Onboarding Complete!"
echo "═══════════════════════════════════════════════════"
echo " Project ID:    ${PROJECT_ID}"
echo " Namespace:     ${NS_NAME}"
echo " User:          ${TENANT_NAME} / ${TENANT_EMAIL}"
echo " Tier:          ${TENANT_TIER}"
echo " CPU Limit:     ${CPU_LIM} cores"
echo " Memory Limit:  ${MEM_LIM}"
echo " Storage:       ${STORAGE}"
echo " Backup Sched:  Daily 03:00 UTC, 30-day retention"
echo ""
echo " Next step: Deploy app →"
echo "   ./deploy-wordpress.sh ${TENANT_NAME} ${NS_NAME}"
echo "═══════════════════════════════════════════════════"
```

```bash
chmod +x onboard-tenant.sh

# Onboard tenants
./onboard-tenant.sh alice     alice@example.com    professional
./onboard-tenant.sh bob       bob@example.com      starter
./onboard-tenant.sh acmecorp  ops@acmecorp.com     enterprise
```

---

## 11. SaaS App Deployment — WordPress per Tenant

### 11.1 — WordPress Helm Deployment Script (`deploy-wordpress.sh`)

```bash
#!/usr/bin/env bash
# deploy-wordpress.sh — Deploy WordPress+MariaDB to a tenant namespace
# Usage: ./deploy-wordpress.sh <tenant-name> <namespace>
# Example: ./deploy-wordpress.sh alice tenant-alice-ns

set -euo pipefail

TENANT_NAME="${1:?Usage: $0 <tenant-name> <namespace>}"
NAMESPACE="${2:-tenant-${TENANT_NAME}-ns}"
DOMAIN="${TENANT_NAME}.tenant.prod.example.com"
WP_RELEASE="wordpress-${TENANT_NAME}"

echo "Deploying WordPress for tenant: ${TENANT_NAME}"
echo "Namespace: ${NAMESPACE}"
echo "Domain:    ${DOMAIN}"

helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Generate secrets (store these securely in Vault/Sealed Secrets in production)
WP_ADMIN_PASS=$(openssl rand -base64 16)
DB_ROOT_PASS=$(openssl rand -base64 16)
DB_PASS=$(openssl rand -base64 16)

cat > /tmp/wordpress-${TENANT_NAME}.yaml <<EOF
# WordPress configuration for tenant: ${TENANT_NAME}
wordpressUsername: admin
wordpressPassword: "${WP_ADMIN_PASS}"
wordpressEmail: "webmaster@${DOMAIN}"
wordpressFirstName: "Admin"
wordpressLastName: "${TENANT_NAME^}"
wordpressBlogName: "${TENANT_NAME^}'s Site"

# Resources — constrained to not exhaust namespace quota
resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: "1"
    memory: 1Gi

# Persistence — Longhorn storage
persistence:
  enabled: true
  storageClass: longhorn
  accessMode: ReadWriteOnce
  size: 10Gi

# MariaDB (bundled)
mariadb:
  enabled: true
  auth:
    rootPassword: "${DB_ROOT_PASS}"
    database: "wordpress_${TENANT_NAME}"
    username: "wp_${TENANT_NAME}"
    password: "${DB_PASS}"
  primary:
    persistence:
      enabled: true
      storageClass: longhorn
      size: 5Gi
    resources:
      requests:
        cpu: 100m
        memory: 256Mi
      limits:
        cpu: "1"
        memory: 1Gi

# Service: ClusterIP only — ingress via Traefik
service:
  type: ClusterIP

# Ingress — Traefik + cert-manager
ingress:
  enabled: true
  ingressClassName: traefik
  hostname: ${DOMAIN}
  tls: true
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    traefik.ingress.kubernetes.io/router.middlewares: "${NAMESPACE}-security-headers@kubernetescrd"

# Pod security context (restricted profile)
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1001
  fsGroup: 1001
  seccompProfile:
    type: RuntimeDefault

containerSecurityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: false   # WordPress needs to write to /var/www
  capabilities:
    drop: [ALL]

# Labels for Velero backup targeting
commonLabels:
  tenant: "${TENANT_NAME}"
  backup.velero.io/backup-volumes: "data"

# Pod disruption budget
pdb:
  create: true
  minAvailable: 1

# Affinity: spread across workers
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: wordpress
          topologyKey: kubernetes.io/hostname

# Liveness / readiness probes
startupProbe:
  enabled: true
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 30
EOF

# Deploy Traefik security headers middleware first
kubectl apply -f - <<MIDDLEWARE
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: security-headers
  namespace: ${NAMESPACE}
spec:
  headers:
    browserXssFilter: true
    contentTypeNosniff: true
    frameDeny: true
    stsSeconds: 31536000
    stsIncludeSubdomains: true
    customResponseHeaders:
      X-Tenant-ID: "${TENANT_NAME}"
MIDDLEWARE

# Deploy WordPress
helm upgrade --install "${WP_RELEASE}" bitnami/wordpress \
  --namespace "${NAMESPACE}" \
  --values /tmp/wordpress-${TENANT_NAME}.yaml \
  --wait \
  --timeout 10m \
  --atomic   # Rollback on failure

echo ""
echo "WordPress deployed for tenant: ${TENANT_NAME}"
echo "URL:            https://${DOMAIN}"
echo "Admin:          https://${DOMAIN}/wp-admin"
echo "Admin Password: ${WP_ADMIN_PASS}"
echo "(Store this password securely — it will not be shown again)"

# Clean up temp values (contains passwords)
shred -u /tmp/wordpress-${TENANT_NAME}.yaml
```

```bash
chmod +x deploy-wordpress.sh

./deploy-wordpress.sh alice tenant-alice-ns
./deploy-wordpress.sh bob   tenant-bob-ns
```

### 11.2 — Verify WordPress Deployment

```bash
TENANT=alice
NS="tenant-${TENANT}-ns"

# Check pods
kubectl get pods -n ${NS}

# Check ingress
kubectl get ingress -n ${NS}

# Check PVCs
kubectl get pvc -n ${NS}

# Test HTTP response
curl -sk -o /dev/null -w "%{http_code}" https://alice.tenant.prod.example.com
# Expected: 200 or 302 (redirect to wp-admin on fresh install)
```

---

## 12. Velero — Backup & Restore per Tenant

### 12.1 — Deploy MinIO (On-Cluster Object Storage)

MinIO provides S3-compatible storage for Velero backups. For production, use external S3 (AWS, Ceph, or standalone MinIO cluster).

```bash
helm repo add minio https://charts.min.io/
helm repo update

kubectl create namespace minio-system

cat > minio-values.yaml <<'EOF'
mode: standalone     # Use mode: distributed for production (4+ nodes)
replicas: 1

rootUser: minio-admin
rootPassword: "ChangeMe-Minio-Secret-2026!"

persistence:
  enabled: true
  storageClass: longhorn
  size: 500Gi

resources:
  requests:
    cpu: 500m
    memory: 1Gi
  limits:
    cpu: 2
    memory: 4Gi

service:
  type: ClusterIP

# Create velero bucket automatically
buckets:
  - name: velero-backups
    policy: none
    purge: false

# Metrics
metrics:
  serviceMonitor:
    enabled: true
    namespace: monitoring
EOF

helm install minio minio/minio \
  --namespace minio-system \
  --values minio-values.yaml \
  --wait

kubectl get pods -n minio-system
```

### 12.2 — Install Velero v1.18.0 with CSI + Longhorn Plugin

```bash
# Create Velero credentials file
cat > /tmp/velero-credentials <<'EOF'
[default]
aws_access_key_id=minio-admin
aws_secret_access_key=ChangeMe-Minio-Secret-2026!
EOF

# Get MinIO service IP
MINIO_SVC="http://minio.minio-system.svc.cluster.local:9000"

# Install Velero with CSI support (for Longhorn v2 snapshot integration)
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
helm repo update

cat > velero-values.yaml <<'EOF'
configuration:
  backupStorageLocation:
    - name: default
      provider: aws
      bucket: velero-backups
      default: true
      config:
        region: minio
        s3ForcePathStyle: "true"
        s3Url: "http://minio.minio-system.svc.cluster.local:9000"
        publicUrl: "http://minio.minio-system.svc.cluster.local:9000"

  volumeSnapshotLocation:
    - name: longhorn-vsl
      provider: csi
      config: {}

  # Enable CSI volume snapshots (Longhorn v1.12 v2 engine)
  features: EnableCSI

credentials:
  secretContents:
    cloud: |
      [default]
      aws_access_key_id=minio-admin
      aws_secret_access_key=ChangeMe-Minio-Secret-2026!

initContainers:
  - name: velero-plugin-for-aws
    image: velero/velero-plugin-for-aws:v1.12.0
    volumeMounts:
      - mountPath: /target
        name: plugins
  - name: velero-plugin-for-csi
    image: velero/velero-plugin-for-csi:v0.8.0
    volumeMounts:
      - mountPath: /target
        name: plugins

resources:
  requests:
    cpu: 500m
    memory: 128Mi
  limits:
    cpu: "1"
    memory: 512Mi

schedules: {}   # Per-tenant schedules created by onboard script

# Prometheus metrics
metrics:
  enabled: true
  serviceMonitor:
    enabled: true
    namespace: monitoring
EOF

helm install velero vmware-tanzu/velero \
  --namespace velero \
  --create-namespace \
  --version 9.0.0 \
  --values velero-values.yaml \
  --wait

kubectl get pods -n velero
velero backup-location get
# Expected: default  Available
```

### 12.3 — Backup Operations

```bash
# ── List all backup schedules ──────────────────────────
velero schedule get

# Expected output:
# NAME              STATUS    SCHEDULE     BACKUP TTL   LAST BACKUP   SELECTOR
# backup-alice      Enabled   0 3 * * *    720h0m0s    ...
# backup-bob        Enabled   0 3 * * *    720h0m0s    ...
# backup-acmecorp   Enabled   0 3 * * *    720h0m0s    ...

# ── Trigger manual backup (any tenant) ─────────────────
velero backup create "manual-alice-$(date +%Y%m%d-%H%M%S)" \
  --include-namespaces tenant-alice-ns \
  --snapshot-volumes=true \
  --volume-snapshot-locations longhorn-vsl \
  --labels "tenant=alice,type=manual" \
  --wait

# ── List backups for a tenant ───────────────────────────
velero backup get --selector "tenant=alice"

# ── Inspect a backup ───────────────────────────────────
velero backup describe manual-alice-20260611-120000 --details

# ── Check backup logs ──────────────────────────────────
velero backup logs manual-alice-20260611-120000
```

### 12.4 — Restore Operations

```bash
# ── Restore a tenant namespace to a new namespace ──────
BACKUP_NAME="backup-alice-20260611000000"
RESTORE_NS="tenant-alice-restored"

velero restore create "restore-alice-$(date +%Y%m%d-%H%M%S)" \
  --from-backup "${BACKUP_NAME}" \
  --include-namespaces tenant-alice-ns \
  --namespace-mappings "tenant-alice-ns:${RESTORE_NS}" \
  --restore-volumes=true \
  --wait

# ── Monitor restore progress ────────────────────────────
velero restore describe "restore-alice-20260611-130000" --details

# ── Restore to SAME namespace (disaster recovery) ──────
# First: delete existing namespace (DESTRUCTIVE — use with care)
# kubectl delete namespace tenant-alice-ns
# Then restore:
velero restore create "dr-restore-alice-$(date +%Y%m%d)" \
  --from-backup "${BACKUP_NAME}" \
  --include-namespaces tenant-alice-ns \
  --restore-volumes=true \
  --wait

# ── Verify restored WordPress ───────────────────────────
kubectl get pods -n ${RESTORE_NS}
kubectl get pvc  -n ${RESTORE_NS}
curl -sk -o /dev/null -w "%{http_code}" https://alice.tenant.prod.example.com
```

### 12.5 — Backup Schedule Management

```bash
# ── Create premium backup schedule (enterprise tier: hourly) ──
velero schedule create "backup-acmecorp-hourly" \
  --schedule="0 * * * *" \
  --ttl 168h \
  --include-namespaces tenant-acmecorp-ns \
  --snapshot-volumes=true \
  --labels "tenant=acmecorp,tier=enterprise"

# ── Disable a schedule (tenant suspended) ──────────────
velero schedule delete backup-bob --confirm

# ── List all backups across all tenants ────────────────
velero backup get

# ── Get backup size and status summary ─────────────────
velero backup get -o json | jq -r '
  .items[] | {
    name: .metadata.name,
    tenant: .metadata.labels.tenant,
    status: .status.phase,
    size: .status.progress.totalBytes,
    created: .metadata.creationTimestamp
  } | @base64' | while read b; do
  echo $b | base64 -d | jq .
done
```

---

## 13. Monitoring & Observability

### 13.1 — Install kube-prometheus-stack v86.2.2

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

kubectl create namespace monitoring

cat > kube-prometheus-values.yaml <<'EOF'
# ── Prometheus ───────────────────────────────────────────
prometheus:
  prometheusSpec:
    replicas: 2         # HA Prometheus with Longhorn-backed storage
    replicaExternalLabelName: __replica__
    retention: 30d
    retentionSize: "80GB"
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: longhorn
          accessModes: [ReadWriteOnce]
          resources:
            requests:
              storage: 100Gi
    resources:
      requests:
        cpu: 500m
        memory: 2Gi
      limits:
        cpu: 2
        memory: 8Gi
    # Scrape all namespaces
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    ruleSelectorNilUsesHelmValues: false
    # Additional scrape config for Longhorn, Rancher, Velero
    additionalScrapeConfigs:
      - job_name: 'longhorn-manager'
        static_configs:
          - targets: ['longhorn-backend.longhorn-system.svc.cluster.local:9500']
      - job_name: 'velero'
        static_configs:
          - targets: ['velero.velero.svc.cluster.local:8085']
      - job_name: 'traefik'
        static_configs:
          - targets: ['traefik.traefik.svc.cluster.local:9100']

# ── Grafana ──────────────────────────────────────────────
grafana:
  enabled: true
  replicas: 2
  adminPassword: "GrafanaAdmin-2026!"

  persistence:
    enabled: true
    storageClassName: longhorn
    size: 10Gi

  service:
    type: LoadBalancer
    annotations:
      metallb.universe.tf/address-pool: admin-pool
      metallb.universe.tf/loadBalancerIPs: "192.168.29.103"

  ingress:
    enabled: true
    ingressClassName: traefik
    hosts: [grafana.prod.example.com]
    tls:
      - secretName: grafana-tls
        hosts: [grafana.prod.example.com]
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod

  # Pre-install useful dashboards
  dashboardProviders:
    dashboardproviders.yaml:
      apiVersion: 1
      providers:
        - name: default
          orgId: 1
          folder: ''
          type: file
          disableDeletion: false
          editable: true
          options:
            path: /var/lib/grafana/dashboards/default

  dashboards:
    default:
      kubernetes-cluster:
        gnetId: 7249
        revision: 1
        datasource: Prometheus
      namespace-by-pod:
        gnetId: 11453
        revision: 1
        datasource: Prometheus
      longhorn:
        gnetId: 13032
        revision: 6
        datasource: Prometheus
      traefik:
        gnetId: 17347
        revision: 9
        datasource: Prometheus
      velero:
        gnetId: 15470
        revision: 1
        datasource: Prometheus

  resources:
    requests:
      cpu: 250m
      memory: 256Mi
    limits:
      cpu: 1
      memory: 1Gi

# ── Alertmanager ─────────────────────────────────────────
alertmanager:
  enabled: true
  alertmanagerSpec:
    replicas: 2
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: longhorn
          resources:
            requests:
              storage: 5Gi

# ── Node Exporter ────────────────────────────────────────
nodeExporter:
  enabled: true

# ── kube-state-metrics ───────────────────────────────────
kubeStateMetrics:
  enabled: true

# ── Additional PrometheusRules ───────────────────────────
additionalPrometheusRulesMap:
  tenant-alerts:
    groups:
      - name: tenant.resource.quota
        rules:
          - alert: NamespaceCPUQuotaHigh
            expr: |
              (kube_resourcequota{resource="limits.cpu",type="used"}
               / kube_resourcequota{resource="limits.cpu",type="hard"}) > 0.80
            for: 5m
            labels:
              severity: warning
            annotations:
              summary: "Namespace {{ $labels.namespace }} CPU quota >80%"
              description: "Tenant may be near their CPU limit. Consider tier upgrade."

          - alert: NamespaceMemoryQuotaHigh
            expr: |
              (kube_resourcequota{resource="limits.memory",type="used"}
               / kube_resourcequota{resource="limits.memory",type="hard"}) > 0.80
            for: 5m
            labels:
              severity: warning
            annotations:
              summary: "Namespace {{ $labels.namespace }} memory quota >80%"

          - alert: TenantPodCrashLooping
            expr: |
              increase(kube_pod_container_status_restarts_total{
                namespace=~"tenant-.*"}[10m]) > 3
            for: 5m
            labels:
              severity: critical
            annotations:
              summary: "Pod {{ $labels.pod }} in tenant namespace {{ $labels.namespace }} is crash-looping"

          - alert: VeleroBackupFailed
            expr: |
              velero_backup_failure_total > 0
            for: 5m
            labels:
              severity: critical
            annotations:
              summary: "Velero backup failed for {{ $labels.schedule }}"

          - alert: LonghornVolumeUnhealthy
            expr: |
              longhorn_volume_robustness == 3
            for: 5m
            labels:
              severity: critical
            annotations:
              summary: "Longhorn volume {{ $labels.volume }} is degraded/faulted"
EOF

helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --version 86.2.2 \
  --values kube-prometheus-values.yaml \
  --wait \
  --timeout 15m

kubectl get pods -n monitoring
echo "Grafana UI: https://grafana.prod.example.com"
```

### 13.2 — Per-Tenant Grafana Dashboard

```bash
# Create a per-tenant dashboard ConfigMap
cat > tenant-dashboard.json <<'DASHBOARD'
{
  "title": "Tenant Namespace Overview",
  "uid": "tenant-overview",
  "panels": [
    {
      "title": "CPU Usage by Namespace",
      "type": "timeseries",
      "targets": [{
        "expr": "sum(rate(container_cpu_usage_seconds_total{namespace=~\"tenant-.*\"}[5m])) by (namespace)"
      }]
    },
    {
      "title": "Memory Usage by Namespace",
      "type": "timeseries",
      "targets": [{
        "expr": "sum(container_memory_working_set_bytes{namespace=~\"tenant-.*\"}) by (namespace)"
      }]
    },
    {
      "title": "Pod Count by Namespace",
      "type": "stat",
      "targets": [{
        "expr": "count(kube_pod_info{namespace=~\"tenant-.*\"}) by (namespace)"
      }]
    },
    {
      "title": "Quota Usage %",
      "type": "gauge",
      "targets": [{
        "expr": "kube_resourcequota{resource=\"limits.cpu\",type=\"used\",namespace=~\"tenant-.*\"} / kube_resourcequota{resource=\"limits.cpu\",type=\"hard\",namespace=~\"tenant-.*\"} * 100"
      }]
    }
  ]
}
DASHBOARD

kubectl create configmap tenant-dashboard \
  --from-file=tenant-dashboard.json \
  -n monitoring \
  --dry-run=client -o yaml | kubectl apply -f -
```

---

## 14. OpenCost — Cost Allocation per Namespace

```bash
helm repo add opencost-charts https://opencost.github.io/opencost-helm-chart
helm repo update

kubectl create namespace opencost
kubectl label namespace opencost pod-security.kubernetes.io/enforce=privileged --overwrite

cat > opencost-values.yaml <<'EOF'
opencost:
  exporter:
    # On-premises bare-metal pricing (adjust to your hardware costs)
    customPricing:
      enabled: true
      costModel:
        description: "Production VMware bare-metal cluster"
        CPU: "0.032"          # USD per vCPU-hour
        RAM: "0.004"          # USD per GiB-hour
        storage: "0.000060"   # USD per GiB-hour (Longhorn replicated)
        GPU: "0.000000"

    env:
      - name: PROMETHEUS_SERVER_ENDPOINT
        value: "http://kube-prometheus-stack-prometheus.monitoring.svc.cluster.local:9090"
      - name: CLUSTER_ID
        value: "prod-saas-cluster"

    persistence:
      enabled: true
      storageClass: longhorn
      size: 10Gi

    resources:
      requests:
        cpu: 100m
        memory: 256Mi
      limits:
        cpu: "1"
        memory: 1Gi

  ui:
    enabled: true
    resources:
      requests:
        cpu: 50m
        memory: 64Mi
      limits:
        cpu: 500m
        memory: 256Mi

  mcp:
    enabled: true
    port: 8081
EOF

helm install opencost opencost-charts/opencost \
  --namespace opencost \
  --values opencost-values.yaml \
  --wait \
  --timeout 10m

# Expose OpenCost UI (admin only)
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: opencost-lb
  namespace: opencost
  annotations:
    metallb.universe.tf/address-pool: admin-pool
spec:
  type: LoadBalancer
  loadBalancerIP: 192.168.29.102
  selector:
    app.kubernetes.io/name: opencost
  ports:
    - name: ui
      port: 80
      targetPort: 9090
    - name: api
      port: 9003
      targetPort: 9003
EOF

echo "OpenCost UI:  http://192.168.29.102"
echo "OpenCost API: http://192.168.29.102:9003"
```

### 14.1 — Per-Tenant Cost Queries

```bash
OPENCOST_API="http://192.168.29.102:9003"

# ── Cost by namespace (all tenants) ───────────────────
curl -sG ${OPENCOST_API}/allocation \
  -d window=30d \
  -d aggregate=namespace \
  -d accumulate=true \
  -d 'filter=namespace:~"tenant-.*"' | \
  jq -r '.data[0] | to_entries[] | [.key, (.value.totalCost | tostring)] | @tsv' | \
  sort -t$'\t' -k2 -rn | \
  awk -F'\t' '{printf "%-40s $%s\n", $1, $2}'

# ── Cost for a specific tenant ─────────────────────────
curl -sG ${OPENCOST_API}/allocation \
  -d window=30d \
  -d aggregate=namespace \
  -d accumulate=true \
  -d 'filter=namespace:"tenant-alice-ns"' | \
  jq '.data[0]'

# ── Monthly invoice for all tenants ───────────────────
START="$(date +%Y-%m)-01T00:00:00Z"
END="$(date -u +%Y-%m-%dT%H:%M:%SZ)"

curl -sG ${OPENCOST_API}/allocation \
  -d "window=${START},${END}" \
  -d aggregate=namespace \
  -d accumulate=true \
  -d 'filter=namespace:~"tenant-.*"' | \
  jq -r '["Namespace","CPU","Memory","Storage","Total"],
    (.data[0] | to_entries[] |
      [.key, .value.cpuCost, .value.ramCost, .value.pvCost, .value.totalCost]) |
    @csv'
```

---

## 15. Security Hardening

### 15.1 — Talos OS Immutability (Built-in)

Talos Linux provides the following security guarantees by default:
- Root filesystem mounted **read-only** — no filesystem writes outside designated paths
- **No SSH, no shell, no console access** — all management via mTLS-secured API
- **Mutual TLS** on all Talos API calls — `talosconfig` PKI managed certificates
- **Immutable kernel parameters** — no runtime changes without machine config update
- Minimal attack surface: only ~12 binaries in the entire OS

### 15.2 — PodSecurity Admission (Kubernetes-native)

Already configured in `cp-patch.yaml`. Verification:

```bash
# Verify PodSecurity is enforcing 'restricted' on tenant namespaces
kubectl get namespace -l "tenant" \
  -o custom-columns='NAME:.metadata.name,ENFORCE:.metadata.labels.pod-security\.kubernetes\.io/enforce'

# Test: this should FAIL — privileged pod rejected
kubectl run test-priv --image=alpine --privileged=true \
  -n tenant-alice-ns -- sleep 60
# Expected: Error from server: pods "test-priv" is forbidden: ...

# Test: this should SUCCEED — standard non-root pod
kubectl run test-ok --image=alpine:3.21 \
  --overrides='{"spec":{"securityContext":{"runAsNonRoot":true,"runAsUser":1000,"seccompProfile":{"type":"RuntimeDefault"}},"containers":[{"name":"test-ok","image":"alpine:3.21","command":["sleep","60"],"securityContext":{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]}}}]}}' \
  -n tenant-alice-ns
kubectl delete pod test-ok -n tenant-alice-ns
```

### 15.3 — Calico GlobalNetworkPolicies

```bash
kubectl apply -f - <<'EOF'
# ── Protect admin platform namespaces from non-admin IPs ──
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: allow-admin-to-platform-services
spec:
  order: 100
  selector: >
    projectcalico.org/namespace in
    {'longhorn-system','opencost','monitoring','velero','cattle-system','traefik'}
  ingress:
    - action: Allow
      source:
        nets:
          - 192.168.1.10/32
          - 10.10.0.0/24
          - 192.168.1.20/32
    # Allow in-cluster access (pods within cluster)
    - action: Allow
      source:
        nets:
          - 10.244.0.0/16
          - 10.96.0.0/12
  egress:
    - action: Allow
  types: [Ingress, Egress]
---
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: deny-non-admin-to-platform-services
spec:
  order: 200
  selector: >
    projectcalico.org/namespace in
    {'longhorn-system','opencost','monitoring','velero','cattle-system'}
  ingress:
    - action: Deny
  types: [Ingress]
---
# ── Prevent tenant-to-tenant cross-namespace traffic ──
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: tenant-isolation
spec:
  order: 300
  selector: "tenant != ''"
  ingress:
    - action: Allow
      source:
        selector: "tenant == ''"   # Same tenant
    - action: Allow
      source:
        namespaceSelector: "kubernetes.io/metadata.name == 'traefik'"
    - action: Deny
      source:
        selector: "tenant != ''"   # Different tenant — explicit deny
  types: [Ingress]
---
# ── Protect Kubernetes API ─────────────────────────────
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
    - action: Allow
      protocol: TCP
      destination:
        nets: [192.168.29.10/32, 192.168.29.11/32, 192.168.29.12/32,
               192.168.29.13/32, 192.168.29.14/32, 192.168.29.15/32]
        ports: [6443]
      source:
        nets: [192.168.1.10/32, 10.10.0.0/24, 192.168.1.20/32]
    - action: Deny
      protocol: TCP
      destination:
        nets: [192.168.29.10/32, 192.168.29.11/32, 192.168.29.12/32,
               192.168.29.13/32, 192.168.29.14/32, 192.168.29.15/32]
        ports: [6443]
  types: [Ingress]
EOF
```

### 15.4 — Talos Host Firewall

```bash
cat > talos-firewall-patch.yaml <<'EOF'
machine:
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
      - name: allow-k8s-internal-talos
        protocol: TCP
        destinationPort: 50000-51000
        action: accept
        ingressSubnets:
          - 192.168.29.0/24    # Nodes talking to each other
      - name: deny-all-talos-api
        protocol: TCP
        destinationPort: 50000-51000
        action: drop
EOF

for NODE_IP in 192.168.29.11 192.168.29.12 192.168.29.13 192.168.29.14 192.168.29.15 \
               192.168.29.21 192.168.29.22 192.168.29.23 192.168.29.24 192.168.29.25; do
  echo "Applying firewall to ${NODE_IP}..."
  talosctl --talosconfig cluster-configs/talosconfig \
    patch machineconfig -n ${NODE_IP} -e ${NODE_IP} \
    --patch @talos-firewall-patch.yaml
done
```

### 15.5 — Falco Runtime Security

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update

cat > falco-values.yaml <<'EOF'
driver:
  kind: modern_ebpf    # No kernel module needed on Talos

# Falco Sidekick — send alerts to Slack / PagerDuty / AlertManager
falcosidekick:
  enabled: true
  config:
    slack:
      webhookurl: "https://hooks.slack.com/REPLACE_WITH_YOUR_WEBHOOK"
      minimumpriority: warning
    alertmanager:
      hostport: "http://kube-prometheus-stack-alertmanager.monitoring.svc.cluster.local:9093"
      minimumpriority: warning

# Custom rules for tenant namespace events
customRules:
  tenant-security.yaml: |-
    - rule: Tenant Shell Execution
      desc: Shell spawned in a tenant namespace
      condition: >
        spawned_process and
        (proc.name in (shell_binaries)) and
        k8s.ns.name startswith "tenant-"
      output: >
        Shell in tenant namespace (user=%user.name pod=%k8s.pod.name
        ns=%k8s.ns.name cmd=%proc.cmdline)
      priority: WARNING
      tags: [tenant, shell]

    - rule: Tenant Privilege Escalation
      desc: Privilege escalation attempt in tenant namespace
      condition: >
        evt.type = execve and
        proc.uid = 0 and
        k8s.ns.name startswith "tenant-"
      output: >
        Privilege escalation in tenant ns (pod=%k8s.pod.name ns=%k8s.ns.name)
      priority: CRITICAL
      tags: [tenant, privilege_escalation]
EOF

helm install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  --values falco-values.yaml \
  --wait

kubectl get pods -n falco
```

---

## 16. Admin Access Control

```bash
# ── Verify all GlobalNetworkPolicies ───────────────────
kubectl get globalnetworkpolicy \
  -o custom-columns='NAME:.metadata.name,ORDER:.spec.order,ACTION:.spec.ingress[0].action'

# ── Verify Talos firewall on a node ────────────────────
talosctl --talosconfig cluster-configs/talosconfig \
  --nodes 192.168.29.21 get networkrulesets

# ── Test: Longhorn UI accessible from admin IP ─────────
curl -s -o /dev/null -w "Longhorn: %{http_code}\n" http://192.168.29.100

# ── Test: OpenCost API health ──────────────────────────
curl -s http://192.168.29.102:9003/healthz
# Expected: {"status":"ok"}

# ── Test: Rancher health ───────────────────────────────
curl -sk https://rancher.prod.example.com/healthz
# Expected: ok

# ── Test: Grafana accessible ───────────────────────────
curl -sk -o /dev/null -w "Grafana: %{http_code}\n" https://grafana.prod.example.com
# Expected: 302 (redirect to login)
```

---

## 17. Operations & Maintenance

### 17.1 — Daily Health Check

```bash
#!/usr/bin/env bash
# daily-health-check.sh — Run every morning via cron

set -euo pipefail

echo "════════════════════════════════════════════"
echo " Daily Cluster Health — $(date -u)"
echo "════════════════════════════════════════════"

echo -e "\n[1] Node Status (10 nodes expected):"
kubectl get nodes \
  -o custom-columns='NAME:.metadata.name,STATUS:.status.conditions[-1].type,VERSION:.status.nodeInfo.kubeletVersion'

echo -e "\n[2] Failed / Pending Pods:"
kubectl get pods -A | grep -vE 'Running|Completed|Succeeded' | grep -v NAME || echo "  None"

echo -e "\n[3] Node Resource Usage:"
kubectl top nodes

echo -e "\n[4] Longhorn Volume Health:"
kubectl get volumes.longhorn.io -n longhorn-system \
  -o custom-columns='NAME:.metadata.name,STATE:.status.state,ROBUSTNESS:.status.robustness' | \
  grep -v Healthy || echo "  All volumes healthy"

echo -e "\n[5] Velero Backup Status (last 24h):"
velero backup get --selector '' | awk 'NR<=10'

echo -e "\n[6] Tenant Namespace Quotas:"
for NS in $(kubectl get ns -l tenant --no-headers -o name | cut -d/ -f2); do
  echo "  --- ${NS} ---"
  kubectl describe resourcequota tenant-quota -n ${NS} 2>/dev/null | \
    grep -E 'limits.cpu|limits.memory|pods' | head -6
done

echo -e "\n[7] etcd Health:"
talosctl --talosconfig cluster-configs/talosconfig \
  --endpoints 192.168.29.10 etcd members

echo -e "\n[8] VIP Reachability:"
ping -c 2 192.168.29.10 && echo "VIP OK" || echo "VIP UNREACHABLE!"

echo -e "\nHealth check complete."
```

### 17.2 — etcd Backup

```bash
# Snapshot etcd (run on cp01)
SNAPSHOT_FILE="etcd-snapshot-$(date +%Y%m%d-%H%M%S).db"
talosctl --talosconfig cluster-configs/talosconfig \
  --nodes 192.168.29.11 \
  etcd snapshot /tmp/${SNAPSHOT_FILE}

talosctl --talosconfig cluster-configs/talosconfig \
  --nodes 192.168.29.11 \
  cp /tmp/${SNAPSHOT_FILE} ./${SNAPSHOT_FILE}

# Encrypt and archive
gpg --symmetric --cipher-algo AES256 ${SNAPSHOT_FILE}
rm ${SNAPSHOT_FILE}
echo "Snapshot saved: ${SNAPSHOT_FILE}.gpg"
```

### 17.3 — Kubernetes Version Upgrade (v1.34.x → v1.35.x)

```bash
NEW_K8S_VERSION="v1.35.0"

# ── Step 1: Upgrade control planes (one at a time) ────
for CP_IP in 192.168.29.11 192.168.29.12 192.168.29.13 192.168.29.14 192.168.29.15; do
  echo "Upgrading Kubernetes on ${CP_IP}..."
  talosctl --talosconfig cluster-configs/talosconfig \
    --nodes ${CP_IP} upgrade-k8s --to ${NEW_K8S_VERSION}
  echo "Waiting 3 minutes for CP to stabilise..."
  sleep 180
  kubectl get nodes | grep -v Ready | grep -v NAME || echo "All nodes ready"
done

# ── Step 2: Upgrade workers (drain, upgrade, uncordon) ──
for W_IP in 192.168.29.21 192.168.29.22 192.168.29.23 192.168.29.24 192.168.29.25; do
  NODE_NAME=$(kubectl get nodes -o wide | grep ${W_IP} | awk '{print $1}')
  echo "Draining ${NODE_NAME} (${W_IP})..."
  kubectl drain ${NODE_NAME} \
    --ignore-daemonsets \
    --delete-emptydir-data \
    --grace-period=60 \
    --timeout=5m

  echo "Upgrading Kubernetes on ${W_IP}..."
  talosctl --talosconfig cluster-configs/talosconfig \
    --nodes ${W_IP} upgrade-k8s --to ${NEW_K8S_VERSION}
  sleep 120

  echo "Uncordoning ${NODE_NAME}..."
  kubectl uncordon ${NODE_NAME}
  sleep 30
done

kubectl get nodes
```

### 17.4 — Talos OS Upgrade (v1.13.0 → v1.14.x)

```bash
NEW_TALOS="v1.14.0"
SCHEMATIC_ID="a9d2d26b9f0b1b1c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7"

# Upgrade CPs
for CP_IP in 192.168.29.11 192.168.29.12 192.168.29.13 192.168.29.14 192.168.29.15; do
  talosctl --talosconfig cluster-configs/talosconfig \
    --nodes ${CP_IP} upgrade \
    --image factory.talos.dev/installer/${SCHEMATIC_ID}:${NEW_TALOS}
  sleep 180
done

# Upgrade workers with drain/uncordon
for W_IP in 192.168.29.21 192.168.29.22 192.168.29.23 192.168.29.24 192.168.29.25; do
  NODE_NAME=$(kubectl get nodes -o wide | grep ${W_IP} | awk '{print $1}')
  kubectl drain ${NODE_NAME} --ignore-daemonsets --delete-emptydir-data
  talosctl --talosconfig cluster-configs/talosconfig \
    --nodes ${W_IP} upgrade \
    --image factory.talos.dev/installer/${SCHEMATIC_ID}:${NEW_TALOS}
  sleep 180
  kubectl uncordon ${NODE_NAME}
done
```

### 17.5 — Add a New Worker Node

```bash
NEW_NODE="w06"
NEW_IP="192.168.29.26"

# Generate config
gen_worker ${NEW_NODE} ${NEW_IP}   # Uses function from Step 6

# Apply to new node (in maintenance mode)
talosctl apply-config --insecure \
  --nodes ${NEW_IP} \
  --file cluster-configs/${NEW_NODE}.yaml

sleep 300
kubectl get nodes
# w06 should appear as Ready

# Label for workloads
kubectl label node ${NEW_NODE} dedicated=workloads

# Disk preparation (Longhorn)
talosctl --talosconfig cluster-configs/talosconfig \
  patch machineconfig -n ${NEW_IP} -e ${NEW_IP} \
  --patch @longhorn-disk-patch.yaml
```

### 17.6 — Offboard a Tenant

```bash
#!/usr/bin/env bash
# offboard-tenant.sh — Remove tenant cleanly
TENANT_NAME="${1:?Provide tenant name}"
NS_NAME="tenant-${TENANT_NAME}-ns"

echo "Offboarding tenant: ${TENANT_NAME}"

# 1. Take a final backup before deletion
velero backup create "final-${TENANT_NAME}-$(date +%Y%m%d)" \
  --include-namespaces ${NS_NAME} \
  --snapshot-volumes=true \
  --wait

# 2. Delete Velero schedule
velero schedule delete "backup-${TENANT_NAME}" --confirm || true

# 3. Remove WordPress Helm release
helm uninstall "wordpress-${TENANT_NAME}" -n ${NS_NAME} || true

# 4. Delete namespace (removes all resources)
kubectl delete namespace ${NS_NAME} --timeout=5m

# 5. Delete Rancher project via API
PROJECT_ID=$(curl -sk -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  "${RANCHER_URL}/v3/projects?name=Tenant-${TENANT_NAME}" | jq -r '.data[0].id')
curl -sk -X DELETE \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  "${RANCHER_URL}/v3/projects/${PROJECT_ID}"

echo "Tenant ${TENANT_NAME} offboarded. Final backup retained 30 days."
```

---

## 18. Scaling & HA Best Practices

### Scaling Workers Horizontally

```bash
# Add workers w06-w10 for increased capacity
for i in 6 7 8 9 10; do
  gen_worker "w0${i}" "192.168.29.2${i}"
  talosctl apply-config --insecure \
    --nodes "192.168.29.2${i}" \
    --file "cluster-configs/w0${i}.yaml"
done
```

### Node Affinity for Tenant Workloads

```yaml
# Dedicate specific nodes to specific tier workloads
# Apply labels to nodes first:
# kubectl label node w01 w02 workload-tier=starter
# kubectl label node w03 w04 workload-tier=professional
# kubectl label node w05     workload-tier=enterprise

affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: workload-tier
              operator: In
              values:
                - professional    # or starter / enterprise
```

### etcd Performance Tuning (5-node cluster)

```yaml
# Add to cp-patch.yaml under cluster.etcd:
cluster:
  etcd:
    advertisedSubnets:
      - 192.168.29.0/24
    # Performance tuning for 5-node etcd
    extraArgs:
      election-timeout: "2500"       # ms — increase if CP nodes are under load
      heartbeat-interval: "500"      # ms
      snapshot-count: "10000"
      quota-backend-bytes: "8589934592"  # 8 GB etcd DB limit
      auto-compaction-retention: "1"     # Compact every 1 hour
      auto-compaction-mode: "periodic"
```

### Pod Disruption Budgets (Cluster-wide)

```bash
# Ensure Traefik always has at least 2 pods
kubectl apply -f - <<'EOF'
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: traefik-pdb
  namespace: traefik
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: traefik
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: rancher-pdb
  namespace: cattle-system
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: rancher
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: prometheus-pdb
  namespace: monitoring
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: prometheus
EOF
```

### Horizontal Pod Autoscaling for Tenant Apps

```bash
# Enable HPA for WordPress in a tenant namespace
kubectl apply -f - <<EOF
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: wordpress-hpa
  namespace: tenant-alice-ns
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: wordpress-alice
  minReplicas: 1
  maxReplicas: 5             # Constrained by namespace quota
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
EOF
```

### Best Practices Summary

| Category | Recommendation |
|----------|----------------|
| **etcd** | Always odd number (3 or 5). Never 2 or 4. Backup every 6h. |
| **Control Planes** | 5 for production (tolerates 2 simultaneous failures) |
| **Worker Nodes** | Minimum 3 for Longhorn 3× replication. 5+ for rolling upgrades without draining |
| **Storage** | Dedicated disks for Longhorn — never share with OS disk |
| **Upgrades** | Always upgrade CPs first, then workers one at a time |
| **Tenants** | 1 namespace per tenant minimum. Use separate projects for billing isolation |
| **Backups** | Daily Velero + hourly etcd snapshot + monthly restore test |
| **Certificates** | cert-manager with Let's Encrypt or internal CA. Rotate before expiry |
| **Monitoring** | Alert at 80% quota usage — SaaS users need time to upgrade before hitting hard limits |
| **Network** | Always test NetworkPolicies with deny-all before production |
| **Security** | Scan all container images with Trivy. Enable Falco on all workers |

---

## 19. Troubleshooting

### Node Not Joining Cluster

```bash
# Check node status via Talos API
talosctl --talosconfig cluster-configs/talosconfig \
  --nodes 192.168.29.13 service kubelet status

# Check etcd connectivity from new CP
talosctl --talosconfig cluster-configs/talosconfig \
  --nodes 192.168.29.13 service etcd status

# Re-apply config
talosctl apply-config --nodes 192.168.29.13 \
  --file cluster-configs/cp03.yaml --mode reboot
```

### VIP Not Responding

```bash
# Check which CP currently holds the VIP
for NODE in 192.168.29.11 192.168.29.12 192.168.29.13 192.168.29.14 192.168.29.15; do
  echo -n "${NODE}: "
  talosctl --talosconfig cluster-configs/talosconfig \
    --nodes ${NODE} get links | grep -A2 ens192 | grep -i vip || echo "(no VIP)"
done

# Check etcd leader
talosctl --talosconfig cluster-configs/talosconfig \
  --endpoints 192.168.29.10 etcd status
```

### Rancher Pods CrashLooping

```bash
kubectl get pods -n cattle-system
kubectl logs -n cattle-system -l app=rancher --tail=100 | grep -i error
kubectl describe pod -n cattle-system -l app=rancher | grep -A5 "Events:"

# Common fix: cert-manager not ready when Rancher starts
kubectl rollout restart deployment/rancher -n cattle-system
```

### Velero Backup Failing

```bash
# Check Velero pod logs
kubectl logs -n velero deployment/velero --tail=100

# Check backup storage location
velero backup-location get

# Test MinIO connectivity from Velero pod
kubectl exec -n velero deployment/velero -- \
  wget -qO- http://minio.minio-system.svc.cluster.local:9000/minio/health/live

# List all failed backups
velero backup get | grep PartiallyFailed
velero backup describe BACKUP_NAME --details
velero backup logs BACKUP_NAME
```

### Longhorn Volume Degraded

```bash
# Check volume status
kubectl get volumes.longhorn.io -n longhorn-system | grep -v Healthy

# Check Longhorn nodes
kubectl get nodes.longhorn.io -n longhorn-system

# Check disk health on workers
for W_IP in 192.168.29.21 192.168.29.22 192.168.29.23 192.168.29.24 192.168.29.25; do
  echo "--- ${W_IP} disk usage ---"
  talosctl --talosconfig cluster-configs/talosconfig --nodes ${W_IP} df /dev/sdb
done

# Force rebuild replica
kubectl -n longhorn-system edit volume.longhorn.io VOLUME_NAME
# Set spec.nodeSelector to trigger re-scheduling
```

### Tenant Quota Exhausted

```bash
TENANT=alice
NS="tenant-${TENANT}-ns"

# Show current quota usage
kubectl describe resourcequota tenant-quota -n ${NS}

# Show all pods and their resource requests
kubectl get pods -n ${NS} \
  -o custom-columns='NAME:.metadata.name,CPU-REQ:.spec.containers[0].resources.requests.cpu,MEM-REQ:.spec.containers[0].resources.requests.memory'

# Upgrade tenant to higher tier (update quota)
kubectl patch resourcequota tenant-quota -n ${NS} \
  --type='json' \
  -p='[
    {"op":"replace","path":"/spec/hard/limits.cpu","value":"8"},
    {"op":"replace","path":"/spec/hard/limits.memory","value":"16Gi"}
  ]'
```

### MetalLB Service Pending

```bash
kubectl get svc -A --field-selector spec.type=LoadBalancer | grep Pending

# Check MetalLB controller
kubectl logs -n metallb-system -l component=controller --tail=100 | grep error

# Check speaker DaemonSet
kubectl get pods -n metallb-system -l component=speaker -o wide

# Verify IP pool has available IPs
kubectl get ipaddresspool -n metallb-system -o yaml
```

### Calico NetworkPolicy Not Working

```bash
# Verify Felix is running on all nodes
kubectl get pods -n calico-system -l k8s-app=calico-node

# Check GlobalNetworkPolicies
kubectl get globalnetworkpolicy \
  -o custom-columns='NAME:.metadata.name,ORDER:.spec.order'

# Check Felix logs for policy errors
kubectl logs -n calico-system -l k8s-app=calico-node \
  -c calico-node --tail=100 | grep -iE "policy|error|deny"
```

### Emergency Cluster Reset (⚠️ DESTRUCTIVE)

```bash
# WARNING: This destroys ALL data permanently.
# Ensure etcd snapshot and Velero backups are current before running.

echo "LAST CHANCE — Press Ctrl+C within 10 seconds to abort..."
sleep 10

for NODE in 192.168.29.11 192.168.29.12 192.168.29.13 192.168.29.14 192.168.29.15 \
            192.168.29.21 192.168.29.22 192.168.29.23 192.168.29.24 192.168.29.25; do
  echo "Resetting ${NODE}..."
  talosctl --talosconfig cluster-configs/talosconfig \
    reset -n ${NODE} -e ${NODE} --graceful=false --reboot
done
```

---

## 20. Quick Reference

### Essential Commands

```bash
# ── Cluster ────────────────────────────────────────────
kubectl get nodes -o wide                          # All 10 nodes
kubectl get pods -A | grep -vE 'Running|Completed' # Problem pods
talosctl --talosconfig cluster-configs/talosconfig health

# ── Tenants ────────────────────────────────────────────
kubectl get ns -l tenant                           # All tenant namespaces
./onboard-tenant.sh <name> <email> <tier>          # Add tenant
./deploy-wordpress.sh <name> <namespace>           # Deploy app
./offboard-tenant.sh <name>                        # Remove tenant

# ── Resources ──────────────────────────────────────────
kubectl top nodes
kubectl top pods -A --sort-by=memory
kubectl describe resourcequota tenant-quota -n <ns>

# ── Storage ────────────────────────────────────────────
kubectl get pvc -A
kubectl get volumes.longhorn.io -n longhorn-system
kubectl get storageclass

# ── Network ────────────────────────────────────────────
kubectl get svc -A --field-selector spec.type=LoadBalancer
kubectl get globalnetworkpolicy
kubectl get ingress -A

# ── Backup ─────────────────────────────────────────────
velero schedule get
velero backup get
velero backup create manual-<tenant>-$(date +%Y%m%d) --include-namespaces <ns>

# ── etcd ───────────────────────────────────────────────
talosctl --talosconfig cluster-configs/talosconfig --endpoints 192.168.29.10 etcd members
talosctl --talosconfig cluster-configs/talosconfig --nodes 192.168.29.11 etcd snapshot /tmp/backup.db

# ── Rancher API ────────────────────────────────────────
curl -sk -H "Authorization: Bearer ${RANCHER_TOKEN}" ${RANCHER_URL}/v3/projects
curl -sk -H "Authorization: Bearer ${RANCHER_TOKEN}" ${RANCHER_URL}/v3/users
```

### Key Files

| File | Purpose |
|------|---------|
| `cluster-configs/talosconfig` | Talos CLI authentication |
| `cluster-configs/kubeconfig` | Kubernetes CLI authentication |
| `cluster-configs/cp01-cp05.yaml` | Control plane node configs |
| `cluster-configs/w01-w05.yaml` | Worker node configs |
| `cp-patch.yaml` | CP base patch (VIP, hostentries, PodSecurity) |
| `worker-patch.yaml` | Worker base patch (Longhorn disk, sysctls) |
| `rancher-values.yaml` | Rancher Helm values |
| `velero-values.yaml` | Velero Helm values |
| `kube-prometheus-values.yaml` | kube-prometheus-stack Helm values |
| `opencost-values.yaml` | OpenCost Helm values |
| `onboard-tenant.sh` | Full tenant onboarding automation |
| `deploy-wordpress.sh` | WordPress deployment per tenant |
| `offboard-tenant.sh` | Tenant cleanup |
| `daily-health-check.sh` | Morning cluster health report |

### Network & Service Reference

| Service | Address | Auth |
|---------|---------|------|
| Kubernetes API | `https://k8s.prod.example.com:6443` | Admin IPs only |
| Rancher UI | `https://rancher.prod.example.com` | Admin IPs only |
| Longhorn UI | `http://192.168.29.100` | Admin IPs only |
| OpenCost UI | `http://192.168.29.102` | Admin IPs only |
| Grafana UI | `https://grafana.prod.example.com` | Admin IPs only |
| Tenant Ingress | `https://<tenant>.tenant.prod.example.com` | Public |
| OpenCost API | `http://192.168.29.102:9003` | Admin IPs only |
| OpenCost MCP | `http://opencost.opencost.svc.cluster.local:8081` | In-cluster |
| MinIO | `http://minio.minio-system.svc.cluster.local:9000` | In-cluster |

### Port Reference

| Port | Service | Allowed From |
|------|---------|-------------|
| 6443 | Kubernetes API | Admin IPs |
| 50000–51000 | Talos API | Admin IPs |
| 80 / 443 | Traefik Ingress | Public |
| 9000 | MinIO | In-cluster |
| 9003 | OpenCost API | Admin IPs |
| 8081 | OpenCost MCP | In-cluster |
| 9090 | Prometheus | In-cluster |
| 3000 | Grafana | Admin IPs via LB |
| 2379–2380 | etcd | CP nodes only |
| 4789 | Calico VXLAN | All cluster nodes |

---

## Support Resources

| Tool | Documentation |
|------|--------------|
| **Talos Linux v1.13** | https://www.talos.dev/v1.13/ |
| **Kubernetes v1.34** | https://kubernetes.io/docs/ |
| **Calico v3.31** | https://docs.tigera.io/calico/3.31/ |
| **MetalLB v0.15** | https://metallb.universe.tf/ |
| **Traefik v3.6** | https://doc.traefik.io/traefik/ |
| **Longhorn v1.12** | https://longhorn.io/docs/1.12.0/ |
| **cert-manager v1.20** | https://cert-manager.io/docs/ |
| **Rancher v2.14** | https://ranchermanager.docs.rancher.com/ |
| **Rancher API v3** | https://ranchermanager.docs.rancher.com/reference-guides/rancher-manager-api |
| **Velero v1.18** | https://velero.io/docs/v1.18/ |
| **kube-prometheus-stack** | https://github.com/prometheus-community/helm-charts |
| **OpenCost** | https://opencost.io/docs/ |
| **Falco** | https://falco.org/docs/ |

---

## What We Built

✅ **5-Node HA Control Plane** — VIP `192.168.29.10`, quorum 3/5, tolerates 2 simultaneous CP failures

✅ **5-Node Worker Pool** — 200 GB dedicated Longhorn disks each, 1 TB total raw storage (333 GB usable at 3× replica)

✅ **Calico v3.31.5 CNI** — VXLAN mode, GlobalNetworkPolicy, cross-tenant isolation

✅ **MetalLB v0.15.3** — Three IP pools: admin-only, tenant-ingress, general

✅ **Traefik v3.6.13** — HA Ingress (3 replicas), TLS termination, per-tenant middleware

✅ **Longhorn v1.12.0** — 3× replication, v2 data engine, CSI snapshots for Velero

✅ **cert-manager v1.20.2** — Automatic Let's Encrypt TLS for Rancher and all tenant domains

✅ **Rancher v2.14.2** — Multi-cluster management, Projects, RBAC, Helm catalog

✅ **Multi-Tenancy** — Isolated namespaces, resource quotas (3 tiers), NetworkPolicies, PodSecurity

✅ **Rancher API Automation** — `onboard-tenant.sh`: project + namespace + quota + RBAC + backup in one command

✅ **WordPress per Tenant** — `deploy-wordpress.sh`: Helm-deployed, Longhorn PVC, Traefik Ingress, TLS, HPA

✅ **Velero v1.18.0** — CSI + Longhorn snapshots, per-tenant daily schedules, 30-day retention, concurrent backup

✅ **kube-prometheus-stack v86.2.2** — Prometheus HA + Grafana + Alertmanager + tenant quota alerts

✅ **OpenCost** — Per-namespace cost allocation, custom on-premises pricing, monthly invoice API

✅ **Falco** — Runtime security, tenant shell-execution detection, Slack/Alertmanager alerts

✅ **Security Hardening** — Talos immutable OS + PodSecurity restricted + Calico GlobalNetPol + host firewall

---

*Last Updated: June 2026 — Version 8.0*
*Based on: [vaheed/talos-kubernetes-cluster](https://github.com/vaheed/talos-kubernetes-cluster) — Extended to 5CP+5W with full SaaS multi-tenancy*
