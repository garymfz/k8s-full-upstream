
K8S FULL Upstream Vanilla Ansible Playbook
===========
# Operative system playbook based:
Redhat
# What does this do?
Install a kubernetes cluster on a set inventory "inventory/dev/k8s.ini"
The Cluster Consist of 3 controlplanes for HA and all the workers you may want.
It also includes a role task on tech_k8s named auto-inventory.yml to automate your inventory creation based on your own provider (vsphere, nutanix, aws, gcp etc.)

# Components
Kubernetes version: 1:31
Container Engine: cri-o
Pod networking: Calico
LoadBalancer: MetalLB
Ingress Controller: Traefik
External Storage: Trident 4 netapp / Here we install helm and a metrics-server because without the ms we can't install trident.

# Kubernetes High Availability Cluster Deployment

## 🏗️ Technical Architecture Overview

This deployment creates a production-grade, highly available Kubernetes cluster utilizing modern cloud-native technologies. The architecture is designed to provide resilience, scalability, and maintainability while following best practices for enterprise deployments.

### Control Plane Components
The control plane is architected for high availability with the following core components:

#### Kubernetes Core (v1.31.x)
- **API Server**: Handles all cluster API operations
  - Configured with secure TLS communication
  - Load balanced through kube-vip
  - Horizontal scaling across multiple control plane nodes

- **etcd**: Distributed key-value store
  - Configuration: Stacked topology (co-located with control plane)
  - Automatic leader election
  - Consistent data replication across nodes
  - Backup/restore capabilities integrated

- **Controller Manager**: Manages core control loops
  - Node controller for node lifecycle management
  - Deployment controller for application scaling
  - StatefulSet controller for stateful applications
  - Custom configurations for reconciliation periods

- **Scheduler**: Handles pod placement
  - Custom scheduling policies support
  - Resource-based scheduling
  - Affinity/anti-affinity rules
  - Taints and tolerations management

#### Container Runtime (CRI-O v1.31)
Chosen for its kubernetes-native approach and security features:
- OCI-compliant container runtime
- Direct integration with Kubernetes CRI
- Optimized for Kubernetes workloads
- Advanced security features:
  - SELinux integration
  - Seccomp profiles
  - cgroups v2 support
  - Rootless container support

#### High Availability Components

##### kube-vip
Provides control plane load balancing:
- Layer 2 leader election
- ARP-based virtual IP management
- Seamless failover capabilities
- Configuration:
```yaml
vip:
  interface: <interface_name>
  address: <vip_address>
  leaderElection:
    enabled: true
    duration: 5s
  prometheus:
    enabled: true
```

##### IPVS for kube-proxy
Advanced load balancing for service traffic:
```bash
# IPVS Modules Configuration
ip_vs
ip_vs_rr  # Round Robin scheduling
ip_vs_wrr # Weighted Round Robin
ip_vs_sh  # Source Hashing
nf_conntrack

# Performance Tuning
net.ipv4.vs.conn_reuse_mode = 0
net.ipv4.vs.expire_nodest_conn = 1
net.ipv4.vs.expire_quiescent_template = 1
```

### Infrastructure Components

#### Network Layer

##### Calico CNI
Production-grade networking solution:
- BGP-based routing
- Network policy enforcement
- Enhanced IPAM capabilities
```yaml
# Calico Configuration Example
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: default-ipv4-ippool
spec:
  cidr: 10.10.0.0/16
  ipipMode: Always
  natOutgoing: true
  nodeSelector: all()
  vxlanMode: Never
```

##### MetalLB
Layer 2 load balancing for bare metal clusters:
- ARP/NDP-based announcement
- BGP support for advanced networking
- Configurable address pools
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: production-pool
  namespace: metallb-system
spec:
  addresses:
    - <ip_range_start>-<ip_range_end>
  autoAssign: true
  avoidBuggyIPs: true
```

##### Traefik v2.x Ingress Controller
Advanced ingress capabilities:
- SSL/TLS termination
- Automatic certificate management
- TCP/UDP routing
- Middleware support
```yaml
# Advanced Traefik Configuration
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: example
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`example.com`) && PathPrefix(`/api`)
      kind: Rule
      services:
        - name: api-service
          port: 8080
      middlewares:
        - name: rate-limit
        - name: circuit-breaker
  tls:
    secretName: example-tls
```

#### Storage Layer

##### NetApp Trident v24.10.0
Enterprise storage orchestration:
- Dynamic volume provisioning
- Storage class management
- Snapshot support
- QoS policies
```yaml
# Advanced Backend Configuration
{
  "version": 1,
  "storageDriverName": "ontap-nas",
  "backendName": "nas1",
  "managementLIF": "<mgmt_ip>",
  "dataLIF": "<data_ip>",
  "svm": "<svm_name>",
  "username": "<username>",
  "password": "<password>",
  "defaults": {
    "spaceAllocation": "false",
    "encryption": "false",
    "qosPolicy": "premium",
    "snapshotPolicy": "default",
    "snapshotReserve": "10",
    "size": "100Gi",
    "exportPolicy": "default"
  }
}
```

## 🔧 Detailed Component Installation and Configuration

### 1. System Preparation and Prerequisites

#### System Requirements
Detailed hardware and OS specifications for optimal performance:

```plaintext
Control Plane Nodes:
CPU:
  - Minimum: 2 cores
  - Recommended: 4 cores
  - Architecture: x86_64
RAM:
  - Minimum: 4GB
  - Recommended: 8GB
  - Swap: Disabled
Storage:
  - System: 50GB+ SSD
  - etcd: 20GB+ high-performance SSD
  - Container storage: 100GB+

Worker Nodes:
CPU:
  - Minimum: 4 cores
  - Recommended: 8+ cores
  - Architecture: x86_64
RAM:
  - Minimum: 8GB
  - Recommended: 16GB+
  - Swap: Disabled
Storage:
  - System: 100GB+
  - Container storage: 200GB+
  - Local volumes: As needed
```

#### Network Configuration
Comprehensive networking setup for cluster communication:

```bash
# Required Network Ports
# Control Plane
tcp:6443 - Kubernetes API Server
tcp:2379-2380 - etcd client/server
tcp:10250 - Kubelet API
tcp:10259 - Kube Scheduler
tcp:10257 - Kube Controller Manager

# Worker Nodes
tcp:10250 - Kubelet API
tcp:30000-32767 - NodePort Services

# Networking Parameters
sysctl -w net.ipv4.ip_forward=1
sysctl -w net.bridge.bridge-nf-call-iptables=1
sysctl -w net.ipv6.conf.all.forwarding=1
```

### 2. Core Components Installation

#### Container Runtime (CRI-O)
Detailed installation and configuration process:

```bash
# System Modules
cat > /etc/modules-load.d/crio.conf << EOF
overlay
br_netfilter
EOF

# CRI-O Configuration
cat > /etc/crio/crio.conf.d/02-crio.conf << EOF
[crio.runtime]
conmon_cgroup = "pod"
cgroup_manager = "systemd"
default_capabilities = [
    "CHOWN",
    "DAC_OVERRIDE",
    "FSETID",
    "FOWNER",
    "SETGID",
    "SETUID",
    "SETPCAP",
    "NET_BIND_SERVICE",
    "KILL"
]
EOF

# Storage Configuration
cat > /etc/containers/storage.conf << EOF
[storage]
driver = "overlay2"
runroot = "/var/run/containers/storage"
graphroot = "/var/lib/containers/storage"
[storage.options]
size = "120G"
EOF
```

#### Kubernetes Core Components
Advanced configuration for core components:

```yaml
# API Server Configuration
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
apiServer:
  extraArgs:
    authorization-mode: "Node,RBAC"
    enable-admission-plugins: "NodeRestriction,PodSecurityPolicy"
    audit-log-path: "/var/log/kubernetes/audit.log"
    audit-policy-file: "/etc/kubernetes/audit-policy.yaml"
    encryption-provider-config: "/etc/kubernetes/encryption-config.yaml"
  extraVolumes:
    - name: "audit-policy"
      hostPath: "/etc/kubernetes/audit-policy.yaml"
      mountPath: "/etc/kubernetes/audit-policy.yaml"
      readOnly: true

# Controller Manager Configuration
controllerManager:
  extraArgs:
    node-monitor-period: "5s"
    node-monitor-grace-period: "20s"
    pod-eviction-timeout: "30s"
    node-cidr-mask-size: "24"

# Scheduler Configuration
scheduler:
  extraArgs:
    address: "0.0.0.0"
    config: "/etc/kubernetes/scheduler-config.yaml"
```

### 3. High Availability Setup

#### etcd Cluster Configuration
Detailed etcd setup for high availability:

```yaml
# etcd Configuration
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
etcd:
  local:
    extraArgs:
      heartbeat-interval: "250"
      election-timeout: "1250"
      auto-compaction-retention: "8"
      snapshot-count: "10000"
    serverCertSANs:
      - "localhost"
      - "<node_ip>"
    peerCertSANs:
      - "localhost"
      - "<node_ip>"
```

#### Load Balancer Configuration
Advanced load balancing setup:

```yaml
# MetalLB L2 Configuration
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
    - production-pool
  interfaces:
    - eth0
  nodeSelectors:
    - matchLabels:
        kubernetes.io/hostname: worker1

# Advanced BGP Configuration
apiVersion: metallb.io/v1beta1
kind: BGPAdvertisement
metadata:
  name: bgp-production
  namespace: metallb-system
spec:
  ipAddressPools:
    - production-pool
  peers:
    - peer-1
    - peer-2
```
### 4. Advanced Storage Configuration

#### Trident Storage Orchestration
Detailed storage setup and configuration:

```yaml
# Storage Backend Types and Use Cases
Backends:
  ontap-nas:
    Use: General purpose NFS storage
    Features:
      - Thin provisioning
      - Snapshots
      - Clones
      - Access control
      - QoS policies
  ontap-nas-economy:
    Use: Large-scale workloads
    Features:
      - Qtrees for volumes
      - Limited snapshots
  ontap-san:
    Use: Block storage requirements
    Features:
      - iSCSI/FC support
      - Raw block volumes
      - Multipath I/O

# Volume Snapshot Class
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-snapclass
driver: csi.trident.netapp.io
deletionPolicy: Delete
parameters:
  backendType: "ontap-nas"
  snapshots: "true"
  snapmirror: "false"

# Storage Class with QoS
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: premium-ontap-nas
provisioner: csi.trident.netapp.io
parameters:
  backendType: "ontap-nas"
  media: "ssd"
  provisioningType: "thin"
  snapshots: "true"
  qosPolicy: "premium"
  adaptiveQosPolicy: "adaptive-premium"
mountOptions:
  - nfsvers=4.1
  - tcp
  - intr
  - hard
```

### 5. Service Mesh Implementation

#### Consul Service Mesh
Detailed service mesh configuration and policies:

```yaml
# Consul Service Definition
apiVersion: consul.hashicorp.com/v1alpha1
kind: ServiceDefaults
metadata:
  name: web
spec:
  protocol: http
  meshGateway:
    mode: local
  expose:
    paths:
      - path: /health
        localPathPort: 8080
        listenerPort: 21500
  upstreamConfig:
    defaults:
      connectTimeoutMs: 5000
      protocol: http2
      limits:
        maxConnections: 3000
        maxPendingRequests: 1000
        maxConcurrentRequests: 500
```
### 6. Security Implementation
#### Network Policies
Advanced network security configurations:

```yaml
# Default Deny All
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

# Allowed Communication Paths
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
```

### 7. Monitoring and Observability

#### Metrics Collection
Advanced monitoring configuration:

```yaml
# Metrics Server Configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: metrics-server-config
  namespace: kube-system
data:
  config.yaml: |
    metric-resolution: 15s
    kubelet-preferred-address-types:
    - InternalIP
    - ExternalIP
    kubelet-insecure-tls: true
    metric-resolution-raw: "15s"
    authentication-kubeconfig: "/etc/kubernetes/metrics-server/kubeconfig"
    authorization-kubeconfig: "/etc/kubernetes/metrics-server/kubeconfig"
    client-ca-file: "/etc/kubernetes/pki/ca.crt"

# Resource Metrics Pipeline
apiVersion: metrics.k8s.io/v1beta1
kind: MetricValueList
metadata:
  name: pod-metrics
spec:
  metric:
    name: container_memory_usage_bytes
    selector:
      matchLabels:
        container_name: webapp
    container: true
```

### 8. Disaster Recovery Procedures

#### Backup Configuration
Critical component backup procedures:

```bash
# etcd Backup
ETCDCTL_API=3 etcdctl snapshot save snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Validate Backup
ETCDCTL_API=3 etcdctl snapshot status snapshot.db \
  --write-out=table

# Trident Backup
tridentctl create backup -n trident
```

## 🔍 Advanced Troubleshooting

### Component-Specific Debugging

```bash
# CRI-O Debugging
crictl ps -a
crictl logs <container_id>
crictl inspect <container_id>

# etcd Cluster Health
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Network Debugging
tcpdump -i any port 6443
tcpdump -i any port 2379

# Certificate Verification
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout
```

## 📚 Additional Resources

- [Kubernetes Production Best Practices](https://kubernetes.io/docs/setup/production-environment/tools/)
- [CRI-O Documentation](https://cri-o.io/)
- [NetApp Trident GitHub](https://github.com/NetApp/trident)
- [Consul K8s Documentation](https://www.consul.io/docs/k8s)
- [MetalLB Configuration](https://metallb.universe.tf/)

#### Run the playbook (Please specify inventory)
```shell
ansible-playbook -i inventory/dev/k8s.ini playbooks/services/k8s-cluster/install/k8s-installcluster.yml
```