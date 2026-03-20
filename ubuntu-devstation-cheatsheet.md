# k8lab — Ubuntu Dev Station Setup Cheat Sheet

**Host:** dellc / dellb · Ubuntu 24.04.4 LTS  
**Role:** Ansible control node + developer workstation  
**Session:** 02 · 2026-03-19

---

## Phase 1 — System Base

### Step 01 — Full System Update

```bash
sudo apt update && sudo apt upgrade -y
sudo apt autoremove -y && sudo apt autoclean
sudo reboot
```

> ⚠ Always reboot after a kernel update before installing anything else.

---

### Step 02 — Essential Base Tools

```bash
sudo apt install -y \
  curl wget git vim htop tree \
  net-tools iputils-ping \
  software-properties-common \
  apt-transport-https ca-certificates \
  gnupg lsb-release unzip
```

> Covers networking, debugging, repo management and package signing.

---

## Phase 2 — SSH Key (Ansible Auth)

### Step 03 — Generate SSH Keypair

```bash
ssh-keygen -t ed25519 -C "k8lab" -f ~/.ssh/lab_key
```

Verify the key was created:

```bash
ls -la ~/.ssh/lab_key*
```

> Use a passphrase. `ed25519` is faster and safer than RSA.

---

### Step 04 — Distribute Key to VMs

Copy to each VM (repeat for all):

```bash
ssh-copy-id -i ~/.ssh/lab_key.pub adminuser@192.168.10.XX
```

Test the connection:

```bash
ssh -i ~/.ssh/lab_key adminuser@192.168.10.XX
```

> ⚠ Replace `adminuser` and IPs once VMs have static IPs assigned.

---

## Phase 3 — Ansible

### Step 05 — Install Ansible

```bash
# Add official Ansible PPA (latest stable)
sudo add-apt-repository --yes ppa:ansible/ansible
sudo apt update
sudo apt install -y ansible

# Verify
ansible --version
```

> The PPA gives a much newer version than the default Ubuntu repo.

---

### Step 06 — Ansible Inventory (Skeleton)

Create the project directory:

```bash
mkdir -p ~/k8lab/ansible
cd ~/k8lab/ansible
```

Create `~/k8lab/ansible/inventory.ini`:

```ini
# k8lab inventory

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

> Update IPs to match the `.60` block if working from dellc's k8client.

---

### Step 07 — Test Ansible Connectivity

Ping all nodes:

```bash
ansible all -i inventory.ini -m ping
```

Ad-hoc: check hostname on all nodes:

```bash
ansible all -i inventory.ini -m command -a "hostname -f"
```

Ad-hoc: check uptime:

```bash
ansible all -i inventory.ini -m command -a "uptime"
```

> ✓ All nodes should return `pong`. If not — check IPs, SSH key, and that the admin user exists on the VM.

---

## Phase 4 — VSCodium (Editor)

### Step 08 — Install VSCodium

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

> VSCodium = VS Code without Microsoft telemetry. Fully open source.

---

### Step 09 — Recommended Extensions

```bash
codium --install-extension redhat.ansible
codium --install-extension ms-kubernetes-tools.vscode-kubernetes-tools
codium --install-extension ms-azuretools.vscode-docker
codium --install-extension hashicorp.terraform
codium --install-extension eamodio.gitlens
```

> `redhat.ansible` gives YAML linting, task autocomplete and playbook run support.

---

## Phase 5 — Git Config

### Step 10 — Git Identity & Defaults

```bash
git config --global user.name  "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
git config --global core.editor vim
```

---

### Step 11 — Init k8lab Git Repo

```bash
cd ~/k8lab
git init
echo ".retry"   >> .gitignore
echo "*.log"    >> .gitignore
echo "secrets/" >> .gitignore
git add .
git commit -m "init: k8lab ansible project"
```

> Commit after every working playbook. You'll thank yourself when something breaks.

---

## Phase 6 — VM Snapshots (virsh)

### Step 12 — Snapshot All VMs Before Any Changes

Create snapshot (repeat for each VM):

```bash
virsh snapshot-create-as \
  --domain master-01 \
  --name "clean-install" \
  --description "Rocky9 minimal, pre-ansible" \
  --atomic
```

VMs to snapshot: `k8client`, `master-01`, `worker-01`, `worker-02`, `spare-01`

List snapshots for a VM:

```bash
virsh snapshot-list master-01
```

Revert to snapshot if needed:

```bash
virsh snapshot-revert master-01 clean-install
```

> ⚠ Revert destroys all changes made after the snapshot was taken.

---

## Quick Sanity Check — All Steps Done

```bash
ansible --version          # Ansible installed
python3 --version          # Python available
ls ~/.ssh/lab_key*         # SSH key exists
git config --list          # Git identity set
codium --version           # VSCodium installed
ansible all -i inventory.ini -m ping   # All VMs reachable
```

---

**Next:** `base.yml` → `k8s-prereq.yml` → `master.yml` → `worker.yml`
