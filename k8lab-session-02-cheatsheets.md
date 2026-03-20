# k8lab — Session 02 Combined Cheat Sheet

**Date:** 2026-03-20  
**Hosts:** dellb (`192.168.10.3`) · dellc (`192.168.10.4`)  
**Gateway:** `192.168.10.1` (Mikrotik)

---

## IP Plan — Quick Reference

| VM | dellb | dellc |
|---|---|---|
| k8client | 192.168.10.50 | 192.168.10.60 |
| master-01 | 192.168.10.51 | 192.168.10.61 |
| worker-01 | 192.168.10.52 | 192.168.10.62 |
| worker-02 | 192.168.10.53 | 192.168.10.63 |
| spare-01 | 192.168.10.54 | 192.168.10.64 |

---

# Part 1 — VM Base Configuration (Stage 0 Bootstrap)

> Run these commands **inside each VM console** before Ansible exists.
> Order matters — follow top to bottom.

---

## 1. Add User to Sudo

```bash
su -
visudo
# Add this line:
username ALL=(ALL) NOPASSWD:ALL
```

---

## 2. Update System

```bash
# Rocky Linux 9 / RHEL
sudo dnf update -y

# Ubuntu (k8client)
sudo apt update && sudo apt upgrade -y
sudo apt autoremove -y && sudo apt autoclean
sudo reboot
```

---

## 3. Install Base Packages

```bash
# Rocky Linux 9 / RHEL
sudo dnf install -y epel-release
sudo dnf install -y curl wget git vim htop tree net-tools

# Ubuntu (k8client)
sudo apt install -y \
  curl wget git vim htop tree \
  net-tools iputils-ping \
  software-properties-common \
  apt-transport-https ca-certificates \
  gnupg lsb-release unzip
```

---

## 4. Set Hungarian Keyboard Layout

```bash
# List available keymaps
localectl list-keymaps

# Set Hungarian layout
sudo loadkeys hu

# If the above does not persist — make it permanent
sudo nano /etc/vconsole.conf
# Add line:
KEYMAP=hu
```

---

## 5. Rename User if Needed

```bash
# Check existing users
cat /etc/passwd

# Rename user
sudo usermod -l newname oldname
```

---

## 6. Set Hostname

```bash
# Set per VM — adjust name accordingly
hostnamectl set-hostname master-01

# Verify
hostname
```

Hostname per VM:

| VM | Hostname |
|---|---|
| k8client | k8client |
| Control plane | master-01 |
| Worker 1 | worker-01 |
| Worker 2 | worker-02 |
| Monitoring | spare-01 |

---

## 7. Disable Swap (Kubernetes Requirement)

```bash
# Disable immediately
sudo swapoff -a

# Make permanent — comment out swap line in fstab
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Verify — Swap line should show 0
free -m
```

---

## 8. Load Required Kernel Modules for Kubernetes

```bash
# Load now
sudo modprobe overlay
sudo modprobe br_netfilter

# Rocky only — extra modules package
sudo dnf install -y kernel-modules-extra

# Make permanent
sudo tee /etc/modules-load.d/kubernetes.conf > /dev/null <<EOF
overlay
br_netfilter
EOF
```

---

## 9. Disable SELinux (Rocky Linux 9 only)

```bash
# Check current status
sudo getenforce

# Set permissive for current session
sudo setenforce 0

# Make permanent (requires reboot to fully apply)
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# Verify
sudo getenforce
```

---

## 10. Set Static IP (inside each VM)

```bash
# Find the connection name
nmcli device status

# Set static IP — adjust address per VM
sudo nmcli con modify "Wired connection 1" \
  ipv4.method    manual \
  ipv4.addresses "192.168.10.51/24" \
  ipv4.gateway   "192.168.10.1" \
  ipv4.dns       "192.168.10.1,8.8.8.8"

# Apply
sudo nmcli con down "Wired connection 1"
sudo nmcli con up   "Wired connection 1"

# Verify
nmcli device show
ip addr show

# If duplicate IP appears — remove old one
sudo ip addr del <IP/PREFIX> dev <INTERFACE>
```

---

# Part 2 — SSH Setup

---

## 11. Install SSH Packages

```bash
# Ubuntu / Debian
sudo apt install -y openssh-client openssh-server

# Rocky Linux 9 / RHEL
sudo dnf install -y openssh-clients openssh-server
```

---

## 12. Enable & Start SSH Service

```bash
# Ubuntu
sudo systemctl enable --now ssh
sudo systemctl status ssh

# Rocky Linux 9 / RHEL
sudo systemctl enable --now sshd
sudo systemctl status sshd
```

---

## 13. Generate Named ed25519 Keypair

Run on **k8client** (the Ansible control node):

```bash
ssh-keygen -t ed25519 -C "k8lab" -f ~/.ssh/lab_key
```

```
~/.ssh/lab_key      ← private key (never share this)
~/.ssh/lab_key.pub  ← public key (copy to every VM)
```

Verify:

```bash
ls -la ~/.ssh/lab_key*
```

---

## 14. Distribute Public Key to All VMs

```bash
# dellb cluster
ssh-copy-id -i ~/.ssh/lab_key.pub adminuser@192.168.10.51
ssh-copy-id -i ~/.ssh/lab_key.pub adminuser@192.168.10.52
ssh-copy-id -i ~/.ssh/lab_key.pub adminuser@192.168.10.53
ssh-copy-id -i ~/.ssh/lab_key.pub adminuser@192.168.10.54

# dellc cluster
ssh-copy-id -i ~/.ssh/lab_key.pub adminuser@192.168.10.61
ssh-copy-id -i ~/.ssh/lab_key.pub adminuser@192.168.10.62
ssh-copy-id -i ~/.ssh/lab_key.pub adminuser@192.168.10.63
ssh-copy-id -i ~/.ssh/lab_key.pub adminuser@192.168.10.64
```

---

## 15. Optional — SSH Client Config

Edit `~/.ssh/config` on k8client so you can type `ssh master-01` instead of the full command:

```
# dellb cluster
Host k8client-b
    HostName 192.168.10.50
    User adminuser
    IdentityFile ~/.ssh/lab_key

Host master-01
    HostName 192.168.10.51
    User adminuser
    IdentityFile ~/.ssh/lab_key

Host worker-01
    HostName 192.168.10.52
    User adminuser
    IdentityFile ~/.ssh/lab_key

Host worker-02
    HostName 192.168.10.53
    User adminuser
    IdentityFile ~/.ssh/lab_key

Host spare-01
    HostName 192.168.10.54
    User adminuser
    IdentityFile ~/.ssh/lab_key

# dellc cluster — same pattern at .6x
```

Test:

```bash
ssh master-01
```

---

## 16. Harden SSH on VMs (after key auth confirmed working)

```bash
sudo vim /etc/ssh/sshd_config
```

Set these values:

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
```

Restart:

```bash
# Ubuntu
sudo systemctl restart ssh

# Rocky Linux 9 / RHEL
sudo systemctl restart sshd
```

> ⚠ Only do this after confirming key-based login works. You will lock yourself out otherwise.

---

# Part 3 — KVM Bridge Networking (on the bare-metal hosts)

> Run on **dellb** and **dellc** directly — not inside a VM.

---

## 17. Create br0 on dellb (`eno1` → `br0`)

```bash
# Create bridge
sudo nmcli con add type bridge \
  con-name br0 ifname br0

# Set static IP on bridge
sudo nmcli con mod br0 \
  ipv4.addresses 192.168.10.3/24 \
  ipv4.gateway   192.168.10.1 \
  ipv4.dns       "192.168.10.1 8.8.8.8" \
  ipv4.method    manual \
  bridge.stp     no

# Attach physical NIC as slave
sudo nmcli con add type bridge-slave \
  con-name br0-slave ifname eno1 master br0

# Bring up
sudo nmcli con up br0

# Verify
ip -br addr show br0
```

---

## 18. Create br0 on dellc (`enp0s31f6` → `br0`)

```bash
# Create bridge
sudo nmcli con add type bridge \
  con-name br0 ifname br0

# Set static IP on bridge
sudo nmcli con mod br0 \
  ipv4.addresses 192.168.10.4/24 \
  ipv4.gateway   192.168.10.1 \
  ipv4.dns       "192.168.10.1 8.8.8.8" \
  ipv4.method    manual \
  bridge.stp     no

# Attach physical NIC as slave
sudo nmcli con add type bridge-slave \
  con-name br0-slave ifname enp0s31f6 master br0

# Bring up
sudo nmcli con up br0

# Verify
ip -br addr show br0
```

---

## 19. Bridge Rollback (if something goes wrong)

```bash
sudo nmcli con delete br0
sudo nmcli con delete br0-slave
sudo nmcli con up "Wired connection 1"
```

---

## 20. Switch VMs from NAT to br0

### Option A — virt-manager (GUI)

1. Open virt-manager
2. Right-click VM → Open
3. Click ⓘ (Show hardware details)
4. Select **NIC** in left panel
5. Change Network source: `Virtual network 'default' (NAT)` → `Bridge device: br0`
6. Click **Apply**
7. Repeat for all 5 VMs on each host

> ⚠ VM must be shut off before changing NIC config.

### Option B — virsh (CLI)

```bash
virsh edit master-01
```

Find and replace this XML block:

```xml
<!-- BEFORE -->
<interface type='network'>
  <source network='default'/>
</interface>

<!-- AFTER -->
<interface type='bridge'>
  <source bridge='br0'/>
  <model type='virtio'/>
</interface>
```

Check all VMs at once:

```bash
for vm in k8client master-01 worker-01 worker-02 spare-01; do
  echo "=== $vm ==="
  virsh dumpxml $vm | grep -A3 'interface'
done
```

---

# Part 4 — Ubuntu Dev Station Setup (k8client)

---

## 21. Install Ansible

```bash
# Add official PPA (latest stable)
sudo add-apt-repository --yes ppa:ansible/ansible
sudo apt update
sudo apt install -y ansible

# Verify
ansible --version
```

---

## 22. Create Ansible Project Structure

```bash
mkdir -p ~/k8lab/ansible
cd ~/k8lab/ansible
```

Create `~/k8lab/ansible/inventory.ini`:

```ini
[masters]
master-01 ansible_host=192.168.10.51

[workers]
worker-01 ansible_host=192.168.10.52
worker-02 ansible_host=192.168.10.53

[spare]
spare-01  ansible_host=192.168.10.54

[all:vars]
ansible_user=adminuser
ansible_ssh_private_key_file=~/.ssh/lab_key
```

> Use `.6x` IPs for the dellc k8client inventory.

---

## 23. Test Ansible Connectivity

```bash
# Ping all nodes
ansible all -i inventory.ini -m ping

# Check hostnames
ansible all -i inventory.ini -m command -a "hostname -f"

# Check uptime
ansible all -i inventory.ini -m command -a "uptime"
```

> ✓ All nodes should return `pong`. If not — check static IPs, SSH key distribution, and sudo config on the VM.

---

## 24. Install VSCodium

```bash
# Add GPG key
wget -qO - \
  https://gitlab.com/paulcarroty/vscodium-deb-rpm-repo/raw/master/pub.gpg \
  | gpg --dearmor \
  | sudo dd of=/usr/share/keyrings/vscodium-archive-keyring.gpg

# Add repo
echo 'deb [ signed-by=/usr/share/keyrings/vscodium-archive-keyring.gpg ] https://download.vscodium.com/debs vscodium main' \
  | sudo tee /etc/apt/sources.list.d/vscodium.list

# Install
sudo apt update && sudo apt install -y codium
```

---

## 25. Install VSCodium Extensions

```bash
codium --install-extension redhat.ansible
codium --install-extension ms-kubernetes-tools.vscode-kubernetes-tools
codium --install-extension ms-azuretools.vscode-docker
codium --install-extension hashicorp.terraform
codium --install-extension eamodio.gitlens
```

---

## 26. Configure Git

```bash
# Check version
git --version

# Set identity
git config --global user.name  "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
git config --global core.editor vim
```

---

## 27. Init k8lab Git Repo

```bash
cd ~/k8lab
git init
echo ".retry"   >> .gitignore
echo "*.log"    >> .gitignore
echo "secrets/" >> .gitignore
git add .
git commit -m "init: k8lab ansible project"
```

> Commit after every working playbook. You will thank yourself when something breaks.

---

# Part 5 — VM Snapshots

---

## 28. Take Clean Snapshots (before any changes)

```bash
# Repeat for each VM: k8client, master-01, worker-01, worker-02, spare-01
virsh snapshot-create-as \
  --domain master-01 \
  --name "clean-install" \
  --description "Rocky9 minimal, pre-ansible" \
  --atomic
```

```bash
# List snapshots
virsh snapshot-list master-01

# Revert if needed
virsh snapshot-revert master-01 clean-install
```

> ⚠ Revert destroys all changes made after the snapshot was taken.

---

# Part 6 — Python (Ansible Requirement)

---

## 29. Python 3 Check & Install

```bash
# Ubuntu (k8client) — install if missing
sudo apt install -y python3 python3-pip

# Rocky Linux 9 — already installed, just verify
python3 --version

# Both — verify
python3 --version
which python3
```

> Rocky Linux 9 Minimal ships with Python 3.9 — no action needed on managed nodes.

---

# Final Sanity Check

```bash
# On k8client — verify everything is in place
ansible --version        # Ansible installed
python3 --version        # Python available
ls ~/.ssh/lab_key*       # SSH keypair exists
git config --list        # Git identity configured
codium --version         # VSCodium installed

# Ping all VMs
ansible all -i ~/k8lab/ansible/inventory.ini -m ping
```

All nodes return `pong` → infrastructure is ready → begin writing playbooks.

---

## Quick Reference — Package & Service Names

| | Ubuntu | Rocky Linux 9 / RHEL |
|---|---|---|
| Update command | `apt update && apt upgrade` | `dnf update` |
| SSH client | `openssh-client` | `openssh-clients` |
| SSH server | `openssh-server` | `openssh-server` |
| SSH service name | `ssh` | `sshd` |
| Python | `python3 python3-pip` | pre-installed |
| Extra repos | `add-apt-repository ppa:...` | `dnf install epel-release` |

---

**Next:** `base.yml` → `k8s-prereq.yml` → `master.yml` → `worker.yml`
