# k8lab — KVM Bridge Networking Setup (NetworkManager)

**dellb:** `eno1` → `br0` · 192.168.10.3  
**dellc:** `enp0s31f6` → `br0` · 192.168.10.4  
**Gateway:** 192.168.10.1 (Mikrotik)  
**Renderer:** NetworkManager (nmcli — no Netplan changes needed)

---

## Full IP Plan — 192.168.10.0/24

| Hostname | IP Address | Role | Host |
|---|---|---|---|
| — | 192.168.10.1 | Mikrotik router / gateway | — |
| dellb | 192.168.10.3 | KVM bare-metal host | physical |
| dellc | 192.168.10.4 | KVM bare-metal host (primary) | physical |
| **dellb cluster** | | | |
| k8client | 192.168.10.50 | Ubuntu dev VM / Ansible control node | dellb |
| master-01 | 192.168.10.51 | k8s control plane | dellb |
| worker-01 | 192.168.10.52 | k8s worker | dellb |
| worker-02 | 192.168.10.53 | k8s worker | dellb |
| spare-01 | 192.168.10.54 | monitoring (Prometheus/Grafana) | dellb |
| **dellc cluster** | | | |
| k8client | 192.168.10.60 | Ubuntu dev VM / Ansible control node | dellc |
| master-01 | 192.168.10.61 | k8s control plane | dellc |
| worker-01 | 192.168.10.62 | k8s worker | dellc |
| worker-02 | 192.168.10.63 | k8s worker | dellc |
| spare-01 | 192.168.10.64 | monitoring (Prometheus/Grafana) | dellc |
| — | 192.168.10.5 – .49 | free — other devices, future hosts | — |
| — | 192.168.10.150 – .253 | Mikrotik DHCP pool (untouched) | — |

---

## Phase 1 — Create br0 Bridge via nmcli

> ⚠ Your host IP moves from the physical NIC to `br0` when this is applied. WiFi, VPN and all other NM connections stay untouched.

### dellb — `eno1` → `br0`

```bash
# 1. Create the bridge interface
sudo nmcli con add type bridge \
  con-name br0 ifname br0

# 2. Set static IP on the bridge
sudo nmcli con mod br0 \
  ipv4.addresses 192.168.10.3/24 \
  ipv4.gateway   192.168.10.1 \
  ipv4.dns       "192.168.10.1 8.8.8.8" \
  ipv4.method    manual \
  bridge.stp     no

# 3. Attach physical NIC as bridge slave
sudo nmcli con add type bridge-slave \
  con-name br0-slave ifname eno1 master br0

# 4. Bring the bridge up
sudo nmcli con up br0

# 5. Verify
ip -br addr show br0
# Expected: br0 UP 192.168.10.3/24
```

---

### dellc — `enp0s31f6` → `br0`

```bash
# 1. Create the bridge interface
sudo nmcli con add type bridge \
  con-name br0 ifname br0

# 2. Set static IP on the bridge
sudo nmcli con mod br0 \
  ipv4.addresses 192.168.10.4/24 \
  ipv4.gateway   192.168.10.1 \
  ipv4.dns       "192.168.10.1 8.8.8.8" \
  ipv4.method    manual \
  bridge.stp     no

# 3. Attach physical NIC as bridge slave
sudo nmcli con add type bridge-slave \
  con-name br0-slave ifname enp0s31f6 master br0

# 4. Bring the bridge up
sudo nmcli con up br0

# 5. Verify
ip -br addr show br0
# Expected: br0 UP 192.168.10.4/24
```

---

### Verify bridge is active on both hosts

```bash
# Show all connections and their state
nmcli con show

# Show bridge details
nmcli dev show br0

# Full interface overview
ip -br addr show
```

---

### Rollback — if something goes wrong

```bash
sudo nmcli con delete br0
sudo nmcli con delete br0-slave
sudo nmcli con up "Wired connection 1"
```

---

## Phase 2 — Switch VMs from NAT to br0

> ⚠ Shut off the VM before changing its NIC config.

### Option A — virt-manager (GUI)

1. Open **virt-manager**
2. Right-click VM → **Open**
3. Click the **ⓘ (Show hardware details)** icon
4. Select **NIC** in the left panel
5. Change Network source:
   - from: `Virtual network 'default' (NAT)`
   - to: `Bridge device: br0`
6. Click **Apply**
7. Repeat for all 5 VMs on each host

---

### Option B — virsh (CLI, scriptable)

```bash
virsh edit master-01
```

Find this block in the XML:

```xml
<!-- BEFORE (NAT) -->
<interface type='network'>
  <source network='default'/>
</interface>
```

Replace with:

```xml
<!-- AFTER (bridge) -->
<interface type='bridge'>
  <source bridge='br0'/>
  <model type='virtio'/>
</interface>
```

Repeat for all VMs. Check current NIC config on all at once:

```bash
for vm in k8client master-01 worker-01 worker-02 spare-01; do
  echo "=== $vm ==="
  virsh dumpxml $vm | grep -A3 'interface'
done
```

---

## Phase 3 — Assign Static IPs Inside Each VM

### Rocky Linux 9 VMs — nmcli

Run inside each VM via console or SSH:

```bash
# 1. Find the connection name
nmcli con show

# 2. Set static IP — adjust address per VM (see table below)
nmcli con mod "Wired connection 1" \
  ipv4.addresses 192.168.10.51/24 \
  ipv4.gateway   192.168.10.1 \
  ipv4.dns       "192.168.10.1 8.8.8.8" \
  ipv4.method    manual

# 3. Apply
nmcli con up "Wired connection 1"

# 4. Verify
ip a show eth0
```

### k8client VM — Ubuntu — nmcli

k8client is Ubuntu, but since it's a plain VM (not a daily desktop), nmcli works fine here too:

```bash
# Find connection name
nmcli con show

# Set static IP
sudo nmcli con mod "Wired connection 1" \
  ipv4.addresses 192.168.10.50/24 \
  ipv4.gateway   192.168.10.1 \
  ipv4.dns       "192.168.10.1 8.8.8.8" \
  ipv4.method    manual

# Apply
sudo nmcli con up "Wired connection 1"

# Verify
ip a
```

> Use `.60` instead of `.50` for the k8client on dellc.

---

### IP per VM — quick reference

| VM | dellb | dellc |
|---|---|---|
| k8client | 192.168.10.50 | 192.168.10.60 |
| master-01 | 192.168.10.51 | 192.168.10.61 |
| worker-01 | 192.168.10.52 | 192.168.10.62 |
| worker-02 | 192.168.10.53 | 192.168.10.63 |
| spare-01 | 192.168.10.54 | 192.168.10.64 |

---

## Phase 4 — Verify Cross-Host Connectivity

```bash
# From dellb host → dellc VMs
ping -c3 192.168.10.61
ping -c3 192.168.10.62

# From a VM on dellb → a VM on dellc
# (SSH into master-01 on dellb first)
ssh 192.168.10.51
ping -c3 192.168.10.62

# From k8client — Ansible ping all nodes
ansible all -i inventory.ini -m ping
```

All should succeed. If not:

```bash
# Check bridge is up on the host
ip -br addr show br0

# Check VM NIC is attached to br0
virsh dumpxml master-01 | grep -A3 'interface'

# Check VM has correct static IP
ssh 192.168.10.51 ip a
```

---

**Next:** snapshots → `base.yml` → `k8s-prereq.yml` → `master.yml` → `worker.yml`
