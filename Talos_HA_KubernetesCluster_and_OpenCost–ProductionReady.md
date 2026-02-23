# Talos HA Kubernetes Cluster and OpenCost – Production Ready

Complete production guide for building a high-availability Talos Linux Kubernetes cluster with Longhorn distributed storage, MetalLB load balancing, static node hostnames, admin IP access control, and OpenCost cost monitoring — API-first, copy-paste ready.

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
| Prometheus | latest stable (OpenCost backend) |
| OpenCost | latest stable (CNCF) |

---

## Table of Contents

1. [Cluster Topology](#cluster-topology)
2. [Architecture Overview](#architecture-overview)
3. [Prerequisites](#prerequisites)
4. [Installation Steps](#installation-steps)
5. [Network Components](#network-components)
6. [Storage Configuration](#storage-configuration)
7. [Admin Access Control – IP Allowlist](#admin-access-control--ip-allowlist)
8. [OpenCost – Cost Monitoring](#opencost--cost-monitoring)
9. [Verification & Testing](#verification--testing)
10. [Operations & Maintenance](#operations--maintenance)
11. [Optional Hardening & Improvements](#optional-hardening--improvements)
12. [Troubleshooting](#troubleshooting)

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

| Name | IP | Static Hostname | Role |
|------|----|-----------------|------|
| cp01 | 192.168.29.11 | cp01 | Control Plane + etcd |
| cp02 | 192.168.29.12 | cp02 | Control Plane + etcd |

### Worker Nodes

| Name | IP | Static Hostname | Storage Device |
|------|----|-----------------|----------------|
| w01 | 192.168.29.21 | w01 | /dev/sdb |
| w02 | 192.168.29.22 | w02 | /dev/sdb |
| w03 | 192.168.29.23 | w03 | /dev/sdb |

### Service Ranges

| Service | Address / Range |
|---------|----------------|
| Kubernetes VIP | 192.168.29.10 |
| MetalLB Admin Pool | 192.168.29.100 – 192.168.29.109 |
| MetalLB General Pool | 192.168.29.110 – 192.168.29.199 |
| Longhorn UI | 192.168.29.100 |
| OpenCost UI / API | 192.168.29.101 |
| Pod Network | 10.244.0.0/16 |
| Service Network | 10.96.0.0/12 |

### Admin Source IPs

> **Replace these with your real admin/VPN IPs before applying.**

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
│  Longhorn OSD    │ │  Longhorn OSD    │ │  Longhorn OSD   │
└──────────────────┘ └──────────────────┘ └─────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                   Kubernetes Services                        │
├─────────────────────────────────────────────────────────────┤
│ • Calico v3.30.5 CNI + NetworkPolicy + GlobalNetworkPolicy  │
│ • MetalLB v0.14.9 – Admin pool (.100–.109)                  │
│ • Longhorn v1.10.1 – 3× replicated distributed storage      │
│ • Metrics Server – kubectl top                              │
│ • Prometheus – OpenCost metrics backend                     │
│ • OpenCost – Real-time cost allocation API + UI             │
└─────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

### Hardware Requirements

**Control Plane Nodes (each):** 2+ vCPU, 4 GB+ RAM, 50 GB+ OS disk, interface `ens192`

**Worker Nodes (each):** 4+ vCPU, 8 GB+ RAM, 50 GB+ OS disk (`/dev/sda`), 100 GB+ data disk (`/dev/sdb`), interface `ens192`

### Software Requirements

- VMware vSphere environment
- `talosctl` v1.11.6, `kubectl` v1.30.8+, `helm` v3.x
- DNS: `29.talos.vaheed.net` → `192.168.29.10`
- DNS: `opencost.29.talos.vaheed.net` → `192.168.29.101`

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

# Linux amd64
wget https://github.com/siderolabs/talos/releases/download/v1.11.6/talosctl-linux-amd64
chmod +x talosctl-linux-amd64 && sudo mv talosctl-linux-amd64 /usr/local/bin/talosctl

talosctl version --client
```

### Step 3: Create VMs in vSphere

1. Upload `talos-vmware-1.11.6.iso` to your datastore.
2. Create 5 VMs:
   - **cp01, cp02**: 2 vCPU, 4 GB RAM, 50 GB disk
   - **w01, w02, w03**: 4 vCPU, 8 GB RAM, 50 GB OS disk + 100 GB data disk (`/dev/sdb`)
3. Mount ISO and boot each VM. Nodes enter maintenance mode automatically.

### Step 4: Generate Base Configuration

```bash
talosctl gen config "talos29" "https://29.talos.vaheed.net:6443" \
  --kubernetes-version v1.30.8 \
  --install-image factory.talos.dev/installer/dfd1ac9abdf529ca644694b17af0ce1a2ae23a5cccdff39439aa7f0774901e90:v1.11.6
```

Files created: `controlplane.yaml`, `worker.yaml`, `talosconfig`

### Step 5: Create Control Plane Base Patch

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

### Step 6: Create Worker Base Patch

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

### Step 7: Generate Per-Node Configurations with Static Hostnames

Each node receives both its static IP and a **static hostname** via the `machine.network.hostname` field. Without this, Talos falls back to a random or DHCP-assigned name that changes on reboot.

#### cp01 — IP: 192.168.29.11, Hostname: cp01

```bash
talosctl machineconfig patch controlplane.yaml --patch @cp-patch.yaml --output cp01.yaml

talosctl machineconfig patch cp01.yaml \
  --patch '[
    {"op":"replace","path":"/machine/network/interfaces","value":[
      {"interface":"ens192","dhcp":false,"addresses":["192.168.29.11/24"],
       "routes":[{"network":"0.0.0.0/0","gateway":"192.168.29.1"}],
       "vip":{"ip":"192.168.29.10"}}
    ]},
    {"op":"add","path":"/machine/network/hostname","value":"cp01"}
  ]' \
  --output cp01.yaml
```

#### cp02 — IP: 192.168.29.12, Hostname: cp02

```bash
talosctl machineconfig patch controlplane.yaml --patch @cp-patch.yaml --output cp02.yaml

talosctl machineconfig patch cp02.yaml \
  --patch '[
    {"op":"replace","path":"/machine/network/interfaces","value":[
      {"interface":"ens192","dhcp":false,"addresses":["192.168.29.12/24"],
       "routes":[{"network":"0.0.0.0/0","gateway":"192.168.29.1"}],
       "vip":{"ip":"192.168.29.10"}}
    ]},
    {"op":"add","path":"/machine/network/hostname","value":"cp02"}
  ]' \
  --output cp02.yaml
```

#### w01 — IP: 192.168.29.21, Hostname: w01

```bash
talosctl machineconfig patch worker.yaml --patch @worker-patch.yaml --output w01.yaml

talosctl machineconfig patch w01.yaml \
  --patch '[
    {"op":"replace","path":"/machine/network/interfaces","value":[
      {"interface":"ens192","dhcp":false,"addresses":["192.168.29.21/24"],
       "routes":[{"network":"0.0.0.0/0","gateway":"192.168.29.1"}]}
    ]},
    {"op":"add","path":"/machine/network/hostname","value":"w01"}
  ]' \
  --output w01.yaml
```

#### w02 — IP: 192.168.29.22, Hostname: w02

```bash
talosctl machineconfig patch worker.yaml --patch @worker-patch.yaml --output w02.yaml

talosctl machineconfig patch w02.yaml \
  --patch '[
    {"op":"replace","path":"/machine/network/interfaces","value":[
      {"interface":"ens192","dhcp":false,"addresses":["192.168.29.22/24"],
       "routes":[{"network":"0.0.0.0/0","gateway":"192.168.29.1"}]}
    ]},
    {"op":"add","path":"/machine/network/hostname","value":"w02"}
  ]' \
  --output w02.yaml
```

#### w03 — IP: 192.168.29.23, Hostname: w03

```bash
talosctl machineconfig patch worker.yaml --patch @worker-patch.yaml --output w03.yaml

talosctl machineconfig patch w03.yaml \
  --patch '[
    {"op":"replace","path":"/machine/network/interfaces","value":[
      {"interface":"ens192","dhcp":false,"addresses":["192.168.29.23/24"],
       "routes":[{"network":"0.0.0.0/0","gateway":"192.168.29.1"}]}
    ]},
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
# Run exactly once on cp01 only
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
```

### Step 10: Verify Static Hostnames

```bash
# Node names must be cp01, cp02, w01, w02, w03 — not random strings
kubectl --kubeconfig=kubeconfig get nodes \
  -o custom-columns='NAME:.metadata.name,IP:.status.addresses[0].address,STATUS:.status.conditions[-1].type'

# Verify directly via Talos API
for node in 192.168.29.11 192.168.29.12 192.168.29.21 192.168.29.22 192.168.29.23; do
  echo -n "$node → "
  talosctl --talosconfig talosconfig --nodes $node get hostname 2>/dev/null | awk 'NR==2{print $2}'
done

# Expected output:
# 192.168.29.11 → cp01
# 192.168.29.12 → cp02
# 192.168.29.21 → w01
# 192.168.29.22 → w02
# 192.168.29.23 → w03
```

If a node shows a wrong name, re-apply with `--mode reboot`:

```bash
# Example: fix w01 hostname
talosctl apply-config --nodes 192.168.29.21 --file w01.yaml --mode reboot
sleep 120
kubectl --kubeconfig=kubeconfig get nodes
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
# All nodes should show Ready
```

### Install MetalLB with Split IP Pools

A dedicated **admin pool** (`.100–.109`) is used for platform services (Longhorn UI, OpenCost). These IPs have `autoAssign: false` so only explicitly annotated services receive them. A **general pool** (`.110–.199`) is available for any other LoadBalancer services.

```bash
kubectl --kubeconfig=kubeconfig apply -f \
  https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/metallb-native.yaml

kubectl --kubeconfig=kubeconfig wait --for=condition=ready pod \
  -l app=metallb -n metallb-system --timeout=300s

kubectl --kubeconfig=kubeconfig apply -f - <<'EOF'
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
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: general-pool
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
    - general-pool
EOF

kubectl --kubeconfig=kubeconfig get ipaddresspool -n metallb-system
```

### Install Metrics Server

```bash
kubectl --kubeconfig=kubeconfig apply -f \
  https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Patch for Talos: use InternalIP and allow self-signed kubelet certs
kubectl --kubeconfig=kubeconfig patch deployment metrics-server -n kube-system --type='json' \
  -p='[
    {"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-preferred-address-types=InternalIP"},
    {"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}
  ]'

kubectl --kubeconfig=kubeconfig wait --for=condition=ready pod \
  -l k8s-app=metrics-server -n kube-system --timeout=300s

sleep 30
kubectl --kubeconfig=kubeconfig top nodes
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

### Expose Longhorn UI

The service uses the admin pool IP and `autoAssign: false` — you must specify the IP explicitly.

```bash
kubectl --kubeconfig=kubeconfig apply -f - <<'EOF'
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
kubectl --kubeconfig=kubeconfig get pvc test-pvc
kubectl --kubeconfig=kubeconfig exec test-pod -- sh -c "echo 'Longhorn OK' > /data/test.txt"
kubectl --kubeconfig=kubeconfig exec test-pod -- cat /data/test.txt

# Cleanup
kubectl --kubeconfig=kubeconfig delete pod test-pod
kubectl --kubeconfig=kubeconfig delete pvc test-pvc
```

---

## Admin Access Control – IP Allowlist

This section restricts the Kubernetes API, Talos API, and all admin services (Longhorn UI, OpenCost UI/API, Prometheus) to known admin CIDRs only. Apply this **before** exposing any admin services.

**Replace the sample CIDRs with your real admin/VPN IPs in every block below.**

### Step 1: Calico GlobalNetworkPolicy – Protect Admin Namespaces

`GlobalNetworkPolicy` is cluster-scoped and evaluated before namespace-level `NetworkPolicy`. It is the correct Calico resource for cross-namespace, host-level access control.

```bash
kubectl --kubeconfig=kubeconfig apply -f - <<'EOF'
# Allow admin IPs inbound to platform namespaces
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: allow-admin-to-platform-services
spec:
  order: 100
  selector: >
    projectcalico.org/namespace in
    {'longhorn-system','opencost','prometheus-system'}
  ingress:
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
# Deny all other ingress to platform namespaces
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: deny-non-admin-to-platform-services
spec:
  order: 200
  selector: >
    projectcalico.org/namespace in
    {'longhorn-system','opencost','prometheus-system'}
  ingress:
    - action: Deny
  types:
    - Ingress
EOF
```

### Step 2: Calico GlobalNetworkPolicy – Protect Kubernetes API

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
    # Allow admin IPs to Kubernetes API on VIP and control plane IPs
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
    # Deny all other access to the API
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

### Step 3: Talos Host Firewall – Lock Talos API to Admin IPs

The Talos firewall is configured in the machine config and applied on each node's kernel. This is distinct from Calico and works at the host network level.

Create `talos-firewall-patch.yaml`:

```yaml
machine:
  network:
    kubespan:
      enabled: false
  # Talos host firewall (available Talos v1.6+)
  # Rules are evaluated top-to-bottom; first match wins.
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

### Step 4: Verify Admin Access Control

```bash
# List all GlobalNetworkPolicies
kubectl --kubeconfig=kubeconfig get globalnetworkpolicy

# From an admin IP — Longhorn UI should return HTTP 200
curl -s -o /dev/null -w "Longhorn UI HTTP status: %{http_code}\n" http://192.168.29.100

# From an admin IP — OpenCost API health
curl -s -o /dev/null -w "OpenCost API HTTP status: %{http_code}\n" http://192.168.29.101/healthz

# Verify Talos firewall applied on a node
talosctl --talosconfig talosconfig --nodes 192.168.29.21 get networkrulesets
```

---

## OpenCost – Cost Monitoring

> **OpenCost** is a CNCF incubating project that provides real-time Kubernetes cost allocation via a REST API and web UI. It tracks costs by namespace, pod, deployment, label, and more — backed by Prometheus.
>
> **Project**: https://github.com/opencost/opencost | **Docs**: https://opencost.io/docs | **API reference**: https://opencost.io/docs/integrations/api-examples/

### Architecture

```
Prometheus (prometheus-system)  ←  scrapes node/pod/kubelet metrics
         ↑
OpenCost exporter (opencost namespace, port 9003)
         ↓
OpenCost UI     (opencost namespace, port 9090)
OpenCost MCP    (opencost namespace, port 8081)  ← AI/automation queries
```

### Step 1: Install Prometheus

OpenCost requires Prometheus as its metrics backend. Install it with the OpenCost scrape config pre-wired.

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
  -l app.kubernetes.io/name=prometheus \
  -n prometheus-system --timeout=300s

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
    # On-prem custom pricing — adjust these to reflect your real hardware costs.
    # These values appear directly in /allocation API responses.
    customPricing:
      enabled: true
      costModel:
        description: "On-prem VMware cluster pricing"
        CPU: "0.030000"       # USD per vCPU-hour
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

  # MCP server for AI/automation integration (port 8081)
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

### Step 5: Expose OpenCost (Admin pool only)

```bash
kubectl --kubeconfig=kubeconfig apply -f - <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: opencost-lb
  namespace: opencost
  annotations:
    metallb.universe.tf/address-pool: admin-pool
spec:
  type: LoadBalancer
  loadBalancerIP: 192.168.29.101
  selector:
    app.kubernetes.io/name: opencost
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
kubectl --kubeconfig=kubeconfig get svc -n opencost opencost-lb
echo "OpenCost UI:  http://192.168.29.101    (admin IPs only)"
echo "OpenCost API: http://192.168.29.101:9003 (admin IPs only)"
```

### Step 6: OpenCost API – Complete Operations Reference

All queries target the OpenCost exporter on port `9003`. For in-cluster access use `http://opencost.opencost.svc.cluster.local:9003`.

```bash
OPENCOST_API="http://192.168.29.101:9003"

# ── Health check ──
curl -s $OPENCOST_API/healthz
# Expected: {"status":"ok"}

# ── Total allocation: last 7 days grouped by namespace ──
curl -sG $OPENCOST_API/allocation \
  -d window=7d \
  -d aggregate=namespace \
  -d accumulate=true \
  -d resolution=1m | jq '.data[0] | to_entries[] |
    {namespace: .key, totalCost: .value.totalCost}'

# ── Hourly cost breakdown: last 24h by namespace ──
curl -sG $OPENCOST_API/allocation \
  -d window=24h \
  -d step=1h \
  -d aggregate=namespace \
  -d resolution=1m | jq '.data[] | to_entries[] |
    {namespace: .key, cost: .value.totalCost}'

# ── Cost for a specific namespace (e.g. monitoring) ──
curl -sG $OPENCOST_API/allocation \
  -d window=30d \
  -d aggregate=namespace \
  -d accumulate=true \
  -d 'filter=namespace:"monitoring"' | jq '.data[0]'

# ── Deployment-level drill-down within a namespace ──
curl -sG $OPENCOST_API/allocation \
  -d window=7d \
  -d aggregate=deployment \
  -d accumulate=true \
  -d 'filter=namespace:"longhorn-system"' | jq '.data[0] | to_entries[] |
    {deployment: .key, totalCost: .value.totalCost}'

# ── Asset costs: nodes, disks, load balancers ──
curl -sG $OPENCOST_API/assets \
  -d window=7d \
  -d aggregate=type \
  -d accumulate=true | jq '.data[0] | to_entries[] |
    {type: .key, totalCost: .value.totalCost}'

# ── Cluster-wide total cost: last 30 days ──
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

# ── Current calendar month cost per namespace ──
START=$(date +%Y-%m-01T00:00:00Z)
END=$(date -u +%Y-%m-%dT%H:%M:%SZ)
curl -sG $OPENCOST_API/allocation \
  -d "window=${START},${END}" \
  -d aggregate=namespace \
  -d accumulate=true | jq '.data[0] | to_entries[] |
    {namespace: .key, monthToDateCost: .value.totalCost}'

# ── Current custom pricing config ──
curl -s $OPENCOST_API/costModel/customPricing | jq .
```

### Step 7: Automated Cost Report Script

Create `cost-report.sh`:

```bash
#!/usr/bin/env bash
# cost-report.sh — cost breakdown by namespace for any window
# Usage: ./cost-report.sh [window]   (default: 30d)
set -euo pipefail

OPENCOST_API="http://192.168.29.101:9003"
WINDOW="${1:-30d}"

echo "=== Cluster Cost Report — window: ${WINDOW} ==="
echo "Generated: $(date -u)"
echo ""

COSTS=$(curl -sG "${OPENCOST_API}/allocation" \
  -d "window=${WINDOW}" \
  -d "aggregate=namespace" \
  -d "accumulate=true" \
  -d "resolution=1m")

# Exit if API is unreachable
echo "$COSTS" | jq -e '.code == 200' > /dev/null || {
  echo "ERROR: OpenCost API returned unexpected response"
  echo "$COSTS" | jq .
  exit 1
}

printf "%-40s %12s %12s %10s %10s %12s\n" \
  "Namespace" "CPU Cost" "RAM Cost" "CPU Eff%" "RAM Eff%" "Total Cost"
printf "%-40s %12s %12s %10s %10s %12s\n" \
  "────────────────────────────────────────" "────────────" "────────────" "──────────" "──────────" "────────────"

echo "$COSTS" | jq -r '.data[0] | to_entries[] |
  [.key,
   (.value.cpuCost | tostring),
   (.value.ramCost | tostring),
   ((.value.cpuEfficiency * 100) | floor | tostring),
   ((.value.ramEfficiency * 100) | floor | tostring),
   (.value.totalCost | tostring)] | @tsv' \
  | sort -t$'\t' -k6 -rn \
  | while IFS=$'\t' read -r ns cpu ram cpueff rameff total; do
      printf "%-40s %12.4f %12.4f %9d%% %9d%% %12.4f\n" \
        "$ns" "$cpu" "$ram" "$cpueff" "$rameff" "$total"
    done

echo ""
echo "All costs in USD"
```

```bash
chmod +x cost-report.sh

# Run for last 30 days (default)
./cost-report.sh

# Run for last 7 days
./cost-report.sh 7d

# Run for current month
./cost-report.sh "$(date +%Y-%m-01T00:00:00Z),$(date -u +%Y-%m-%dT%H:%M:%SZ)"
```

### Step 8: Monthly Invoice Script (Automated via Cron)

Create `monthly-invoice.sh`:

```bash
#!/usr/bin/env bash
# monthly-invoice.sh — generate TSV invoice for the previous calendar month
# Add to crontab: 0 8 1 * * /path/to/monthly-invoice.sh
set -euo pipefail

OPENCOST_API="http://192.168.29.101:9003"
MONTH=$(date -d "last month" +%Y-%m 2>/dev/null || date -v-1m +%Y-%m)
START="${MONTH}-01T00:00:00Z"
# End: first second of current month
END="$(date +%Y-%m)-01T00:00:00Z"
OUTPUT="invoice-${MONTH}.tsv"

echo "Generating invoice for $MONTH..."

curl -sG "${OPENCOST_API}/allocation" \
  -d "window=${START},${END}" \
  -d "aggregate=namespace" \
  -d "accumulate=true" \
  -d "resolution=1m" | \
  jq -r '
    ["namespace","cpuCost","ramCost","pvCost","networkCost","totalCost"],
    (.data[0] | to_entries[] |
      [.key,
       .value.cpuCost,
       .value.ramCost,
       .value.pvCost,
       .value.networkCost,
       .value.totalCost])
    | @tsv' > "$OUTPUT"

echo "Invoice saved: $OUTPUT"
column -t "$OUTPUT"
```

```bash
chmod +x monthly-invoice.sh
./monthly-invoice.sh
```

### Verify OpenCost Installation

```bash
kubectl --kubeconfig=kubeconfig get pods -n opencost
kubectl --kubeconfig=kubeconfig get pods -n prometheus-system

# API health
curl -s http://192.168.29.101:9003/healthz
# Expected: {"status":"ok"}

# Confirm data is flowing (should return cost data, not empty)
curl -sG http://192.168.29.101:9003/allocation \
  -d window=1d \
  -d aggregate=namespace | jq '.code, (.data[0] | keys)'

# Verify Prometheus has OpenCost scrape target active
# (run from a pod with network access to prometheus-system, or via port-forward)
kubectl --kubeconfig=kubeconfig port-forward -n prometheus-system \
  svc/prometheus-server 9090:80 &
sleep 3
curl -s "http://localhost:9090/api/v1/targets" | \
  jq '.data.activeTargets[] | select(.labels.job=="opencost") | {job: .labels.job, health: .health}'
kill %1
```

---

## Verification & Testing

### Complete Cluster Health Check

```bash
echo "=== CLUSTER HEALTH CHECK ==="

echo -e "\n1. Nodes (verify static hostnames):"
kubectl --kubeconfig=kubeconfig get nodes \
  -o custom-columns='NAME:.metadata.name,IP:.status.addresses[0].address,STATUS:.status.conditions[-1].type'

echo -e "\n2. Non-running / non-completed pods:"
kubectl --kubeconfig=kubeconfig get pods -A | grep -v Running | grep -v Completed

echo -e "\n3. Storage Classes:"
kubectl --kubeconfig=kubeconfig get storageclass

echo -e "\n4. MetalLB IP Pools:"
kubectl --kubeconfig=kubeconfig get ipaddresspool -n metallb-system

echo -e "\n5. LoadBalancer Services (check IPs):"
kubectl --kubeconfig=kubeconfig get svc -A --field-selector spec.type=LoadBalancer

echo -e "\n6. Longhorn pods:"
kubectl --kubeconfig=kubeconfig get pods -n longhorn-system | grep -v Running | head -10

echo -e "\n7. Prometheus pods:"
kubectl --kubeconfig=kubeconfig get pods -n prometheus-system | grep -v Running | head -5

echo -e "\n8. OpenCost pods:"
kubectl --kubeconfig=kubeconfig get pods -n opencost

echo -e "\n9. Node resource usage:"
kubectl --kubeconfig=kubeconfig top nodes

echo -e "\n10. GlobalNetworkPolicies:"
kubectl --kubeconfig=kubeconfig get globalnetworkpolicy

echo -e "\n11. VIP reachability:"
ping -c 2 192.168.29.10

echo -e "\n12. OpenCost API health:"
curl -s http://192.168.29.101:9003/healthz
```

---

## Operations & Maintenance

### Daily Health Checks

```bash
kubectl --kubeconfig=kubeconfig get nodes
kubectl --kubeconfig=kubeconfig get pods -A | grep -v Running | grep -v Completed
kubectl --kubeconfig=kubeconfig top nodes

# Daily cost snapshot
./cost-report.sh 1d
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
cp *.yaml talosconfig kubeconfig opencost-values.yaml \
   cost-report.sh monthly-invoice.sh $BACKUP_DIR/
tar -czf ${BACKUP_DIR}.tar.gz $BACKUP_DIR/
# Encrypt before storing off-cluster
gpg --symmetric --cipher-algo AES256 ${BACKUP_DIR}.tar.gz
rm -rf $BACKUP_DIR ${BACKUP_DIR}.tar.gz
echo "Encrypted backup: ${BACKUP_DIR}.tar.gz.gpg"
```

### Kubernetes Upgrade

```bash
# Upgrade control planes one at a time
talosctl --talosconfig talosconfig --nodes 192.168.29.11 upgrade-k8s --to 1.31.0
kubectl --kubeconfig=kubeconfig get nodes

talosctl --talosconfig talosconfig --nodes 192.168.29.12 upgrade-k8s --to 1.31.0

# Drain, upgrade, uncordon each worker
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

### Upgrade OpenCost

```bash
helm upgrade opencost opencost-charts/opencost \
  --namespace opencost \
  --kubeconfig=kubeconfig \
  -f opencost-values.yaml
```

### Add a Worker Node

```bash
NEW_IP="192.168.29.24"
NEW_HOSTNAME="w04"

talosctl machineconfig patch worker.yaml --patch @worker-patch.yaml --output ${NEW_HOSTNAME}.yaml

talosctl machineconfig patch ${NEW_HOSTNAME}.yaml \
  --patch '[
    {"op":"replace","path":"/machine/network/interfaces","value":[
      {"interface":"ens192","dhcp":false,"addresses":["'"${NEW_IP}"'/24"],
       "routes":[{"network":"0.0.0.0/0","gateway":"192.168.29.1"}]}
    ]},
    {"op":"add","path":"/machine/network/hostname","value":"'"${NEW_HOSTNAME}"'"}
  ]' \
  --output ${NEW_HOSTNAME}.yaml

talosctl apply-config --insecure --nodes ${NEW_IP} --file ${NEW_HOSTNAME}.yaml
sleep 300
kubectl --kubeconfig=kubeconfig get nodes
```

---

## Optional Hardening & Improvements

These are production recommendations — not required for basic operation.

### OPT-1: cert-manager – Automatic TLS for Services

```bash
helm repo add jetstack https://charts.jetstack.io && helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --kubeconfig=kubeconfig \
  --set installCRDs=true \
  --wait

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

kubectl --kubeconfig=kubeconfig get clusterissuer
```

**Why:** Eliminates manual certificate management. Any service with an Ingress and a `cert-manager.io/cluster-issuer: letsencrypt-prod` annotation gets automatic HTTPS.

---

### OPT-2: Velero – Backup and Disaster Recovery

Backs up Kubernetes resources and PVC data to S3-compatible storage (MinIO or external).

```bash
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts && helm repo update

helm install velero vmware-tanzu/velero \
  --namespace velero \
  --create-namespace \
  --kubeconfig=kubeconfig \
  --set configuration.backupStorageLocation[0].provider=aws \
  --set configuration.backupStorageLocation[0].bucket=velero-backups \
  --set configuration.backupStorageLocation[0].config.region=minio \
  --set configuration.backupStorageLocation[0].config.s3ForcePathStyle=true \
  --set configuration.backupStorageLocation[0].config.s3Url=http://minio.minio-system.svc.cluster.local:9000 \
  --set credentials.secretContents.cloud="[default]\naws_access_key_id=REPLACE\naws_secret_access_key=REPLACE" \
  --set initContainers[0].name=velero-plugin-for-aws \
  --set initContainers[0].image=velero/velero-plugin-for-aws:v1.9.0 \
  --set initContainers[0].volumeMounts[0].mountPath=/target \
  --set initContainers[0].volumeMounts[0].name=plugins \
  --wait

# Daily backup schedule, 30-day retention
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --ttl 720h \
  --include-namespaces '*'
```

**Why:** etcd snapshots protect cluster state. Velero protects PVC data and all Kubernetes objects, enabling full cluster restore.

---

### OPT-3: Prometheus Alerting – Cost Threshold Alerts

Alert when namespace cost exceeds a budget or when quota usage reaches 80%.

```bash
kubectl --kubeconfig=kubeconfig apply -f - <<'EOF'
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cost-and-quota-alerts
  namespace: prometheus-system
spec:
  groups:
    - name: resource-quota-usage
      rules:
        - alert: NamespaceCPUQuotaHigh
          expr: |
            (kube_resourcequota{resource="limits.cpu",type="used"}
            / kube_resourcequota{resource="limits.cpu",type="hard"}) > 0.80
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Namespace {{ $labels.namespace }} is using >80% CPU quota"

        - alert: NamespaceMemoryQuotaHigh
          expr: |
            (kube_resourcequota{resource="limits.memory",type="used"}
            / kube_resourcequota{resource="limits.memory",type="hard"}) > 0.80
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Namespace {{ $labels.namespace }} is using >80% memory quota"
EOF
```

**Why:** Catches runaway workloads before they hit hard quota limits.

---

### OPT-4: Falco – Runtime Security Monitoring

Detects suspicious in-container activity: unexpected shell spawning, privilege escalation, abnormal network connections.

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts && helm repo update

helm install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  --kubeconfig=kubeconfig \
  --set driver.kind=modern_ebpf \
  --set falcosidekick.enabled=true \
  --set falcosidekick.config.slack.webhookurl="https://hooks.slack.com/REPLACE_WITH_YOUR_WEBHOOK" \
  --wait

kubectl --kubeconfig=kubeconfig get pods -n falco
```

**Why:** NetworkPolicy blocks network paths; Falco catches what happens inside running containers — the last defence against a compromised workload.

---

### OPT-5: Node Taints – Separate Platform and Workload Scheduling

```bash
# Dedicate w01 to platform components (Longhorn, OpenCost, Prometheus)
kubectl --kubeconfig=kubeconfig taint nodes w01 dedicated=platform:NoSchedule
kubectl --kubeconfig=kubeconfig label nodes w01 dedicated=platform

# w02 and w03 for general workloads
kubectl --kubeconfig=kubeconfig label nodes w02 w03 dedicated=workloads
```

Add the corresponding toleration and nodeSelector to OpenCost, Prometheus, and Longhorn via their Helm values:

```yaml
tolerations:
  - key: dedicated
    operator: Equal
    value: platform
    effect: NoSchedule
nodeSelector:
  dedicated: platform
```

**Why:** Prevents general workloads from consuming resources that platform components need to remain healthy.

---

## Troubleshooting

### Static Hostname Not Applied

```bash
# Check current hostname on a node
talosctl --talosconfig talosconfig --nodes 192.168.29.21 get hostname

# Re-apply with reboot mode to force hostname update
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
  echo "--- $node ---"
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
  echo "Disk usage on $node:"
  talosctl --talosconfig talosconfig --nodes $node df /dev/sdb
done
```

### MetalLB – Service Stuck in Pending

```bash
# Check speaker DaemonSet is running on all workers
kubectl --kubeconfig=kubeconfig get pods -n metallb-system -l component=speaker -o wide

# Check controller logs for allocation errors
kubectl --kubeconfig=kubeconfig logs -n metallb-system \
  -l component=controller --tail=100 | grep -i error

# Verify the service has the correct annotation for admin-pool services
kubectl --kubeconfig=kubeconfig get svc -n opencost opencost-lb -o yaml | grep annotation -A 3

# Check IP pools
kubectl --kubeconfig=kubeconfig get ipaddresspool,l2advertisement -n metallb-system -o yaml
```

### OpenCost – No Data / Zero Costs

```bash
# Check OpenCost pod logs
kubectl --kubeconfig=kubeconfig logs -n opencost \
  -l app.kubernetes.io/name=opencost --tail=100

# Verify Prometheus endpoint reachable from OpenCost pod
kubectl --kubeconfig=kubeconfig exec -n opencost \
  $(kubectl --kubeconfig=kubeconfig get pod -n opencost \
    -l app.kubernetes.io/name=opencost \
    -o jsonpath='{.items[0].metadata.name}') \
  -- wget -qO- \
  "http://prometheus-server.prometheus-system.svc.cluster.local/api/v1/query?query=up" \
  | head -c 300

# Test allocation API directly
curl -sG http://192.168.29.101:9003/allocation \
  -d window=1d -d aggregate=namespace | jq '.code'
# Expected: 200
```

### OpenCost – Custom Pricing Not Reflected

```bash
# Check current pricing via API
curl -s http://192.168.29.101:9003/costModel/customPricing | jq .

# If wrong, update via Helm upgrade
helm upgrade opencost opencost-charts/opencost \
  --namespace opencost \
  --kubeconfig=kubeconfig \
  -f opencost-values.yaml
```

### GlobalNetworkPolicy Not Blocking

```bash
# Verify Calico is fully running
kubectl --kubeconfig=kubeconfig get pods -n calico-system

# List all GlobalNetworkPolicies and their order
kubectl --kubeconfig=kubeconfig get globalnetworkpolicy \
  -o custom-columns='NAME:.metadata.name,ORDER:.spec.order'

# Check Felix logs for policy errors
kubectl --kubeconfig=kubeconfig logs -n calico-system \
  -l k8s-app=calico-node -c calico-node --tail=100 | grep -i "policy\|error"
```

### Storage Class Issues

```bash
kubectl --kubeconfig=kubeconfig get storageclass

# Set Longhorn as default if it lost that annotation
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
  talosctl --talosconfig talosconfig reset \
    -n $node -e $node --graceful=false --reboot
done
```

---

## Quick Reference

### Essential Commands

```bash
# Cluster
kubectl --kubeconfig=kubeconfig get nodes -o wide       # verify static hostnames
kubectl --kubeconfig=kubeconfig get pods -A | grep -v Running | grep -v Completed
talosctl --talosconfig talosconfig health

# Resources
kubectl --kubeconfig=kubeconfig top nodes
kubectl --kubeconfig=kubeconfig top pods -A --sort-by=memory

# Storage
kubectl --kubeconfig=kubeconfig get storageclass,pvc -A
kubectl --kubeconfig=kubeconfig get volumes -n longhorn-system

# Network
kubectl --kubeconfig=kubeconfig get svc -A --field-selector spec.type=LoadBalancer
kubectl --kubeconfig=kubeconfig get globalnetworkpolicy

# OpenCost
kubectl --kubeconfig=kubeconfig get pods -n opencost
curl -s http://192.168.29.101:9003/healthz
curl -sG http://192.168.29.101:9003/allocation \
  -d window=1d -d aggregate=namespace | jq '.data[0]'
./cost-report.sh
```

### Important Files

| File | Purpose |
|------|---------|
| `talosconfig` | Talos CLI authentication |
| `kubeconfig` | Kubernetes CLI authentication |
| `cp01.yaml`, `cp02.yaml` | Control plane configs (static IP + hostname) |
| `w01.yaml` – `w03.yaml` | Worker configs (static IP + hostname) |
| `cp-patch.yaml` | Control plane base patch |
| `worker-patch.yaml` | Worker base patch |
| `longhorn-disk.yaml` | Longhorn disk mount patch |
| `talos-firewall-patch.yaml` | Talos host firewall (admin IPs only) |
| `opencost-values.yaml` | OpenCost Helm values |
| `cost-report.sh` | Cost breakdown by namespace |
| `monthly-invoice.sh` | Monthly TSV invoice from OpenCost API |

### Network Reference

| Purpose | Address / Range |
|---------|----------------|
| Gateway | 192.168.29.1 |
| Kubernetes VIP | 192.168.29.10 |
| Control Planes | 192.168.29.11 – 12 |
| Workers | 192.168.29.21 – 23 |
| MetalLB Admin Pool | 192.168.29.100 – 109 |
| MetalLB General Pool | 192.168.29.110 – 199 |
| Longhorn UI | 192.168.29.100 |
| OpenCost UI / API | 192.168.29.101 |

### Port Reference

| Port | Service | Access |
|------|---------|--------|
| 6443 | Kubernetes API | Admin IPs only |
| 50000–51000 | Talos API | Admin IPs only |
| 80 | Longhorn UI | Admin IPs only (via MetalLB) |
| 80 | OpenCost UI | Admin IPs only (via MetalLB) |
| 9003 | OpenCost REST API | Admin IPs only (via MetalLB) |
| 8081 | OpenCost MCP | In-cluster only |

---

## Support Resources

- **Talos Linux**: https://www.talos.dev/ | https://www.talos.dev/v1.11/reference/configuration/
- **Kubernetes**: https://kubernetes.io/docs/
- **Calico**: https://docs.projectcalico.org/ | **GlobalNetworkPolicy**: https://docs.projectcalico.org/reference/resources/globalnetworkpolicy
- **Longhorn**: https://longhorn.io/docs/1.10.1/
- **MetalLB**: https://metallb.universe.tf/
- **OpenCost**: https://opencost.io | https://github.com/opencost/opencost | https://opencost.io/docs/integrations/api-examples/
- **cert-manager** (OPT-1): https://cert-manager.io/docs/
- **Velero** (OPT-2): https://velero.io/docs/
- **Falco** (OPT-3): https://falco.org/docs/

---

## Summary

### What We Built

✅ **HA Control Plane** — 2 nodes, VIP `192.168.29.10`, static hostnames `cp01`/`cp02`, automatic etcd failover

✅ **Worker Pool** — 3 workers with static hostnames `w01`–`w03`, isolated from control plane, dedicated Longhorn disks

✅ **Networking** — Calico v3.30.5 CNI, MetalLB v0.14.9 with separate admin (`.100–.109`) and general (`.110–.199`) IP pools

✅ **Storage** — Longhorn v1.10.1, 3× replication, RWO + RWX, auto-provisioning, UI on `192.168.29.100`

✅ **Admin Access Control** — Calico `GlobalNetworkPolicy` restricting all platform namespaces to admin CIDRs; Talos host firewall locking Talos API; MetalLB admin pool with `autoAssign: false`; Kubernetes API restricted by CIDR

✅ **OpenCost** — Real-time cost allocation, on-prem custom pricing, REST API on `192.168.29.101:9003`, UI on `192.168.29.101`, MCP on port `8081`

✅ **Automation** — `cost-report.sh` (any window), `monthly-invoice.sh` (TSV invoice from API) — both pure API-driven

✅ **Prometheus** — Dedicated instance in `prometheus-system`, Longhorn-backed persistence, pre-wired for OpenCost scrape targets

✅ **Metrics Server** — `kubectl top nodes/pods` functional

✅ **VMware Integration** — Factory image with VMware Tools, native vSphere VM support

---

**Access Points:**

| Service | URL / Endpoint | Access |
|---------|----------------|--------|
| Kubernetes API | https://29.talos.vaheed.net:6443 | Admin IPs only |
| Longhorn UI | http://192.168.29.100 | Admin IPs only |
| OpenCost UI | http://192.168.29.101 | Admin IPs only |
| OpenCost REST API | http://192.168.29.101:9003 | Admin IPs only |
| OpenCost MCP | http://opencost.opencost.svc.cluster.local:8081 | In-cluster |

---

*Last Updated: February 2026 — Version 7.0*
