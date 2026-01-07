# Talos HA Kubernetes Cluster – Production Guide v4

Complete production-ready guide for building a high-availability Talos Linux Kubernetes cluster with Longhorn distributed storage, MetalLB load balancing, and APL-Core platform.

## Cluster Specifications

- **Network**: `192.168.29.0/24`
- **Talos**: v1.11.6 with VMware Tools
- **Kubernetes**: v1.30.8
- **2 Control Planes** with VIP failover
- **3 Worker Nodes** with dedicated storage disks
- **Calico**: v3.30.5 CNI
- **MetalLB**: v0.14.9 LoadBalancer
- **Longhorn**: v1.10.1 Distributed Storage
- **Metrics Server**: Resource monitoring
- **APL-Core**: Application Platform with Cloudflare DNS

---

## Table of Contents

1. [Cluster Topology](#cluster-topology)
2. [Architecture Overview](#architecture-overview)
3. [Prerequisites](#prerequisites)
4. [Installation Steps](#installation-steps)
5. [Network Components](#network-components)
6. [Storage Configuration](#storage-configuration)
7. [APL-Core Platform](#apl-core-platform)
8. [Verification & Testing](#verification--testing)
9. [Operations & Maintenance](#operations--maintenance)
10. [Troubleshooting](#troubleshooting)

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

| Name | IP Address | Role |
|------|------------|------|
| cp01 | 192.168.29.11 | Control Plane |
| cp02 | 192.168.29.12 | Control Plane |

### Worker Nodes

| Name | IP Address | Storage Device |
|------|------------|----------------|
| w01 | 192.168.29.21 | /dev/sdb |
| w02 | 192.168.29.22 | /dev/sdb |
| w03 | 192.168.29.23 | /dev/sdb |

### Service Ranges

| Service | Range/Address |
|---------|---------------|
| Kubernetes VIP | 192.168.29.10 |
| MetalLB Pool | 192.168.29.100 - 192.168.29.199 |
| APL-Core | paas.29.talos.vaheed.net |
| Pod Network | 10.244.0.0/16 |
| Service Network | 10.96.0.0/12 |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      Gateway (192.168.29.1)                 │
└─────────────────────────────────────────────────────────────┘
                               │
                               │
┌─────────────────────────────────────────────────────────────┐
│          VIP: 192.168.29.10 (29.talos.vaheed.net)          │
│                  HA Kubernetes API Endpoint                 │
└─────────────────────────────────────────────────────────────┘
                               │
            ┌──────────────────┴──────────────────┐
            │                                      │
┌───────────────────────┐            ┌───────────────────────┐
│   cp01 (.11)          │            │   cp02 (.12)          │
│   Control Plane       │◄──────────►│   Control Plane       │
│   + etcd              │   etcd     │   + etcd              │
│   VMware Tools        │  cluster   │   VMware Tools        │
└───────────────────────┘            └───────────────────────┘
            │                                      │
            └──────────────────┬──────────────────┘
                               │
            ┌──────────────────┼──────────────────┐
            │                  │                  │
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ w01 (.21)        │  │ w02 (.22)        │  │ w03 (.23)        │
│ Worker           │  │ Worker           │  │ Worker           │
│ /dev/sdb (100GB) │  │ /dev/sdb (100GB) │  │ /dev/sdb (100GB) │
│ Longhorn OSD     │  │ Longhorn OSD     │  │ Longhorn OSD     │
└──────────────────┘  └──────────────────┘  └──────────────────┘
            │                  │                  │
            └──────────────────┼──────────────────┘
                               │
                    ┌──────────┴──────────┐
                    │  Longhorn Cluster   │
                    │  3x Replication     │
                    │  Distributed Storage│
                    └─────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Services                       │
├─────────────────────────────────────────────────────────────┤
│ • Calico CNI (Pod Networking)                               │
│ • MetalLB LoadBalancer (.100-.199)                          │
│ • Metrics Server (Resource Monitoring)                       │
│ • Longhorn Storage (Distributed Block/RWX)                   │
│ • APL-Core Platform (paas.29.talos.vaheed.net)              │
└─────────────────────────────────────────────────────────────┘
```

**Key Features:**
- **High Availability**: 2-node etcd cluster with VIP failover
- **Distributed Storage**: Longhorn with 3x replication across workers
- **Load Balancing**: MetalLB for LoadBalancer services
- **Container Networking**: Calico CNI with network policies
- **Application Platform**: APL-Core with Cloudflare DNS integration
- **VMware Integration**: Native VMware Tools support

---

## Prerequisites

### Hardware Requirements

**Control Plane Nodes (each):**
- 2+ vCPU cores
- 4GB+ RAM
- 50GB+ disk
- Network interface: ens192

**Worker Nodes (each):**
- 4+ vCPU cores
- 8GB+ RAM
- 50GB+ OS disk (/dev/sda)
- 100GB+ storage disk (/dev/sdb) for Longhorn
- Network interface: ens192

### Software Requirements

- VMware vSphere environment
- `talosctl` v1.11.6
- `kubectl` v1.30.8+
- `helm` v3.x
- Internet access for initial setup
- DNS: `29.talos.vaheed.net` → `192.168.29.10`
- DNS: `paas.29.talos.vaheed.net` → (APL-Core LoadBalancer IP)

### Network Requirements

- Subnet: `192.168.29.0/24`
- Gateway: `192.168.29.1`
- Required ports:
  - 6443: Kubernetes API
  - 50000-51000: Talos API
  - 80/443: HTTP/HTTPS ingress

### Cloudflare DNS Requirements

- Cloudflare account with Zone access
- API Token with permissions:
  - Zone:Zone:Read
  - Zone:DNS:Edit
- Account ID and Zone ID available

---

## Installation Steps

### Step 1: Download Talos with VMware Tools

```bash
# Download factory ISO with VMware Tools + iscsi tools
wget https://factory.talos.dev/image/dfd1ac9abdf529ca644694b17af0ce1a2ae23a5cccdff39439aa7f0774901e90/v1.11.6/vmware-amd64.iso \
  -O talos-vmware-1.11.6.iso
```

### Step 2: Install talosctl

```bash
# For macOS (Apple Silicon)
wget https://github.com/siderolabs/talos/releases/download/v1.11.6/talosctl-darwin-arm64
chmod +x talosctl-darwin-arm64
sudo mv talosctl-darwin-arm64 /usr/local/bin/talosctl

# For macOS (Intel)
wget https://github.com/siderolabs/talos/releases/download/v1.11.6/talosctl-darwin-amd64
chmod +x talosctl-darwin-amd64
sudo mv talosctl-darwin-amd64 /usr/local/bin/talosctl

# For Linux
wget https://github.com/siderolabs/talos/releases/download/v1.11.6/talosctl-linux-amd64
chmod +x talosctl-linux-amd64
sudo mv talosctl-linux-amd64 /usr/local/bin/talosctl

# Verify installation
talosctl version --client
```

### Step 3: Create VMs in vSphere

1. Upload `talos-vmware-1.11.6.iso` to datastore
2. Create 5 VMs:
   - **cp01, cp02**: 2 vCPU, 4GB RAM, 50GB disk
   - **w01, w02, w03**: 4 vCPU, 8GB RAM, 50GB OS disk + 100GB data disk (/dev/sdb)
3. Mount ISO and boot each VM
4. Nodes will start in maintenance mode

### Step 4: Generate Base Configuration

```bash
# Generate configurations
talosctl gen config "talos29" "https://29.talos.vaheed.net:6443" \
  --kubernetes-version v1.30.8 \
  --install-image factory.talos.dev/installer/dfd1ac9abdf529ca644694b17af0ce1a2ae23a5cccdff39439aa7f0774901e90:v1.11.6
```

**Files created:**
- `controlplane.yaml` - Base control plane config
- `worker.yaml` - Base worker config
- `talosconfig` - Talos authentication

### Step 5: Create Control Plane Patch

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

### Step 7: Generate Node Configurations

#### Control Plane cp01 (192.168.29.11)

```bash
talosctl machineconfig patch controlplane.yaml \
  --patch @cp-patch.yaml \
  --output cp01.yaml

talosctl machineconfig patch cp01.yaml \
  --patch '[{"op":"replace","path":"/machine/network/interfaces","value":[{"interface":"ens192","dhcp":false,"addresses":["192.168.29.11/24"],"routes":[{"network":"0.0.0.0/0","gateway":"192.168.29.1"}],"vip":{"ip":"192.168.29.10"}}]}]' \
  --output cp01.yaml
```

#### Control Plane cp02 (192.168.29.12)

```bash
talosctl machineconfig patch controlplane.yaml \
  --patch @cp-patch.yaml \
  --output cp02.yaml

talosctl machineconfig patch cp02.yaml \
  --patch '[{"op":"replace","path":"/machine/network/interfaces","value":[{"interface":"ens192","dhcp":false,"addresses":["192.168.29.12/24"],"routes":[{"network":"0.0.0.0/0","gateway":"192.168.29.1"}],"vip":{"ip":"192.168.29.10"}}]}]' \
  --output cp02.yaml
```

#### Worker w01 (192.168.29.21)

```bash
talosctl machineconfig patch worker.yaml \
  --patch @worker-patch.yaml \
  --output w01.yaml

talosctl machineconfig patch w01.yaml \
  --patch '[{"op":"replace","path":"/machine/network/interfaces","value":[{"interface":"ens192","dhcp":false,"addresses":["192.168.29.21/24"],"routes":[{"network":"0.0.0.0/0","gateway":"192.168.29.1"}]}]}]' \
  --output w01.yaml
```

#### Worker w02 (192.168.29.22)

```bash
talosctl machineconfig patch worker.yaml \
  --patch @worker-patch.yaml \
  --output w02.yaml

talosctl machineconfig patch w02.yaml \
  --patch '[{"op":"replace","path":"/machine/network/interfaces","value":[{"interface":"ens192","dhcp":false,"addresses":["192.168.29.22/24"],"routes":[{"network":"0.0.0.0/0","gateway":"192.168.29.1"}]}]}]' \
  --output w02.yaml
```

#### Worker w03 (192.168.29.23)

```bash
talosctl machineconfig patch worker.yaml \
  --patch @worker-patch.yaml \
  --output w03.yaml

talosctl machineconfig patch w03.yaml \
  --patch '[{"op":"replace","path":"/machine/network/interfaces","value":[{"interface":"ens192","dhcp":false,"addresses":["192.168.29.23/24"],"routes":[{"network":"0.0.0.0/0","gateway":"192.168.29.1"}]}]}]' \
  --output w03.yaml
```

### Step 8: Apply Configurations

```bash
# Apply to control planes
echo "Configuring cp01..."
talosctl apply-config --insecure --nodes 192.168.29.11 --file cp01.yaml

echo "Configuring cp02..."
talosctl apply-config --insecure --nodes 192.168.29.12 --file cp02.yaml

# Apply to workers
echo "Configuring w01..."
talosctl apply-config --insecure --nodes 192.168.29.21 --file w01.yaml

echo "Configuring w02..."
talosctl apply-config --insecure --nodes 192.168.29.22 --file w02.yaml

echo "Configuring w03..."
talosctl apply-config --insecure --nodes 192.168.29.23 --file w03.yaml

# Wait for nodes to initialize
echo "Waiting 10 minutes for nodes to initialize..."
sleep 600
```

### Step 9: Bootstrap Cluster

```bash
# Bootstrap on first control plane
echo "Bootstrapping cluster on cp01..."
talosctl --talosconfig talosconfig bootstrap \
  --endpoints 192.168.29.11 \
  --nodes 192.168.29.11

# Wait for bootstrap
echo "Waiting 5 minutes for cluster initialization..."
sleep 300

# Verify VIP
ping -c 4 192.168.29.10

# Get kubeconfig
talosctl --talosconfig talosconfig kubeconfig . \
  --nodes 192.168.29.11 \
  --endpoints 192.168.29.11 \
  --force

# Test access
kubectl --kubeconfig=kubeconfig get nodes
```

---

## Network Components

### Install Calico CNI

```bash
echo "Installing Calico CNI..."
kubectl --kubeconfig=kubeconfig apply -f \
  https://raw.githubusercontent.com/projectcalico/calico/v3.30.5/manifests/tigera-operator.yaml

kubectl --kubeconfig=kubeconfig apply -f \
  https://raw.githubusercontent.com/projectcalico/calico/v3.30.5/manifests/custom-resources.yaml

# Wait for Calico
kubectl --kubeconfig=kubeconfig wait --for=condition=ready pod \
  -l k8s-app=calico-node \
  -n calico-system \
  --timeout=300s

# Verify nodes are Ready
kubectl --kubeconfig=kubeconfig get nodes
```

### Install MetalLB

```bash
echo "Installing MetalLB..."
kubectl --kubeconfig=kubeconfig apply -f \
  https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/metallb-native.yaml

# Wait for MetalLB
kubectl --kubeconfig=kubeconfig wait --for=condition=ready pod \
  -l app=metallb \
  -n metallb-system \
  --timeout=300s

# Configure IP pool
kubectl --kubeconfig=kubeconfig apply -f - <<EOF
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
echo "Installing Metrics Server..."
kubectl --kubeconfig=kubeconfig apply -f \
  https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Patch for Talos compatibility
kubectl --kubeconfig=kubeconfig patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-preferred-address-types=InternalIP"}]'

kubectl --kubeconfig=kubeconfig patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'

# Wait for metrics server
kubectl --kubeconfig=kubeconfig wait --for=condition=ready pod \
  -l k8s-app=metrics-server \
  -n kube-system \
  --timeout=300s

# Test metrics
sleep 30
kubectl --kubeconfig=kubeconfig top nodes
```

---

## Storage Configuration

### Install Longhorn

```bash
# Add Longhorn Helm repository
helm repo add longhorn https://charts.longhorn.io
helm repo update

# Create namespace
kubectl --kubeconfig=kubeconfig create namespace longhorn-system

# Configure pod security
kubectl --kubeconfig=kubeconfig label namespace longhorn-system \
  pod-security.kubernetes.io/enforce=privileged \
  --overwrite

# Install Longhorn
helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --kubeconfig=kubeconfig \
  --version 1.10.1 \
  --set defaultSettings.defaultDataPath="/var/lib/longhorn" \
  --set defaultSettings.replicaCount=3 \
  --set persistence.defaultClass=true \
  --set persistence.defaultFsType=xfs \
  --set persistence.defaultClassReplicaCount=3

# Wait for Longhorn
echo "Waiting for Longhorn to be ready (this may take 5-10 minutes)..."
kubectl --kubeconfig=kubeconfig wait --for=condition=ready pod \
  -l app=longhorn-manager \
  -n longhorn-system \
  --timeout=600s

# Verify Longhorn installation
kubectl --kubeconfig=kubeconfig get pods -n longhorn-system
kubectl --kubeconfig=kubeconfig get storageclass
```

### Expose Longhorn UI (Optional)

```bash
# Create LoadBalancer service for Longhorn UI
kubectl --kubeconfig=kubeconfig apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: longhorn-frontend-external
  namespace: longhorn-system
spec:
  type: LoadBalancer
  selector:
    app: longhorn-ui
  ports:
    - name: http
      port: 80
      targetPort: 8000
      protocol: TCP
EOF

sleep 15
echo "Longhorn UI available at:"
kubectl --kubeconfig=kubeconfig get svc -n longhorn-system longhorn-frontend-external
```

### Test Longhorn Storage

```bash
# Create test PVC
kubectl --kubeconfig=kubeconfig apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-longhorn-pvc
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
  name: test-longhorn-pod
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
        claimName: test-longhorn-pvc
EOF

# Wait and verify
sleep 60
kubectl --kubeconfig=kubeconfig get pvc test-longhorn-pvc
kubectl --kubeconfig=kubeconfig get pod test-longhorn-pod

# Test data persistence
kubectl --kubeconfig=kubeconfig exec test-longhorn-pod -- sh -c "echo 'Longhorn test' > /data/test.txt"
kubectl --kubeconfig=kubeconfig exec test-longhorn-pod -- cat /data/test.txt

# Cleanup test resources
kubectl --kubeconfig=kubeconfig delete pod test-longhorn-pod
kubectl --kubeconfig=kubeconfig delete pvc test-longhorn-pvc
```

---

## APL-Core Platform

### Prerequisites

Before installing APL-Core, ensure you have:
- Cloudflare Account ID
- Cloudflare Zone ID for your domain
- Cloudflare API Token with permissions:
  - Zone:Zone:Read
  - Zone:DNS:Edit

```bash
kubectl --kubeconfig=kubeconfig create ns apl-system
kubectl --kubeconfig=kubeconfig create ns apl-operator
kubectl --kubeconfig=kubeconfig create ns gitea

kubectl label namespace apl-operator \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/audit=privileged \
  pod-security.kubernetes.io/warn=privileged \
  --overwrite
kubectl label namespace apl-system \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/audit=privileged \
  pod-security.kubernetes.io/warn=privileged \
  --overwrite
kubectl label namespace gitea \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/audit=privileged \
  pod-security.kubernetes.io/warn=privileged \
  --overwrite
```

### Generate an Age key if you don’t have one

```bash
apt install age -y
age-keygen -o age-key.txt
export SOPS_AGE_KEY=$(grep AGE-SECRET-KEY-1 age-key.txt)

kubectl create secret generic apl-sops-secrets \
  --from-literal=SOPS_AGE_KEY="$SOPS_AGE_KEY" \
  -n apl-operator
```

### Configure Cloudflare DNS Token

```bash
# Set your Cloudflare credentials
export CF_API_TOKEN="your-cloudflare-api-token"
```

### Create APL-Core Values File

Create `apl-values.yaml`:

```yaml
cluster:
  name: paas
  provider: custom
  domainSuffix: paas.vaheed.net
otomi:
  hasExternalDNS: true
dns:
  domainFilters:
    - vaheed.net
  provider:
    cloudflare:
      apiToken: $CF_API_TOKEN
      proxied: false
apps:
  cert-manager:
    issuer: letsencrypt
    stage: production
    email: admin@vaheed.net
```

### Install APL-Core

```bash
# Add APL-Core Helm repository
helm repo add apl https://linode.github.io/apl-core
helm repo update

# Install APL-Core
helm install apl apl/apl \
  --namespace apl-system \
  --create-namespace \
  --kubeconfig=kubeconfig \
  -f apl-values.yaml \
  --wait \
  --timeout=30m

# Wait for APL-Core to be ready
echo "Waiting for APL-Core installation to complete..."
kubectl --kubeconfig=kubeconfig wait --for=condition=ready pod \
  -l app.kubernetes.io/name=apl \
  -n apl-system \
  --timeout=1800s

# Get APL-Core console URL
echo "APL-Core Console will be available at:"
echo "https:/paas.vaheed.net"
echo ""
echo "Default credentials:"
echo "Username: otomi-admin"
echo "Password: check your values file or run:"
kubectl --kubeconfig=kubeconfig get secret apl-admin-password \
  -n apl-system \
  -o jsonpath='{.data.password}' | base64 -d
echo ""
```

### Verify APL-Core Installation

```bash
# Check all APL-Core components
kubectl --kubeconfig=kubeconfig get pods -n apl-system

# Check ingress
kubectl --kubeconfig=kubeconfig get ingress -n apl-system

# Check DNS records (should be auto-created via external-dns)
# Verify in Cloudflare dashboard that DNS records exist for:
# - paas.29.talos.vaheed.net

# Test access
curl -k https://paas.29.talos.vaheed.net
```

### APL-Core Post-Installation

1. **Access Console**: Navigate to `https://paas.vaheed.net`
2. **Login**: Use `otomi-admin` and the password from the secret
3. **Configure Teams**: Create teams for developers
4. **Setup Applications**: Deploy applications through the console
5. **Configure OIDC**: Optional - integrate with Azure Entra ID or other providers

---

## Verification & Testing

### Complete Health Check

```bash
echo "=== CLUSTER HEALTH CHECK ==="

echo -e "\n1. Node Status:"
kubectl --kubeconfig=kubeconfig get nodes -o wide

echo -e "\n2. System Pods:"
kubectl --kubeconfig=kubeconfig get pods -A | grep -E 'kube-system|calico|metallb|longhorn'

echo -e "\n3. Storage Classes:"
kubectl --kubeconfig=kubeconfig get storageclass

echo -e "\n4. MetalLB Configuration:"
kubectl --kubeconfig=kubeconfig get ipaddresspool,l2advertisement -n metallb-system

echo -e "\n5. Longhorn Status:"
kubectl --kubeconfig=kubeconfig get pods -n longhorn-system
kubectl --kubeconfig=kubeconfig get volumes -n longhorn-system

echo -e "\n6. Node Resource Usage:"
kubectl --kubeconfig=kubeconfig top nodes

echo -e "\n7. APL-Core Status:"
kubectl --kubeconfig=kubeconfig get pods -n apl-system

echo -e "\n8. VIP Status:"
ping -c 2 192.168.29.10
```

### Test MetalLB LoadBalancer

```bash
# Deploy test service
kubectl --kubeconfig=kubeconfig create deployment nginx-test --image=nginx
kubectl --kubeconfig=kubeconfig expose deployment nginx-test --port=80 --type=LoadBalancer

sleep 15
LB_IP=$(kubectl --kubeconfig=kubeconfig get svc nginx-test -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "LoadBalancer IP: $LB_IP"
curl -s http://$LB_IP | grep -o "<title>.*</title>"

# Cleanup
kubectl --kubeconfig=kubeconfig delete deployment nginx-test
kubectl --kubeconfig=kubeconfig delete svc nginx-test
```

---

## Operations & Maintenance

### Daily Health Checks

```bash
# Quick status check
kubectl --kubeconfig=kubeconfig get nodes
kubectl --kubeconfig=kubeconfig get pods -A | grep -v Running | grep -v Completed

# Resource usage
kubectl --kubeconfig=kubeconfig top nodes
kubectl --kubeconfig=kubeconfig top pods -A --sort-by=memory | head -20

# Longhorn health
kubectl --kubeconfig=kubeconfig get volumes -n longhorn-system
```

### Backup Procedures

#### etcd Backup

```bash
# Create etcd snapshot
talosctl --talosconfig talosconfig --nodes 192.168.29.11 \
  etcd snapshot /var/lib/etcd/backup-$(date +%Y%m%d-%H%M%S).db

# Download snapshot
talosctl --talosconfig talosconfig --nodes 192.168.29.11 \
  cp /var/lib/etcd/backup-*.db ./
```

#### Configuration Backup

```bash
# Backup all configuration files
mkdir -p cluster-backup-$(date +%Y%m%d)
cp *.yaml talosconfig kubeconfig apl-values.yaml cluster-backup-$(date +%Y%m%d)/
tar -czf cluster-backup-$(date +%Y%m%d).tar.gz cluster-backup-$(date +%Y%m%d)/
```

### Upgrade Procedures

#### Kubernetes Upgrade

```bash
# Upgrade control planes one at a time
talosctl --talosconfig talosconfig --nodes 192.168.29.11 upgrade-k8s --to 1.31.0
# Wait and verify
kubectl --kubeconfig=kubeconfig get nodes

talosctl --talosconfig talosconfig --nodes 192.168.29.12 upgrade-k8s --to 1.31.0

# Upgrade workers
for node in 192.168.29.21 192.168.29.22 192.168.29.23; do
  kubectl --kubeconfig=kubeconfig drain $node --ignore-daemonsets --delete-emptydir-data
  talosctl --talosconfig talosconfig --nodes $node upgrade-k8s --to 1.31.0
  sleep 120
  kubectl --kubeconfig=kubeconfig uncordon $node
done
```

#### Talos Upgrade

```bash
# Upgrade control planes
talosctl --talosconfig talosconfig --nodes 192.168.29.11 upgrade \
  --image factory.talos.dev/installer/${SCHEMATIC_ID}:v1.12.0
sleep 180

talosctl --talosconfig talosconfig --nodes 192.168.29.12 upgrade \
  --image factory.talos.dev/installer/${SCHEMATIC_ID}:v1.12.0
sleep 180

# Upgrade workers
for node in 192.168.29.21 192.168.29.22 192.168.29.23; do
  kubectl --kubeconfig=kubeconfig drain $node --ignore-daemonsets --delete-emptydir-data
  talosctl --talosconfig talosconfig --nodes $node upgrade \
    --image factory.talos.dev/installer/${SCHEMATIC_ID}:v1.12.0
  sleep 180
  kubectl --kubeconfig=kubeconfig uncordon $node
done
```

### Scaling Operations

#### Add Worker Node

```bash
# 1. Create new VM with /dev/sdb disk
# 2. Generate config for new worker (e.g., w04 at 192.168.29.24)
talosctl machineconfig patch worker.yaml \
  --patch @worker-patch.yaml \
  --output w04.yaml

talosctl machineconfig patch w04.yaml \
  --patch '[{"op":"replace","path":"/machine/network/interfaces","value":[{"interface":"ens192","dhcp":false,"addresses":["192.168.29.24/24"],"routes":[{"network":"0.0.0.0/0","gateway":"192.168.29.1"}]}]}]' \
  --output w04.yaml

# 3. Apply configuration
talosctl apply-config --insecure --nodes 192.168.29.24 --file w04.yaml

# 4. Wait and verify
sleep 300
kubectl --kubeconfig=kubeconfig get nodes
```

---

## Troubleshooting

### Nodes Not Ready

```bash
# Check kubelet status
talosctl --talosconfig talosconfig --nodes 192.168.29.21 service kubelet status

# Check CNI pods
kubectl --kubeconfig=kubeconfig get pods -n calico-system
kubectl --kubeconfig=kubeconfig logs -n calico-system -l k8s-app=calico-node --tail=50

# Check node details
kubectl --kubeconfig=kubeconfig describe node w01
```

### VIP Not Responding

```bash
# Check which node has VIP
for node in 192.168.29.11 192.168.29.12; do
  echo "Checking $node..."
  talosctl --talosconfig talosconfig --nodes $node get links | grep -A 2 ens192
done

# Check etcd health
talosctl --talosconfig talosconfig --nodes 192.168.29.11 service etcd status

# View cluster members
talosctl --talosconfig talosconfig --endpoints 192.168.29.10 get members
```

### Longhorn Issues

#### Volumes Not Attaching

```bash
# Check Longhorn manager logs
kubectl --kubeconfig=kubeconfig logs -n longhorn-system -l app=longhorn-manager --tail=100

# Check volume status
kubectl --kubeconfig=kubeconfig get volumes -n longhorn-system
kubectl --kubeconfig=kubeconfig describe volume <volume-name> -n longhorn-system

# Check node status in Longhorn
kubectl --kubeconfig=kubeconfig get nodes.longhorn.io -n longhorn-system
```

#### Replica Issues

```bash
# Check replica status
kubectl --kubeconfig=kubeconfig get replicas.longhorn.io -n longhorn-system

# Check available disk space on workers
for node in 192.168.29.21 192.168.29.22 192.168.29.23; do
  echo "Node $node:"
  talosctl --talosconfig talosconfig --nodes $node df /dev/sdb
done
```

### APL-Core Issues

#### Console Not Accessible

```bash
# Check ingress
kubectl --kubeconfig=kubeconfig get ingress -n apl-system

# Check external-dns logs
kubectl --kubeconfig=kubeconfig logs -n apl-system -l app.kubernetes.io/name=external-dns --tail=50

# Verify DNS record in Cloudflare
# Check that paas.29.talos.vaheed.net points to LoadBalancer IP

# Check cert-manager
kubectl --kubeconfig=kubeconfig get certificate -n apl-system
kubectl --kubeconfig=kubeconfig describe certificate -n apl-system
```

#### Cloudflare DNS Not Updating

```bash
# Verify Cloudflare credentials
kubectl --kubeconfig=kubeconfig get secret -n apl-system cloudflare-api-token -o yaml

# Check external-dns pod
kubectl --kubeconfig=kubeconfig get pods -n apl-system -l app.kubernetes.io/name=external-dns

# View external-dns logs for errors
kubectl --kubeconfig=kubeconfig logs -n apl-system -l app.kubernetes.io/name=external-dns --tail=100

# Verify Cloudflare token has correct permissions:
# - Zone:Zone:Read
# - Zone:DNS:Edit
```

### MetalLB Not Assigning IPs

```bash
# Check speaker pods
kubectl --kubeconfig=kubeconfig get pods -n metallb-system -l component=speaker

# Check controller logs
kubectl --kubeconfig=kubeconfig logs -n metallb-system -l component=controller --tail=100

# Verify IP pool configuration
kubectl --kubeconfig=kubeconfig get ipaddresspool,l2advertisement -n metallb-system -o yaml
```

### Storage Class Issues

```bash
# List storage classes
kubectl --kubeconfig=kubeconfig get storageclass

# Check default storage class
kubectl --kubeconfig=kubeconfig get storageclass -o jsonpath='{.items[?(@.metadata.annotations.storageclass\.kubernetes\.io/is-default-class=="true")].metadata.name}'

# Set Longhorn as default if needed
kubectl --kubeconfig=kubeconfig patch storageclass longhorn \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### Emergency Procedures

#### Complete Cluster Reset

**⚠️ WARNING: This destroys all data!**

```bash
# Delete all workloads
kubectl --kubeconfig=kubeconfig delete pvc --all -A
kubectl --kubeconfig=kubeconfig delete deployments --all -A

# Reset all nodes
for node in 192.168.29.11 192.168.29.12 192.168.29.21 192.168.29.22 192.168.29.23; do
  talosctl --talosconfig talosconfig reset -n $node -e $node --graceful=false --reboot
done
```

---

## Best Practices

### Security

1. **Change default passwords**: Update APL-Core admin password immediately
2. **Enable RBAC**: Configure proper role-based access control
3. **Network policies**: Use Calico network policies to restrict pod communication
4. **Regular updates**: Keep Talos, Kubernetes, and all components updated
5. **Backup encryption**: Encrypt backup files before storing

### Performance

1. **Resource limits**: Set appropriate CPU/memory limits on all pods
2. **Node sizing**: Size worker nodes based on workload requirements
3. **Longhorn replicas**: Use 3 replicas for critical data, 2 for non-critical
4. **Monitoring**: Enable Prometheus/Grafana for detailed metrics
5. **Storage performance**: Use SSDs for Longhorn storage disks (/dev/sdb)

### Operations

1. **Documentation**: Keep cluster configuration documented
2. **Change management**: Test changes in non-production first
3. **Monitoring**: Set up alerts for critical metrics
4. **Regular backups**: Automate daily etcd and configuration backups
5. **Disaster recovery**: Test restore procedures regularly

---

## Quick Reference

### Essential Commands

```bash
# Cluster status
kubectl --kubeconfig=kubeconfig get nodes
kubectl --kubeconfig=kubeconfig get pods -A
talosctl --talosconfig talosconfig health

# Resource usage
kubectl --kubeconfig=kubeconfig top nodes
kubectl --kubeconfig=kubeconfig top pods -A

# Storage status
kubectl --kubeconfig=kubeconfig get storageclass,pvc -A
kubectl --kubeconfig=kubeconfig get volumes -n longhorn-system

# Network services
kubectl --kubeconfig=kubeconfig get svc -A
kubectl --kubeconfig=kubeconfig get ingress -A

# APL-Core
kubectl --kubeconfig=kubeconfig get pods -n apl-system
```

### Important Files

- `talosconfig` - Talos CLI configuration
- `kubeconfig` - Kubernetes CLI configuration
- `cp01.yaml`, `cp02.yaml` - Control plane configs
- `w01.yaml`, `w02.yaml`, `w03.yaml` - Worker configs
- `apl-values.yaml` - APL-Core Helm values
- `cp-patch.yaml` - Control plane patch template
- `worker-patch.yaml` - Worker patch template

### Network Information

| Purpose | Address/Range |
|---------|---------------|
| Gateway | 192.168.29.1 |
| Kubernetes VIP | 192.168.29.10 |
| Control Planes | 192.168.29.11-12 |
| Workers | 192.168.29.21-23 |
| MetalLB Pool | 192.168.29.100-199 |
| APL-Core | paas.29.talos.vaheed.net |

### Port Reference

| Port | Service | Direction |
|------|---------|-----------|
| 6443 | Kubernetes API | Inbound |
| 50000-51000 | Talos API | Inbound |
| 80/443 | HTTP/HTTPS | Inbound |
| 8000 | Longhorn UI | Inbound (LB) |

---

## Support Resources

- **Talos Linux**: https://www.talos.dev/
- **Kubernetes**: https://kubernetes.io/docs/
- **Calico**: https://docs.projectcalico.org/
- **Longhorn**: https://longhorn.io/docs/
- **MetalLB**: https://metallb.universe.tf/
- **APL-Core**: https://techdocs.akamai.com/app-platform/docs/
- **Cloudflare API**: https://developers.cloudflare.com/api/

---

## Summary

### What We Built

✅ **High Availability Control Plane**
- 2 control plane nodes with shared VIP (192.168.29.10)
- Automatic failover for Kubernetes API
- etcd cluster for state management

✅ **Worker Pool**
- 3 dedicated worker nodes
- Workloads isolated from control plane
- Dedicated storage disks for Longhorn

✅ **Networking**
- Calico v3.30.5 CNI with network policies
- MetalLB v0.14.9 LoadBalancer (192.168.29.100-199)
- External DNS with Cloudflare integration

✅ **Distributed Storage**
- Longhorn v1.10.1 with 3x replication
- ReadWriteOnce (RWO) and ReadWriteMany (RWX) support
- Automatic volume provisioning

✅ **Application Platform**
- APL-Core for application deployment
- Cloudflare DNS automation
- Integrated monitoring and logging

✅ **Monitoring**
- Metrics Server for resource monitoring
- kubectl top commands functional
- APL-Core dashboard for platform monitoring

✅ **VMware Integration**
- Factory image with VMware Tools
- Native vSphere integration

✅ **Production Ready**
- All components tested and verified
- Backup procedures documented
- Upgrade paths defined
- Comprehensive troubleshooting guides

---

## Cluster Removal

**⚠️ WARNING: This completely destroys the cluster and all data!**

```bash
# Delete all resources
kubectl --kubeconfig=kubeconfig delete pvc --all -A
kubectl --kubeconfig=kubeconfig delete deployments --all -A
kubectl --kubeconfig=kubeconfig delete services --all -A

# Wipe all nodes
for node in 192.168.29.11 192.168.29.12 192.168.29.21 192.168.29.22 192.168.29.23; do
  echo "Wiping $node..."
  talosctl --talosconfig talosconfig reset -n $node -e $node --graceful=false --reboot
done
```

---

## 🎉 Your Cluster is Ready!

Your Talos HA Kubernetes cluster is now fully operational with:

- ✅ High availability control plane with VIP failover
- ✅ Distributed storage with Longhorn (3x replication)
- ✅ Load balancing with MetalLB
- ✅ Container networking with Calico
- ✅ Resource monitoring with Metrics Server
- ✅ Application platform with APL-Core
- ✅ Automated DNS management with Cloudflare
- ✅ VMware Tools integration

**Access Points:**
- **Kubernetes API**: https://29.talos.vaheed.net:6443
- **APL-Core Console**: https://paas.29.talos.vaheed.net
- **Longhorn UI**: http://\<loadbalancer-ip\>:80 (if exposed)

**Next Steps:**
1. Change APL-Core admin password
2. Create teams in APL-Core
3. Deploy your first application
4. Configure monitoring alerts
5. Set up automated backups

---

*Last Updated: January 2026*
*Version: 4.0*
