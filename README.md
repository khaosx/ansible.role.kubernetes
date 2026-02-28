# Ansible Role: kubernetes

Deploy a production-ready, highly-available Kubernetes cluster using kubeadm with full networking, storage, and TLS capabilities.

## Overview

This role deploys a complete Kubernetes cluster with:

- **HA Control Plane**: 3 control plane nodes with keepalived VIP failover
- **Worker Nodes**: 3 dedicated worker nodes
- **CNI**: Cilium for pod networking
- **Load Balancer**: MetalLB for bare-metal LoadBalancer services
- **Ingress**: Traefik ingress controller
- **TLS**: cert-manager with Let's Encrypt and Cloudflare DNS-01 challenge
- **Block Storage**: Ceph RBD CSI driver (default StorageClass)
- **File Storage**: NFS CSI driver for NAS mounts

## Requirements

- Ubuntu 24.04 on all nodes
- Minimum 4GB RAM per node (8GB+ recommended for control plane)
- Network connectivity between all nodes
- Ansible 2.15+
- Collections: `community.general`, `ansible.posix`
- Role contract: `requires_become: true`

### External Dependencies

- **Ceph cluster**: For block storage (Proxmox Ceph or standalone)
- **NFS server**: For file storage (Synology, TrueNAS, etc.)
- **Cloudflare**: For DNS-01 TLS challenges (domain must be hosted on Cloudflare)

## Dependencies

- Role: `ufw_profiles`
- Collection: `community.general`
- Collection: `ansible.posix`

## Role Variables

### Kubernetes Core

```yaml
kubernetes_version: "1.32"
kubernetes_apt_version: "1.32.*"
kubernetes_cluster_name: "k8s-clst-01"
kubernetes_pod_network_cidr: "10.244.0.0/16"
kubernetes_service_cidr: "10.96.0.0/12"
```

### Control Plane HA

```yaml
kubernetes_control_plane_endpoint: "10.0.20.246:6443"  # VIP:port
kubernetes_control_plane_vip: "10.0.20.246"            # VIP address
kubernetes_primary_control_plane: "ctrl-01"            # First node to initialize
kubernetes_keepalived_password: "{{ vault_kubernetes_keepalived_password }}"
```

### CNI (Cilium)

```yaml
kubernetes_cni: "cilium"
kubernetes_cilium_version: "1.17.0"
```

### MetalLB

```yaml
kubernetes_metallb_enabled: true
kubernetes_metallb_version: "0.14.9"
kubernetes_metallb_address_range: "10.0.20.30-10.0.20.39"
```

### Traefik Ingress

```yaml
kubernetes_traefik_enabled: true
kubernetes_traefik_version: "34.3.0"
kubernetes_traefik_loadbalancer_ip: "10.0.20.30"
kubernetes_cluster_domain: "khaosx.io"
kubernetes_traefik_dashboard_host: "traefik.{{ kubernetes_cluster_domain }}"
kubernetes_traefik_dashboard_tls_secret_name: "traefik-dashboard-tls"
kubernetes_traefik_dashboard_tls_enabled: true
```

### cert-manager

```yaml
kubernetes_cert_manager_enabled: true
kubernetes_cert_manager_version: "1.17.1"
kubernetes_cert_manager_email: "{{ vault_configure_ssl_email }}"
kubernetes_cert_manager_cloudflare_api_token: "{{ vault_configure_ssl_cloudflare_api_token }}"
kubernetes_cert_manager_dns_zone: "{{ vault_site_domain }}"
kubernetes_traefik_dashboard_certificate_enabled: true
kubernetes_traefik_dashboard_certificate_issuer: "letsencrypt-prod"
```

### Ceph Storage

```yaml
kubernetes_ceph_enabled: true
kubernetes_ceph_cluster_id: "7ecb9951-45cd-4825-af85-d70faea45ea6"
kubernetes_ceph_monitors:
  - "10.0.30.101:6789"
  - "10.0.30.102:6789"
  - "10.0.30.103:6789"
kubernetes_ceph_rbd_pool: "rbd_data"
kubernetes_ceph_user: "kubernetes"
kubernetes_ceph_user_key: "{{ vault_kubernetes_ceph_user_key }}"
kubernetes_ceph_default_storage_class: true
```

### NFS Storage

```yaml
kubernetes_nfs_enabled: true
kubernetes_nfs_csi_version: "v4.10.0"
```

## Inventory Requirements

Nodes must be assigned to groups and have `kubernetes_role` set:

```yaml
kubernetes:
  children:
    kubernetes_control_plane:
    kubernetes_workers:

kubernetes_control_plane:
  hosts:
    ctrl-01:
      ansible_host: 10.0.20.81
      kubernetes_role: control_plane
    ctrl-02:
      ansible_host: 10.0.20.82
      kubernetes_role: control_plane
    ctrl-03:
      ansible_host: 10.0.20.83
      kubernetes_role: control_plane

kubernetes_workers:
  hosts:
    work-01:
      ansible_host: 10.0.20.84
      kubernetes_role: worker
    work-02:
      ansible_host: 10.0.20.85
      kubernetes_role: worker
    work-03:
      ansible_host: 10.0.20.86
      kubernetes_role: worker
```

## Vault Variables Required

Define these in your Ansible Vault:

```yaml
vault_kubernetes_keepalived_password:
vault_kubernetes_ceph_user_key:
vault_configure_ssl_email:
vault_configure_ssl_cloudflare_api_token:
vault_site_domain:
```

`kubernetes_traefik_loadbalancer_ip` must be within `kubernetes_metallb_address_range`.

## Tags

Run specific components using tags:

| Tag | Description |
|-----|-------------|
| `kubernetes` | Full role |
| `kubernetes_set_vars` | Fact loading and role variable prep |
| `kubernetes_firewall` | UFW configuration |
| `kubernetes_prerequisites` | System prerequisites |
| `kubernetes_containerd` | Container runtime |
| `kubernetes_install` | Kubernetes package install |
| `kubernetes_keepalived` | VIP failover |
| `kubernetes_init` | Cluster initialization |
| `kubernetes_join` | Node joining |
| `kubernetes_cni` | Cilium CNI |
| `kubernetes_metallb` | MetalLB load balancer |
| `kubernetes_traefik` | Traefik ingress |
| `kubernetes_cert_manager` | TLS certificate management |
| `kubernetes_ceph` | Ceph CSI storage |
| `kubernetes_nfs` | NFS CSI storage |
| `kubernetes_storage` | All storage (Ceph + NFS) |
| `kubernetes_ha` | HA components (keepalived) |

### Examples

```bash
# Full cluster deployment
ansible-playbook site.yml --tags kubernetes

# Just storage drivers
ansible-playbook site.yml --tags kubernetes_storage

# Just ingress and TLS
ansible-playbook site.yml --tags kubernetes_traefik,kubernetes_cert_manager
```

## Playbook Configuration

The playbook must use `serial: 1` for proper node ordering:

```yaml
- name: Deploy Kubernetes Cluster
  hosts: kubernetes
  gather_facts: true
  become: true
  serial: 1
  roles:
    - role: khaosx.homelab.kubernetes
      tags: [kubernetes, k8s]
```

## Outbound Artifacts

- System packages and services: `containerd`, `kubelet`, `kubeadm`, `kubectl`, optional `keepalived`
- Kubernetes control-plane and node join state under `/etc/kubernetes/`
- Generated kubeconfigs at `/root/.kube/config` and `{{ ansible_user_dir }}/.kube/config`
- Installed cluster components (conditional): Cilium, MetalLB, Traefik, cert-manager, Ceph CSI, NFS CSI
- Firewall rules applied through `ufw_profiles`

## Idempotency

true

## Atomic

false

## Rollback

No automated rollback is implemented. For rollback, reset nodes with `kubeadm reset` and redeploy.

## Architecture

```
                    ┌─────────────────────────────────────────┐
                    │           VIP: 10.0.20.246              │
                    │         (keepalived failover)           │
                    └──────────────────┬──────────────────────┘
                                       │
        ┌──────────────────────────────┼──────────────────────────────┐
        │                              │                              │
   ┌────┴────┐                   ┌─────┴────┐                   ┌─────┴────┐
   │ ctrl-01 │                   │ ctrl-02  │                   │ ctrl-03  │
   │ MASTER  │                   │ BACKUP   │                   │ BACKUP   │
   │ :6443   │                   │ :6443    │                   │ :6443    │
   └─────────┘                   └──────────┘                   └──────────┘
        │                              │                              │
        └──────────────────────────────┼──────────────────────────────┘
                                       │
        ┌──────────────────────────────┼──────────────────────────────┐
        │                              │                              │
   ┌────┴────┐                   ┌─────┴────┐                   ┌─────┴────┐
   │ work-01 │                   │ work-02  │                   │ work-03  │
   │ worker  │                   │ worker   │                   │ worker   │
   └─────────┘                   └──────────┘                   └──────────┘

   LoadBalancer IPs: 10.0.20.30-39 (MetalLB)
   Traefik Ingress:  10.0.20.30
```

## Post-Installation

After deployment, on any control plane node:

```bash
# Verify nodes
kubectl get nodes

# Verify system pods
kubectl get pods -n kube-system

# Verify storage classes
kubectl get storageclass

# Verify ingress
kubectl get svc -n traefik

# Verify cert-manager
kubectl get clusterissuers
```

## Creating Workloads

### PersistentVolumeClaim (Ceph RBD)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myapp-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ceph-rbd
  resources:
    requests:
      storage: 10Gi
```

### NFS PersistentVolume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: myapp-nfs
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  csi:
    driver: nfs.csi.k8s.io
    volumeHandle: nas.example.com/path/to/export#myapp
    volumeAttributes:
      server: nas.example.com
      share: /path/to/export
```

### TLS Certificate

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: myapp-tls
spec:
  secretName: myapp-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - myapp.khaosx.io
```

### Ingress with TLS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: traefik
  tls:
    - hosts:
        - myapp.khaosx.io
      secretName: myapp-tls
  rules:
    - host: myapp.khaosx.io
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 80
```

## Resetting the Cluster

If the deployment fails partway through, reset all nodes before retrying:

```bash
# On each node (control planes first, then workers):
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d /var/lib/etcd /etc/kubernetes/manifests/*
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X
```

## License

MIT

## Author Notes

Generated with the assistance of AI
