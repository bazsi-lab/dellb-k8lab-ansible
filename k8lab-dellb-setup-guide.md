# K8Lab – Kubernetes Cluster Setup via Ansible
### Host: DELLB (Dell Precision M6800) | Ubuntu 24.04 LTS | KVM/QEMU

---

## 📋 Overview

This document covers the complete setup of a single-host Kubernetes lab cluster on **DELLB**,
using **Ansible** for automated provisioning. The cluster consists of 1 control plane node and
3 worker nodes, all running as KVM virtual machines.

---

## 🖥️ Infrastructure

| Role | Hostname | IP | OS | User |
|---|---|---|---|---|
| KVM Host | DELLB | 192.168.10.3 | Ubuntu 24.04 LTS | dextr |
| Ansible Control Node | dellb-k8client | 192.168.10.58 | Ubuntu 24.04 LTS | k8client |
| Control Plane | dellb-master-01 | 192.168.10.59 | Rocky Linux 9 Minimal | kubermaster |
| Worker 01 | dellb-worker-01 | 192.168.10.51 | Rocky Linux 9 Minimal | kuber |
| Worker 02 | dellb-worker-02 | 192.168.10.52 | Rocky Linux 9 Minimal | kuber |
| Worker 03 | dellb-worker-03 | 192.168.10.53 | Rocky Linux 9 Minimal | kuber |

**Network:** Mikrotik router at 192.168.10.1, DHCP pool .150–.253, VMs use static IPs.

**VM disk images location on DELLB:** `/mnt/VM/virtVM/`

---

## 📁 Project Structure

```
~/groot-dellb-k8client/dellb-k8lab-ansible-books/
├── ansible.cfg
├── inventory.ini
├── base.yml               # Base OS prep – masters
├── base-workers.yml       # Base OS prep – workers
├── k8s-prereq.yml         # K8s prerequisites – masters
├── k8s-prereq-workers.yml # K8s prerequisites – workers
├── master.yml             # Control plane init
└── workers.yml            # Worker join
```

---

## ⚙️ Ansible Configuration

### `ansible.cfg`
```ini
[defaults]
inventory         = inventory.ini

private_key_file  = ~/.ssh/id_ed25519_dellb-k8client_all
host_key_checking = False

[privilege_escalation]
become        = True
become_method = sudo
become_user   = root
```

### `inventory.ini`
```ini
[masters]
dellb-master-01 ansible_host=192.168.10.59 ansible_user=kubermaster

[workers]
dellb-worker-01 ansible_host=192.168.10.51 ansible_user=kuber
dellb-worker-02 ansible_host=192.168.10.52 ansible_user=kuber
dellb-worker-03 ansible_host=192.168.10.53 ansible_user=kuber
```

---

## 📜 Playbooks

### 1. `base.yml` — Base OS Configuration (Masters)

Targets `[masters]`. Prepares the OS for Kubernetes:
- Full package update via `dnf`
- Disable swap (runtime + permanent via `/etc/fstab`)
- Set SELinux to permissive (runtime + permanent)
- Add master IP/hostname to `/etc/hosts`
- Open firewalld ports: `6443/tcp` (API server), `10250/tcp` (kubelet)
- Load kernel modules: `overlay`, `br_netfilter` (runtime + boot)
- Apply sysctl parameters for bridge networking and IP forwarding

```yaml
---
- name: Base OS configuration for Kubernetes nodes
  hosts: masters
  become: true

  tasks:

    - name: Update all packages
      ansible.builtin.dnf:
        name: "*"
        state: latest

    - name: Disable swap immediately
      ansible.builtin.command: swapoff -a
      changed_when: false

    - name: Remove swap from /etc/fstab permanently
      ansible.builtin.replace:
        path: /etc/fstab
        regexp: '^([^#].*\sswap\s.*)$'
        replace: '#\1'

    - name: Set SELinux to permissive (runtime)
      ansible.builtin.command: setenforce 0
      ignore_errors: true

    - name: Set SELinux to permissive (permanent)
      ansible.builtin.lineinfile:
        path: /etc/selinux/config
        regexp: '^SELINUX='
        line: 'SELINUX=permissive'

    - name: Add hostname to /etc/hosts
      ansible.builtin.lineinfile:
        path: /etc/hosts
        line: "192.168.10.59 dellb-master-01"
        state: present

    - name: Open firewalld port 6443
      ansible.posix.firewalld:
        port: 6443/tcp
        permanent: true
        state: enabled
        immediate: true

    - name: Open firewalld port 10250
      ansible.posix.firewalld:
        port: 10250/tcp
        permanent: true
        state: enabled
        immediate: true

    - name: Load kernel modules now
      community.general.modprobe:
        name: "{{ item }}"
        state: present
      loop:
        - overlay
        - br_netfilter

    - name: Make kernel modules load at boot
      ansible.builtin.copy:
        dest: /etc/modules-load.d/kubernetes.conf
        content: |
          overlay
          br_netfilter

    - name: Set required sysctl parameters
      ansible.builtin.copy:
        dest: /etc/sysctl.d/kubernetes.conf
        content: |
          net.bridge.bridge-nf-call-iptables  = 1
          net.bridge.bridge-nf-call-ip6tables = 1
          net.ipv4.ip_forward                 = 1

    - name: Apply sysctl parameters
      ansible.builtin.command: sysctl --system
      changed_when: false
```

---

### 2. `base-workers.yml` — Base OS Configuration (Workers)

Identical to `base.yml` but targets `[workers]`. Firewall differences:
- **Workers open:** `10250/tcp` (kubelet), `30000-32767/tcp` (NodePort range)
- **Workers do NOT open:** `6443/tcp` (that's only for the API server on master)

```yaml
---
- name: Base OS configuration for Kubernetes worker nodes
  hosts: workers
  become: true

  tasks:

    - name: Update all packages
      ansible.builtin.dnf:
        name: "*"
        state: latest

    - name: Disable swap immediately
      ansible.builtin.command: swapoff -a
      changed_when: false

    - name: Remove swap from /etc/fstab permanently
      ansible.builtin.replace:
        path: /etc/fstab
        regexp: '^([^#].*\sswap\s.*)$'
        replace: '#\1'

    - name: Set SELinux to permissive (runtime)
      ansible.builtin.command: setenforce 0
      ignore_errors: true

    - name: Set SELinux to permissive (permanent)
      ansible.builtin.lineinfile:
        path: /etc/selinux/config
        regexp: '^SELINUX='
        line: 'SELINUX=permissive'

    - name: Add master hostname to /etc/hosts
      ansible.builtin.lineinfile:
        path: /etc/hosts
        line: "192.168.10.59 dellb-master-01"
        state: present

    - name: Open firewalld port 10250 (kubelet API)
      ansible.posix.firewalld:
        port: 10250/tcp
        permanent: true
        state: enabled
        immediate: true

    - name: Open firewalld NodePort range
      ansible.posix.firewalld:
        port: 30000-32767/tcp
        permanent: true
        state: enabled
        immediate: true

    - name: Load kernel modules now
      community.general.modprobe:
        name: "{{ item }}"
        state: present
      loop:
        - overlay
        - br_netfilter

    - name: Make kernel modules load at boot
      ansible.builtin.copy:
        dest: /etc/modules-load.d/kubernetes.conf
        content: |
          overlay
          br_netfilter

    - name: Set required sysctl parameters
      ansible.builtin.copy:
        dest: /etc/sysctl.d/kubernetes.conf
        content: |
          net.bridge.bridge-nf-call-iptables  = 1
          net.bridge.bridge-nf-call-ip6tables = 1
          net.ipv4.ip_forward                 = 1

    - name: Apply sysctl parameters
      ansible.builtin.command: sysctl --system
      changed_when: false
```

---

### 3. `k8s-prereq.yml` — Kubernetes Prerequisites (Masters)

Targets `[masters]`. Installs and configures the container runtime and Kubernetes binaries:
- Install `conntrack-tools`
- Add Docker CE repo (provides `containerd.io`)
- Install and configure `containerd` with `SystemdCgroup = true`
- Add Kubernetes yum repo (v1.31 stable)
- Install `kubeadm`, `kubelet`, `kubectl`
- Enable and start `kubelet`

```yaml
---
- name: Install Kubernetes prerequisites
  hosts: masters
  become: true

  tasks:

    - name: Install conntrack-tools
      ansible.builtin.dnf:
        name: conntrack-tools
        state: present

    - name: Add Docker repo (provides containerd)
      ansible.builtin.get_url:
        url: https://download.docker.com/linux/centos/docker-ce.repo
        dest: /etc/yum.repos.d/docker-ce.repo

    - name: Install containerd
      ansible.builtin.dnf:
        name: containerd.io
        state: present

    - name: Remove old containerd config
      ansible.builtin.file:
        path: /etc/containerd/config.toml
        state: absent

    - name: Generate fresh containerd config
      ansible.builtin.shell: containerd config default > /etc/containerd/config.toml

    - name: Enable SystemdCgroup in containerd config
      ansible.builtin.replace:
        path: /etc/containerd/config.toml
        regexp: 'SystemdCgroup = false'
        replace: 'SystemdCgroup = true'

    - name: Remove disabled_plugins line if present
      ansible.builtin.lineinfile:
        path: /etc/containerd/config.toml
        regexp: '^disabled_plugins'
        state: absent

    - name: Enable and start containerd
      ansible.builtin.service:
        name: containerd
        state: restarted
        enabled: true

    - name: Add Kubernetes yum repo
      ansible.builtin.copy:
        dest: /etc/yum.repos.d/kubernetes.repo
        content: |
          [kubernetes]
          name=Kubernetes
          baseurl=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/
          enabled=1
          gpgcheck=1
          gpgkey=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/repodata/repomd.xml.key

    - name: Install kubeadm, kubelet, kubectl
      ansible.builtin.dnf:
        name:
          - kubeadm
          - kubelet
          - kubectl
        state: present

    - name: Enable kubelet service
      ansible.builtin.service:
        name: kubelet
        enabled: true
        state: started
```

---

### 4. `k8s-prereq-workers.yml` — Kubernetes Prerequisites (Workers)

Identical to `k8s-prereq.yml` but targets `[workers]`. Exact same tasks.

---

### 5. `master.yml` — Control Plane Initialization

Targets `[masters]`. Initializes the Kubernetes control plane:
- Runs `kubeadm init` with pod CIDR `10.244.0.0/16` (Flannel)
- Creates `.kube/config` for the `kubermaster` user
- Installs **Flannel CNI** for pod networking

```yaml
---
- name: Initialize Kubernetes control plane
  hosts: masters
  become: true

  vars:
    pod_cidr: "10.244.0.0/16"
    apiserver_ip: "192.168.10.59"
    kube_user: "kubermaster"

  tasks:

    - name: Initialize the control plane
      ansible.builtin.command: >
        kubeadm init
        --apiserver-advertise-address={{ apiserver_ip }}
        --pod-network-cidr={{ pod_cidr }}
        --upload-certs
      args:
        creates: /etc/kubernetes/admin.conf

    - name: Create .kube directory for user
      ansible.builtin.file:
        path: /home/{{ kube_user }}/.kube
        state: directory
        owner: "{{ kube_user }}"
        group: "{{ kube_user }}"
        mode: '0755'

    - name: Copy admin.conf to user kubeconfig
      ansible.builtin.copy:
        src: /etc/kubernetes/admin.conf
        dest: /home/{{ kube_user }}/.kube/config
        remote_src: true
        owner: "{{ kube_user }}"
        group: "{{ kube_user }}"
        mode: '0600'

    - name: Install Flannel CNI
      ansible.builtin.command: >
        kubectl apply -f
        https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
      environment:
        KUBECONFIG: /etc/kubernetes/admin.conf
      changed_when: false
```

---

### 6. `workers.yml` — Join Workers to Cluster

Two plays in one file:
1. **Play 1** — connects to master, generates a fresh join token via `kubeadm token create --print-join-command`, stores it as a host fact
2. **Play 2** — connects to each worker, checks if already joined (via `/etc/kubernetes/kubelet.conf`), joins if not → **fully idempotent**

```yaml
---
- name: Fetch join command from control plane
  hosts: masters
  become: true

  tasks:

    - name: Generate kubeadm join command
      ansible.builtin.command: kubeadm token create --print-join-command
      register: join_cmd
      changed_when: false

    - name: Store join command as fact
      ansible.builtin.set_fact:
        join_command: "{{ join_cmd.stdout }}"

- name: Join worker nodes to the cluster
  hosts: workers
  become: true

  tasks:

    - name: Check if node is already joined
      ansible.builtin.stat:
        path: /etc/kubernetes/kubelet.conf
      register: kubelet_conf

    - name: Join the cluster
      ansible.builtin.command: "{{ hostvars['dellb-master-01']['join_command'] }}"
      when: not kubelet_conf.stat.exists
      register: join_result

    - name: Show join result
      ansible.builtin.debug:
        msg: "{{ join_result.stdout_lines | default(['Node already part of cluster — skipped']) }}"
```

---

## 🚀 Execution Order

```bash
# Step 1 – Master base OS prep
ansible-playbook base.yml

# Step 2 – Master K8s prerequisites
ansible-playbook k8s-prereq.yml

# Step 3 – Initialize control plane
ansible-playbook master.yml

# Step 4 – Worker base OS prep
ansible-playbook base-workers.yml

# Step 5 – Worker K8s prerequisites
ansible-playbook k8s-prereq-workers.yml

# Step 6 – Join workers to cluster
ansible-playbook workers.yml
```

---

## ✅ Cluster Verification

```bash
ssh dellb-master-01
kubectl get nodes
```

**Expected output:**
```
NAME              STATUS   ROLES           AGE     VERSION
dellb-master-01   Ready    control-plane   51m     v1.31.14
dellb-worker-01   Ready    <none>          2m43s   v1.31.14
dellb-worker-02   Ready    <none>          2m27s   v1.31.14
dellb-worker-03   Ready    <none>          2m42s   v1.31.14
```

**Additional checks:**
```bash
# All system pods running
kubectl get pods -n kube-system

# Node resource usage
kubectl describe nodes | grep -A5 "Allocated resources"
```

---

## 🔑 Firewall Port Reference

| Port | Protocol | Node | Purpose |
|---|---|---|---|
| 6443 | TCP | Master | Kubernetes API server |
| 10250 | TCP | All | Kubelet API |
| 30000–32767 | TCP | Workers | NodePort services |

---

## 🔄 Shutdown & Startup Procedure

### Graceful Shutdown (production-safe)
```bash
# 1. Drain workers first (on master)
kubectl drain dellb-worker-01 --ignore-daemonsets --delete-emptydir-data
kubectl drain dellb-worker-02 --ignore-daemonsets --delete-emptydir-data
kubectl drain dellb-worker-03 --ignore-daemonsets --delete-emptydir-data

# 2. Power off workers
# 3. Power off master last
```

### Startup Order
```
1. dellb-master-01  ← always first, wait ~60 seconds
2. workers          ← any order, they reconnect automatically
```

### After Restart — Uncordon Workers
If workers were drained before shutdown they will come back as `SchedulingDisabled`:
```bash
kubectl uncordon dellb-worker-01
kubectl uncordon dellb-worker-02
kubectl uncordon dellb-worker-03
```

### What Auto-Starts on Boot
- `containerd` — enabled as systemd service ✅
- `kubelet` — enabled as systemd service ✅
- Control plane components (apiserver, scheduler, etcd) — run as static pods, started by kubelet automatically ✅

---

## 💾 VM Backup

### Disk Image Locations
```
/mnt/VM/virtVM/
├── k8-master01-rocky9-7.qcow2        ~9.9G (real) / 37G (provisioned)
├── k8-ubuntu2404-client011.qcow2     ~20G  (real) / 71G (provisioned)
├── k8-worker011-rocky9-7.qcow2       ~9.5G (real) / 72G (provisioned)
├── k8-worker022-rocky9-7.qcow2       ~9.2G (real) / 72G (provisioned)
└── k8-worker033-rocky9-7.qcow2       ~9.4G (real) / 72G (provisioned)
```

> **Note:** `ls -lha` shows provisioned size. `du -sh` shows real data size (~58G total).
> Always use `du -sh` to check actual backup space needed.

### Backup Method — `qemu-img convert` (recommended)

VMs must be **powered off** before backup. `qemu-img convert` understands the qcow2
format internally and only copies real data, unlike `rsync` which copies the full
provisioned size.

**With compression** (slowest, smallest output — ~50% size reduction):
```bash
sudo qemu-img convert -c -O qcow2 \
  /mnt/VM/virtVM/k8-master01-rocky9-7.qcow2 \
  /media/dextr/BackupVM/dellb/k8lab/k8-master01-rocky9-7.qcow2
```

**Without compression** (faster, output close to real `du` size — recommended):
```bash
sudo qemu-img convert -O qcow2 \
  /mnt/VM/virtVM/k8-master01-rocky9-7.qcow2 \
  /media/dextr/BackupVM/dellb/k8lab/k8-master01-rocky9-7.qcow2
```

**Backup all VMs in a loop (no compression):**
```bash
for vm in k8-master01-rocky9-7 \
          k8-ubuntu2404-client011 \
          k8-worker011-rocky9-7 \
          k8-worker022-rocky9-7 \
          k8-worker033-rocky9-7; do
  echo ">>> Backing up $vm ..."
  sudo qemu-img convert -O qcow2 \
    /mnt/VM/virtVM/${vm}.qcow2 \
    /media/dextr/BackupVM/dellb/k8lab/${vm}.qcow2
  echo ">>> Done: $(du -sh /media/dextr/BackupVM/dellb/k8lab/${vm}.qcow2)"
done
```

### Why Not rsync?
`rsync --sparse` does not work reliably with qcow2 files — the empty blocks are not
exposed as filesystem-level sparse holes, so rsync copies the full provisioned size
and will fill up the destination drive.

### Future Backup Solution
**Proxmox Backup Server (PBS)** — free, open source, built for KVM/QEMU:
- Incremental backups (only changed blocks after first run)
- Deduplication + compression
- Web UI, scheduling, retention policies
- Recommended: install as a VM on **dellc** to back up dellb over the network

---

## 🐛 Troubleshooting Encountered

### Worker-03: Missing sudo password
**Symptom:** `fatal: [dellb-worker-03]: FAILED! => Missing sudo password`

**Cause:** The `kuber` user on worker-03 did not have passwordless sudo configured.

**Fix** (SSH into worker-03 manually):
```bash
echo "kuber ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/kuber
sudo chmod 440 /etc/sudoers.d/kuber
```

**Verify:**
```bash
# On k8client
ansible dellb-worker-03 -m command -a "whoami" --become
# Expected: root
```

---

## 📝 Notes

- All playbooks are **idempotent** — safe to re-run at any time
- `workers.yml` checks for `/etc/kubernetes/kubelet.conf` before joining — already-joined nodes are skipped
- The Python interpreter warning (`/usr/bin/python3.9`) is cosmetic and does not affect execution
- Kubernetes version across all nodes: **v1.31.14**
- CNI: **Flannel** with pod CIDR `10.244.0.0/16`
- Container runtime: **containerd** with `SystemdCgroup = true`
