# K8Lab – Part 3: Kubernetes Fundamentals
### Session Summary | DELLB Cluster | v1.31.14

---

## 📋 Infrastructure Recap

| Role | Hostname | IP | OS |
|---|---|---|---|
| KVM Host | DELLB | 192.168.10.3 | Ubuntu 24.04 LTS |
| Ansible Control Node | dellb-k8client | 192.168.10.58 | Ubuntu 24.04 LTS |
| Control Plane | dellb-master-01 | 192.168.10.59 | Rocky Linux 9 |
| Worker 01 | dellb-worker-01 | 192.168.10.51 | Rocky Linux 9 |
| Worker 02 | dellb-worker-02 | 192.168.10.52 | Rocky Linux 9 |
| Worker 03 | dellb-worker-03 | 192.168.10.53 | Rocky Linux 9 |

**CNI:** Flannel — pod CIDR `10.244.0.0/16`  
**Container runtime:** containerd with `SystemdCgroup = true`  
**Kubernetes version:** v1.31.14 across all nodes

---

## 🧠 Theory

### Helm and Helm Charts

**Ansible** is a general-purpose automation tool. It deploys and configures infrastructure —
VMs, OS settings, installing packages, bootstrapping Kubernetes itself.

**Helm** is a package manager specifically for Kubernetes — analogous to `apt` or `dnf`,
but instead of installing software on an OS, it installs applications *into* a running cluster.

A **Helm chart** is the package — a bundle of pre-written Kubernetes YAML templates
(Deployments, Services, ConfigMaps, etc.) for a specific application. Instead of writing
all that YAML by hand, someone already did it, and you install it with one command.

```
Ansible  = "set up the machine so Kubernetes can run"
Helm     = "deploy applications inside Kubernetes"
```

They complement each other — Ansible built the cluster, Helm puts things on top of it.

---

### Kubernetes Architecture

The fundamental split in Kubernetes:

- **Control Plane** = the brain — makes decisions, stores state, watches everything
- **Data Plane (Workers)** = the muscle — actually runs workloads

#### Control Plane Components

| Component | Role |
|---|---|
| **API Server** | Single entry point for everything. Every `kubectl` command and every internal component communicates exclusively through the API server. Port `6443`. If it dies, the cluster is unmanageable even if workloads keep running. |
| **etcd** | Key-value database — stores the entire cluster state. Every object (nodes, pods, configs, secrets) lives here. Kubernetes itself is stateless — everything reads from and writes to etcd. If etcd is gone, cluster state is gone. **Back this up in production.** |
| **Scheduler** | Watches for newly created pods with no node assigned and decides which worker to place them on, based on available resources, taints, tolerations, and affinity rules. If no node fits, pods sit in `Pending` state indefinitely. |
| **Controller Manager** | A collection of control loops. The ReplicaSet controller ensures the correct number of pod replicas exist. The Node controller monitors node health. The Deployment controller manages rolling updates. Constantly compares *desired state* vs *actual state* and acts to close the gap. |
| **Cloud Controller Manager** | Only relevant on cloud providers (AWS, GCP, Azure). Handles cloud-specific resources like load balancers. **Irrelevant on bare metal — ignore it.** |

#### Worker Node Components

| Component | Role |
|---|---|
| **kubelet** | Agent running on every worker. Receives pod specs from the API server and tells the container runtime to start/stop containers. Reports node and pod status back. Enabled as a systemd service. |
| **Container Runtime** | Does the actual work of pulling images and running containers. In this lab: **containerd**. kubelet talks to it via CRI (Container Runtime Interface). Chain: API Server → kubelet → containerd → running container. |
| **kube-proxy** | Runs on every worker. Maintains iptables rules so traffic destined for a Kubernetes Service reaches the correct pod on any node. It does not actually "listen" on ports — it creates NAT rules. This is why `ss -tlnp` shows nothing for NodePort ports. |

#### The Core Philosophy — Desired State

You never say "start this container now." You declare "I want 3 replicas of this always
running" and Kubernetes continuously enforces that. Kill a pod manually — it comes back.
This is why Kubernetes self-heals.

#### Key Relationships to Internalize

1. **Everything goes through the API server.** No component talks directly to another.
2. **Kubernetes is eventually consistent.** Apply something → scheduler assigns it →
   kubelet pulls image → container starts. You will see `ContainerCreating` states.
3. **The master never runs your workloads** (by default). A taint on the control plane
   node prevents scheduling there. Workers are for workloads.
4. **Pods are ephemeral.** They get killed, rescheduled, get new IPs. Never rely on
   a pod's IP directly — always use a **Service** in front of it.

---

### Kubernetes Object Types (used in this session)

| Object | Purpose |
|---|---|
| **Deployment** | Declares desired state for a set of pods — image, replicas, update strategy. Manages ReplicaSets underneath. |
| **ReplicaSet** | Ensures a specified number of pod replicas are running. Created and managed by Deployments. |
| **Pod** | The smallest deployable unit. One or more containers sharing network and storage. Ephemeral — gets a new IP every time it restarts. |
| **Service** | Stable network endpoint in front of pods. Abstracts pod IPs — you talk to the Service, it routes to healthy pods. |
| **NodePort** | Service type that exposes a port on every worker node (30000–32767 range). The bare-metal equivalent of a cloud LoadBalancer. |

---

### Namespaces

Everything in Kubernetes lives in a namespace. Default is `default`. System components
live in `kube-system`. Flannel lives in `kube-flannel`.

If you can't find something, you're probably in the wrong namespace.
`kubectl get pods -A` shows everything across all namespaces.

---

### Firewalld vs kube-proxy on Workers — Important Lesson

kube-proxy manages iptables rules directly on worker nodes to handle pod routing and
NodePort traffic. Firewalld is an additional firewall layer that sits on top and **blocks
forwarded traffic between interfaces** (flannel.1 ↔ cni0) even when ports are explicitly
opened via `firewall-cmd`.

**Result:** pods on different nodes cannot communicate — manifests as "No route to host"
even when routing tables and Flannel are correctly configured.

**Solution:** disable firewalld entirely on worker nodes. kube-proxy is the firewall on
workers. The master node keeps firewalld because it is the single external entry point
and worth protecting.

**Flannel VXLAN port:** `8472/udp` — required for cross-node pod networking. Must be open
on the master's firewall. This port was missing from the original Ansible playbooks and
caused the NodePort connectivity issue in this session.

---

## ⌨️ Commands Reference

### Setup — kubectl Tab Completion and Alias

Run once on any machine where you use `kubectl`:

```bash
# Install bash-completion (Ubuntu/Debian)
sudo apt install bash-completion

# Enable kubectl tab completion permanently
echo 'source <(kubectl completion bash)' >> ~/.bashrc

# Add k alias (everyone uses this in practice)
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc

# Apply immediately
source ~/.bashrc
```

After this, Tab completes commands, resource types, and actual resource names from
the live cluster. `k get pods [TAB]` shows real pod names.

---

### Step 1 — Explore the Cluster

Run every morning before touching anything. Gives a full health overview.

```bash
kubectl get nodes                          # Are all nodes Ready?
kubectl get pods -n kube-system            # Are all system pods running?
kubectl get pods -n kube-flannel           # Is Flannel healthy? (note: own namespace)
kubectl describe node dellb-master-01      # Detailed: resources, conditions, events
kubectl cluster-info                       # API server + CoreDNS endpoints
```

**Expected healthy output:**
```
NAME              STATUS   ROLES           AGE     VERSION
dellb-master-01   Ready    control-plane   7h9m    v1.31.14
dellb-worker-01   Ready    <none>          6h20m   v1.31.14
dellb-worker-02   Ready    <none>          6h20m   v1.31.14
dellb-worker-03   Ready    <none>          6h20m   v1.31.14
```

---

### Step 2 — First Deployment

Creates a Deployment (which creates a ReplicaSet, which creates Pods).

```bash
kubectl create deployment my-nginx --image=nginx --replicas=2
kubectl get deployments                    # Shows READY count (e.g. 2/2)
kubectl get pods                           # Shows pod status
kubectl get pods -o wide                   # Shows which worker each pod landed on
```

**What to notice:**
- Pods start in `ContainerCreating` while the image is pulled, then move to `Running`
- The scheduler automatically spreads pods across different workers for resilience
- Pod IPs are in the Flannel range (`10.244.x.x`) — internal only

---

### Step 3 — Expose the Deployment (NodePort Service)

Creates a Service that makes the deployment reachable from outside the cluster.

```bash
kubectl expose deployment my-nginx --port=80 --type=NodePort
kubectl get service my-nginx
```

**Example output:**
```
NAME       TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
my-nginx   NodePort   10.98.121.2   <none>        80:32443/TCP   17s
```

- `CLUSTER-IP` — internal virtual IP, reachable only inside the cluster
- `80:32443` — internal port 80 maps to external NodePort 32443
- `EXTERNAL-IP <none>` — expected on bare metal (only fills in on cloud providers)
- The assigned NodePort is reachable on **every worker node**, regardless of which
  node the pod is actually running on — kube-proxy routes it

```bash
# Test from any worker node or browser
curl http://192.168.10.51:32443
curl http://192.168.10.52:32443
curl http://192.168.10.53:32443
```

---

### Step 4 — Scale

Scale up and down with a single command. The Service tracks pod changes automatically.

```bash
# Scale up to 5 replicas
kubectl scale deployment my-nginx --replicas=5
kubectl get pods -o wide --watch           # Watch scheduler spread pods across workers

# Scale back down to 1
kubectl scale deployment my-nginx --replicas=1
kubectl get pods -o wide
```

**What to notice:**
- Scaling up: scheduler distributes new pods across workers automatically
- Scaling down: Kubernetes picks which pods to terminate, keeps the oldest
- The Service on port 32443 remains reachable throughout — zero downtime
- `--watch` streams live updates, Ctrl+Z to stop

---

### Step 5 — Self-Healing

Delete a pod manually and watch the ReplicaSet controller replace it immediately.

```bash
kubectl get pods                                        # Note the pod name
kubectl delete pod my-nginx-8474f7bd66-t85w5           # Kill it
kubectl get pods                                        # New pod already running
```

**What happens:**
1. Pod is deleted
2. ReplicaSet controller notices: "desired=1, actual=0"
3. Creates a new pod immediately
4. New pod gets a different name and different IP
5. Service automatically routes to the new pod — no intervention needed

This is the self-healing guarantee. You cannot permanently delete a pod that is managed
by a Deployment — only the Deployment itself.

---

### Step 6 — Rolling Update

Update the container image version with zero downtime.

```bash
kubectl set image deployment/my-nginx nginx=nginx:1.25
kubectl rollout status deployment/my-nginx             # Watch rollout progress
kubectl get pods -o wide                               # See new ReplicaSet hash in name
```

**What happens:**
- Kubernetes creates a new ReplicaSet for the new version
- Starts the new pod, waits until it is `Running` and healthy
- Only then terminates the old pod
- With multiple replicas, does this one at a time — service never goes down
- Old ReplicaSet is kept at 0 replicas (not deleted) — enables rollback

**Pod name change signals the new version:**
```
Old: my-nginx-8474f7bd66-sfphx   ← ReplicaSet hash 8474f7bd66 (original)
New: my-nginx-dc87bb9fb-p7vc8    ← ReplicaSet hash dc87bb9fb  (nginx:1.25)
```

---

### Step 7 — Rollback

Instantly revert to the previous version. Kubernetes kept the old ReplicaSet for this.

```bash
kubectl rollout undo deployment/my-nginx
kubectl rollout status deployment/my-nginx
kubectl get pods
```

**What happens:**
- Old ReplicaSet scales back up
- New ReplicaSet scales back down to 0
- Completes in seconds
- Pod name hash returns to the original

---

### Step 8 — Clean Up

Delete all resources created during the session.

```bash
kubectl delete deployment my-nginx
kubectl delete service my-nginx

# Verify clean state
kubectl get pods                           # Should be empty (or briefly show Terminating)
kubectl get services                       # Should show only the default 'kubernetes' service
```

**Expected final state:**
```
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   17h
```

---

## 🐛 Troubleshooting Encountered

### NodePort unreachable — root cause: firewalld blocking Flannel VXLAN

**Symptom:** `curl http://192.168.10.51:32443` hangs, no response.

**Diagnosis steps:**
```bash
# On worker — check if iptables rule exists (it does, but kube-proxy uses NAT not listening sockets)
sudo iptables -t nat -L | grep 32443

# On master — check kube-proxy pod logs
kubectl get pods -n kube-system | grep kube-proxy
kubectl logs -n kube-system kube-proxy-47xbk

# Check cross-node pod connectivity
# From worker-02: curl http://10.244.3.2   → works (same node)
# From worker-03: curl http://10.244.3.2   → fails (cross-node)
```

**Finding:** same-node pod access works, cross-node fails → Flannel VXLAN tunnel blocked.

**Root cause:** firewalld on worker nodes blocks forwarded traffic between `flannel.1`
and `cni0` interfaces, even when ports are explicitly opened with `firewall-cmd`.

**Fix — disable firewalld on all workers:**
```bash
for node in dellb-worker-01 dellb-worker-02 dellb-worker-03; do
  ssh $node "sudo systemctl disable --now firewalld"
done
```

**Why:** kube-proxy manages its own iptables rules. Firewalld is redundant and conflicting
on worker nodes. Workers do not need their own firewall — kube-proxy handles routing.

---

### Flannel appeared missing from kube-system

**Symptom:** `kubectl get pods -n kube-system | grep flannel` returns nothing.

**Explanation:** Newer versions of Flannel deploy into their own namespace `kube-flannel`,
not `kube-system`. Always check the correct namespace:

```bash
kubectl get pods -n kube-flannel
```

---

### Common kubectl mistakes to avoid

- Running `kubectl` on a **worker node** — workers have no kubeconfig, so kubectl
  tries `localhost:8080` and fails. Always use the master or k8client.
- Grepping for pods in the wrong namespace — use `-A` flag to search all namespaces.
- Using `ss -tlnp` to check if NodePort is listening — kube-proxy uses iptables NAT,
  not listening sockets. The port won't show up in `ss` output.

---

## 🔧 Ansible Playbook Fixes

Two playbooks were updated based on lessons from this session:

### `base-workers.yml` — changes
- **Removed** all `firewalld` open-port tasks
- **Added** task to stop, disable, and **mask** firewalld permanently
- Masking prevents firewalld from being accidentally started by another service

### `base.yml` (master) — changes
- **Kept** firewalld (master is the single external entry point, worth protecting)
- **Added** `8472/udp` for Flannel VXLAN cross-node pod networking

```yaml
# New task added to base.yml
- name: Open firewalld port 8472 UDP (Flannel VXLAN overlay)
  ansible.posix.firewalld:
    port: 8472/udp
    permanent: true
    state: enabled
    immediate: true
```

```yaml
# New task replacing all firewalld tasks in base-workers.yml
- name: Stop and disable firewalld on workers
  ansible.builtin.systemd:
    name: firewalld
    state: stopped
    enabled: false
    masked: true
```

---

## ✅ Full Lifecycle Completed

| Step | Command | Result |
|---|---|---|
| Explore | `kubectl get nodes/pods` | All 4 nodes Ready, system pods healthy |
| Deploy | `kubectl create deployment` | 2 nginx pods, spread across workers |
| Expose | `kubectl expose --type=NodePort` | Port 32443, reachable on all workers |
| Scale up | `kubectl scale --replicas=5` | 5 pods spread across 3 workers |
| Scale down | `kubectl scale --replicas=1` | 1 pod, service uninterrupted |
| Self-heal | `kubectl delete pod` | Replacement pod up in seconds |
| Update | `kubectl set image` | Rolling update, zero downtime |
| Rollback | `kubectl rollout undo` | Back to previous version instantly |
| Clean up | `kubectl delete deployment/service` | Cluster back to clean state |

---

## 📝 Key Takeaways

1. **kube-proxy uses iptables NAT, not listening sockets** — don't use `ss` to verify NodePort
2. **Flannel lives in `kube-flannel` namespace**, not `kube-system` in newer versions
3. **Disable firewalld on workers** — kube-proxy is the firewall, they conflict
4. **Port 8472/udp** must be open on master's firewall for Flannel VXLAN cross-node traffic
5. **Pods are ephemeral** — always use Services, never rely on pod IPs
6. **Desired state is everything** — you declare what you want, Kubernetes enforces it continuously
7. **Rolling updates keep old ReplicaSets** — rollback is instant and safe
8. **kubectl goes on master or k8client** — never run it on worker nodes

---

## ⏭️ Next Session Topics (candidates)

- **Namespaces** — organising workloads, not everything in `default`
- **YAML manifests** — stop using imperative commands, write proper deployment files
- **Helm** — deploy a real application (monitoring stack: Prometheus + Loki + Grafana)
- **Ingress** — proper way to expose services instead of NodePort
- **Monitoring VM** — Rocky Linux 9 + Podman + Quadlets + Traefik
