# k8lab — Session 02 Summary & Handover Document

**Date:** 2026-03-20
**Author:** Claude (Anthropic)
**Status:** Infrastructure design complete — ready for Stage 0 VM bootstrap and Ansible setup

---

## 1. What This Session Covered

Session 01 established the physical hardware and installed the VMs. Session 02 was entirely focused on **designing and documenting the infrastructure** that those VMs will live in — networking, tooling, and the exact steps needed before Kubernetes can be installed. No Kubernetes was touched yet. Everything in this session is prerequisite groundwork.

Topics covered in order:

- Confirmed the two-host setup (dellb + dellc) and their VM schema
- Identified the NAT networking problem and designed the bridge solution
- Defined the full static IP plan for both clusters
- Produced cheat sheets for Ubuntu dev station setup, SSH, KVM bridge networking
- Clarified the Ansible bootstrap problem (chicken-and-egg)
- Defined Stage 0 — the manual per-VM bootstrap before Ansible can run
- Explained NetworkManager vs systemd-networkd and chose NM for the hosts
- Explained network bridges, IP forwarding and Linux virtual networking concepts
- User produced a consolidated command list (`k8vm-base-config.txt`) combining all steps

---

## 2. Physical Infrastructure — Confirmed

| Property | dellb | dellc |
|---|---|---|
| Model | Dell Precision M6800 | Dell Precision 7720 |
| RAM | 32 GB | 32 GB |
| Storage | ~1 TB SSD | 5 TB SSD |
| Role | KVM bare-metal host | KVM bare-metal host (primary) |
| OS | Ubuntu 24.04.4 LTS | Ubuntu 24.04.4 LTS |
| Hypervisor | KVM/QEMU + virt-manager | KVM/QEMU + virt-manager |
| Host IP | 192.168.10.3 (static) | 192.168.10.4 (static) |
| Physical NIC | `eno1` | `enp0s31f6` |
| Network renderer | NetworkManager | NetworkManager |

Both laptops are on the same home LAN, connected to the same Mikrotik switch.

---

## 3. VM Schema — Identical on Both Hosts

| VM Hostname | Role | OS | vCPU | RAM | Disk |
|---|---|---|---|---|---|
| `k8client` | Ubuntu dev VM / Ansible control node | Ubuntu 24.04 | 2 | 4 GB | 35 GB |
| `master-01` | Kubernetes control plane | Rocky Linux 9 Minimal | 2 | 4 GB | 35 GB |
| `worker-01` | Kubernetes worker | Rocky Linux 9 Minimal | 2 | 6 GB | 70 GB |
| `worker-02` | Kubernetes worker | Rocky Linux 9 Minimal | 2 | 6 GB | 70 GB |
| `spare-01` | Monitoring (Prometheus/Grafana) | Rocky Linux 9 Minimal | 2 | 4 GB | 35 GB |

`master-02` is deferred — not included in early playbooks to keep complexity low.

---

## 4. Network Design — Final

**Router:** Mikrotik · `192.168.10.1`
**DHCP pool:** `192.168.10.150 – 192.168.10.253` (untouched)
**All static IPs:** below `.150` — no conflict possible

### The NAT Problem (why we need bridging)

By default KVM uses NAT (`virbr0`, `192.168.122.x`). VMs on dellb and VMs on dellc are each behind their own isolated NAT — they cannot reach each other across laptops, and they are not reachable from the LAN. This makes Ansible cross-host impossible.

### The Solution — br0 Bridge (NetworkManager)

A Linux bridge (`br0`) is created on each host. The physical NIC becomes a slave of the bridge. The host IP moves to `br0`. All VMs connect to `br0` instead of the NAT network. Every VM gets an IP directly on the home LAN — reachable from anywhere, including VMs on the other laptop.

NetworkManager was chosen as the renderer (not systemd-networkd) because both laptops are used as daily machines with frequent network changes (WiFi, VPN etc). NM handles this correctly; networkd does not.

### Full IP Address Plan

| Hostname | IP Address | Role | Host |
|---|---|---|---|
| — | 192.168.10.1 | Mikrotik gateway | — |
| dellb | 192.168.10.3 | KVM host | physical |
| dellc | 192.168.10.4 | KVM host (primary) | physical |
| k8client | 192.168.10.50 | Dev VM / Ansible control node | dellb |
| master-01 | 192.168.10.51 | k8s control plane | dellb |
| worker-01 | 192.168.10.52 | k8s worker | dellb |
| worker-02 | 192.168.10.53 | k8s worker | dellb |
| spare-01 | 192.168.10.54 | monitoring | dellb |
| k8client | 192.168.10.60 | Dev VM / Ansible control node | dellc |
| master-01 | 192.168.10.61 | k8s control plane | dellc |
| worker-01 | 192.168.10.62 | k8s worker | dellc |
| worker-02 | 192.168.10.63 | k8s worker | dellc |
| spare-01 | 192.168.10.64 | monitoring | dellc |
| — | 192.168.10.5 – .49 | free / future devices | — |
| — | 192.168.10.150 – .253 | Mikrotik DHCP pool | — |

---

## 5. Key Design Decisions Made This Session

| Topic | Decision | Reason |
|---|---|---|
| VM network | Bridge (`br0`) not NAT | Cross-host VM communication required |
| Bridge renderer | NetworkManager (`nmcli`) | Both laptops used as daily machines |
| SSH keypair | `ed25519`, named `lab_key` | Faster and more secure than RSA |
| Ansible control node | `k8client` VM on each host | Keeps automation isolated from bare host |
| DNS | No dedicated DNS VM | Mikrotik handles LAN DNS; CoreDNS handles k8s internal |
| SELinux | Permissive initially | Avoid blocking k8s; harden later |
| Swap | Disabled permanently | Kubernetes hard requirement |
| Python | Rocky 9 has Python 3.9 built in | No extra install needed on managed nodes |
| Editor | VSCodium | VS Code without Microsoft telemetry |
| k8s install method | `kubeadm` | Standard, minimal, production-grade |
| Container runtime | `containerd` | Lightweight, k8s-native |
| CNI plugin | Flannel or Cilium (TBD) | Decide when reaching master.yml |

---

## 6. The Ansible Bootstrap Problem — Explained

Ansible connects via SSH and requires on each managed node:

1. A reachable IP address
2. SSH running
3. A user with SSH access
4. That user has passwordless sudo
5. Python 3 installed

Rocky Linux 9 Minimal ships with SSH and Python 3 already — items 2 and 5 are free. Items 1, 3, 4 must be done manually once per VM. This is called **Stage 0**.

### Two-Stage Model

```
STAGE 0 — Manual bootstrap (done once per VM via console)
  ├─ Set static IP (nmcli)
  ├─ Set hostname (hostnamectl)
  ├─ Create sudo user
  ├─ Disable swap
  ├─ Set SELinux permissive
  └─ Copy SSH key from k8client

STAGE 1 — Ansible owns everything after this
  ├─ base.yml          (kernel params, updates, firewall)
  ├─ k8s-prereq.yml    (containerd, kubeadm, kubelet, kubectl)
  ├─ master.yml        (kubeadm init, CNI)
  └─ worker.yml        (kubeadm join)
```

---

## 7. Concepts Explained This Session

**NetworkManager vs systemd-networkd**
NM is built for desktops — dynamic, GUI-driven, handles WiFi/VPN. networkd is built for servers — static, file-driven, no GUI. Both manage network interfaces but for different contexts. We keep NM because the laptops are daily machines.

**Network bridge**
A bridge is a virtual switch. It merges multiple interfaces into one flat network. `br0` extends the home LAN into the VMs. A bridge operates at Layer 2 (MAC addresses) — it does not route between different networks, it merges them into one.

**Bridge vs Router**
A bridge connects interfaces into one network. A router connects different networks together. The Mikrotik is the router; `br0` is the bridge.

**Linux as a virtual router**
Linux can route between different networks natively using `ip_forward=1` and `iptables`. This is also a prerequisite for Kubernetes — all k8s nodes must have IP forwarding enabled.

**Linux virtual networking devices**
Linux can emulate bridges, routers, VLANs, bonded NICs, veth pairs, tun/tap devices and more in software. Kubernetes uses veth pairs and bridges internally for pod networking.

---

## 8. Current State

| Item | Status |
|---|---|
| VMs installed (Rocky 9 + k8client Ubuntu) | ✓ done — both hosts |
| Clean snapshots taken | ⏳ pending |
| br0 bridge created on hosts | ⏳ pending |
| VMs switched from NAT to br0 | ⏳ pending |
| Stage 0 bootstrap (static IP, hostname, user, swap, SELinux) | ⏳ pending |
| SSH keys generated and distributed | ⏳ pending |
| Ansible installed on k8client | ⏳ pending |
| `ansible all -m ping` returns pong | ⏳ pending |

---

## 9. What To Do Next (in order)

1. **Take clean snapshots** of all VMs before touching anything (`virsh snapshot-create-as`)
2. **Create br0 on dellb** via nmcli (`eno1` → `br0`, IP `.3`)
3. **Create br0 on dellc** via nmcli (`enp0s31f6` → `br0`, IP `.4`)
4. **Switch all VMs** from NAT to br0 (virt-manager or virsh edit)
5. **Run Stage 0 bootstrap** inside each VM console — static IP, hostname, sudo user, swap off, SELinux permissive
6. **Install Ansible** on k8client
7. **Generate SSH keypair** on k8client, distribute to all VMs
8. **Create Ansible inventory** with correct IPs
9. **Verify** `ansible all -m ping` — all nodes return pong
10. **Begin writing playbooks** — `base.yml` first

---

## 10. Notes for the Next Session

- User is learning Ansible in parallel — explain the *why* briefly, then give the code
- Always provide exact commands, not general descriptions
- Prefer `virsh`/CLI over virt-manager GUI
- dellc is the primary machine — work there first
- Both clusters use identical VM naming and same IP pattern (.50 block on dellb, .60 block on dellc)
- `master-02` is deferred — do not include in early playbooks
- Rocky 9 Minimal has Python 3 pre-installed — no prep needed on managed nodes
- k8client is Ubuntu, all other VMs are Rocky Linux 9 — commands differ between them (apt vs dnf, ssh vs sshd)
- Hungarian keyboard layout is needed on the VMs (`loadkeys hu` or `/etc/vconsole.conf`)
- User prefers NetworkManager over networkd — do not suggest switching renderers
