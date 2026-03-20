# Lab Session 01 — Summary & Handover Document
**Date:** 2026-03-19  
**Author:** Claude (Anthropic)  
**Status:** VMs installed, clean snapshots pending — ready for Ansible setup

---

## 1. Goal of This Lab

Build a minimal, resource-efficient **Kubernetes cluster** on a personal laptop running 24/7, hosting databases for software development. The entire setup will be automated with **Ansible** so it is repeatable, consistent, and never needs to be done manually twice.

---

## 2. Physical Host — Dell Precision M6800

| Property | Value |
|---|---|
| Model | Dell Precision M6800 (2014) |
| OS | Ubuntu 24.04.4 LTS x86_64 |
| Kernel | 6.17.0-14-generic |
| CPU | Intel i7-4810MQ — 4 cores / 8 threads @ 3.8 GHz |
| RAM | 32 GB DDR3 (currently ~10.6 GB used at idle with GUI) |
| GPU | Intel 4th Gen + NVIDIA Quadro K4100M |
| Hypervisor | KVM/QEMU (Type 1 in-kernel — best choice for this hardware) |
| VM Storage | 350 GB dedicated SSD reserved for VMs |

---

## 3. VM Plan — 4 Virtual Machines

All VMs run **Rocky Linux 9 — Minimal Install** (no GUI, no extras).

| Hostname | Role | vCPUs | RAM | Disk | Notes |
|---|---|---|---|---|---|
| `master-01` | Kubernetes Control Plane | 2 | 4 GB | 35 GB thin | First master, second added later |
| `master-02` | Kubernetes Control Plane | 2 | 4 GB | 35 GB thin | Add in a later session |
| `worker-01` | Kubernetes Worker Node | 2 | 6 GB | 70 GB thin | ~3 pods × 3 containers |
| `worker-02` | Kubernetes Worker Node | 2 | 6 GB | 70 GB thin | ~3 pods × 3 containers |
| `spare-01` *(optional)* | Monitoring / spare | 2 | 4 GB | 35 GB thin | Prometheus, Grafana, etc. |

**Thin provisioning** is used on all disks — actual space consumed only as needed.

### SSD Budget (350 GB)
| Allocation | Size |
|---|---|
| 4 active VMs | ~210 GB |
| Snapshots + backup headroom | ~105 GB |
| Buffer | ~35 GB |

### RAM Budget (32 GB)
| Allocation | RAM |
|---|---|
| Ubuntu host at idle | ~11 GB |
| master-01 | 4 GB |
| worker-01 + worker-02 | 12 GB |
| spare-01 | 4 GB |
| **Headroom** | ~1 GB |

KVM does not pre-allocate RAM — memory ballooning means actual usage is lower at idle.

---

## 4. Current State of the VMs

- All 4 VMs are **installed with Rocky Linux 9 Minimal**
- Nothing additional is installed or configured on any VM
- **Next immediate step:** Take a clean snapshot of each VM before any changes
- After snapshots: begin configuration via Ansible

---

## 5. Automation Strategy — Ansible

**Control node:** Ubuntu host (the laptop itself)  
**Managed nodes:** All 4 Rocky Linux VMs via SSH  
**No agent required** on the VMs — Ansible only needs SSH + Python (Python is included in Rocky 9 minimal)

### Planned Ansible Playbooks

| Playbook | Target | Purpose |
|---|---|---|
| `base.yml` | all nodes | Hostname, timezone, updates, swap off, SELinux permissive, kernel params |
| `k8s-prereq.yml` | all nodes | Kernel modules (overlay, br_netfilter), sysctl, containerd, kubeadm, kubelet, kubectl |
| `master.yml` | masters group | `kubeadm init`, CNI plugin install, kubeconfig setup |
| `worker.yml` | workers group | `kubeadm join` with token from master |

### Ansible Inventory Structure (planned)
```ini
[masters]
master-01 ansible_host=192.168.122.10

[workers]
worker-01 ansible_host=192.168.122.11
worker-02 ansible_host=192.168.122.12

[all:vars]
ansible_user=<admin_user>
ansible_ssh_private_key_file=~/.ssh/lab_key
```

> IP addresses above are placeholders — actual IPs to be noted after VMs are configured with static IPs.

---

## 6. Kubernetes Design Decisions

| Topic | Decision | Reason |
|---|---|---|
| Install method | `kubeadm` | Standard, minimal, production-grade |
| Container runtime | `containerd` | Lightweight, k8s-native, no Docker overhead |
| CNI plugin | Flannel or Cilium (TBD) | Flannel = simple; Cilium = more powerful |
| Swap | Disabled permanently | Kubernetes hard requirement |
| SELinux | Permissive initially | Avoid blocking k8s; harden later |
| Firewall | `nftables` explicit rules | Replace `firewalld` for precise k8s port control |
| HA Control Plane | 2 masters (later) | `master-02` added once `master-01` is stable |

### Worker Pod Sizing Basis
Each worker hosts ~3 pods × 3 containers = 9 containers, primarily **database workloads** (PostgreSQL, MySQL, Redis, etc.). Databases are RAM-intensive, which is why workers are allocated 6 GB each.

---

## 7. What To Do Next (in order)

1. **Snapshot all 4 VMs** in their clean minimal state via `virt-manager` or `virsh snapshot-create`
2. **Install Ansible** on the Ubuntu host: `sudo apt install ansible`
3. **Generate SSH keypair** on the host and distribute to all 4 VMs: `ssh-copy-id`
4. **Create Ansible inventory** file with hostnames and IPs
5. **Write and run `base.yml`** — first playbook: swap off, SELinux, hostname, updates
6. **Write and run `k8s-prereq.yml`** — containerd + kubeadm stack
7. **Write and run `master.yml`** — initialize the cluster
8. **Write and run `worker.yml`** — join workers to the cluster
9. **Verify cluster** with `kubectl get nodes`
10. **Plan database deployments** — which databases, which namespaces, persistent volumes

---

## 8. Notes for the Next Claude Session

- The user knows their hardware well and wants **precise, no-waste** solutions
- Always provide **exact commands**, not general descriptions
- The user is learning Ansible in parallel — explain the *why* briefly, then give the code
- Prefer **virsh/CLI** over virt-manager GUI for VM operations (consistent with minimal philosophy)
- Rocky Linux 9 minimal has **Python 3 pre-installed** — Ansible will work without extra prep on managed nodes
- The user runs this machine **24/7** — power efficiency and stability matter
- When IPs are assigned, update the inventory file and note them in the next session summary
- The second master node (`master-02`) is **deferred** — do not include it in early playbooks to keep complexity low
