# k8lab Helm Setup — Complete Session Summary

## Overview

This document describes the complete process of installing and configuring Helm on a bare-metal Kubernetes lab cluster, and deploying a minimal but enterprise-grade foundation stack of three components. The goal was to **learn Helm** — not build a full production cluster.

---

## Environment

| Machine | Role | IP | OS |
|---|---|---|---|
| dellb-k8client | Operations workstation (Helm, kubectl, Ansible) | 192.168.10.57 | Ubuntu 24.04 |
| dellb-master-01 | Kubernetes control plane | 192.168.10.59 | Rocky Linux 9 |
| dellb-worker-01 | Worker node | 192.168.10.51 | Rocky Linux 9 |
| dellb-worker-02 | Worker node | 192.168.10.52 | Rocky Linux 9 |
| dellb-worker-03 | Worker node | 192.168.10.53 | Rocky Linux 9 |

**Key principle:** All operational tooling (Helm, kubectl, Ansible) runs on `k8client`. The master node runs the Kubernetes control plane only and is not used for day-to-day operations. This mirrors real enterprise practice.

---

## What is Helm and Why Use It

Kubernetes applications are defined as YAML manifests. A single application can require 15–25 separate YAML files covering Deployments, Services, ConfigMaps, RBAC, ServiceAccounts, and more. Managing these manually creates several problems:

- No versioning — there is no way to track what version of an application is running
- No templating — separate environments (dev, staging, prod) require duplicate files
- No rollback — reverting a broken upgrade means manually finding and reapplying old files
- No packaging standard — every team reinvents how to bundle manifests

Helm solves all of these by introducing three concepts:

- **Chart** — a packaged, versioned, templated collection of all manifests for one application
- **Release** — a named, tracked deployment of a chart into the cluster. Every change creates a new revision
- **Repository** — an HTTPS server hosting chart packages, similar to apt repositories

Unlike `apt` on Ubuntu, Helm ships with **zero built-in repositories**. Every repo must be explicitly registered. ArtifactHub (artifacthub.io) is the public search engine for finding charts and their repo URLs.

---

## Step 1 — Install Helm via Ansible

Helm is installed on `k8client` using an Ansible playbook so the installation is reproducible, version-pinned, and can be rerun on any future control node.

### Inventory fix required

Before running any playbook, the Ansible inventory needed one fix. Since `dellb-k8client` is both the machine *running* Ansible and a *target* in the inventory, it would try to SSH to itself. The fix is `ansible_connection=local`:

**`inventory.ini`**
```ini
[masters]
dellb-master-01 ansible_host=192.168.10.59 ansible_user=kubermaster

[k8client]
dellb-k8client  ansible_host=192.168.10.57  ansible_user=k8client  ansible_connection=local
dellc-k8client  ansible_host=192.168.10.67  ansible_user=k8client

[workers]
dellb-worker-01 ansible_host=192.168.10.51 ansible_user=kuber
dellb-worker-02 ansible_host=192.168.10.52 ansible_user=kuber
dellb-worker-03 ansible_host=192.168.10.53 ansible_user=kuber
```

### Passwordless sudo

The playbook uses `become: true` for installing system packages. To avoid being prompted for a password on every run, grant the `k8client` user passwordless sudo once on the VM:

```bash
echo "k8client ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/k8client
sudo chmod 0440 /etc/sudoers.d/k8client
```

### The Helm install playbook

**`install-helm.yml`**
```yaml
---
- name: Install Helm 3 on k8client nodes
  hosts: k8client
  become: true
  gather_facts: true

  vars:
    helm_version: "3.17.1"
    helm_arch_map:
      x86_64: amd64
      aarch64: arm64
    helm_os: linux
    helm_arch: "{{ helm_arch_map[ansible_architecture] | default('amd64') }}"
    helm_binary_url: >-
      https://get.helm.sh/helm-v{{ helm_version }}-{{ helm_os }}-{{ helm_arch }}.tar.gz
    helm_install_dir: /usr/local/bin

    helm_repos:
      - name: metallb
        url: https://metallb.github.io/metallb
      - name: nginx-stable
        url: https://helm.nginx.com/stable
      - name: jetstack
        url: https://charts.jetstack.io

  tasks:

    - name: Assert OS is Ubuntu
      assert:
        that:
          - ansible_distribution == "Ubuntu"

    - name: Check if Helm is already installed
      command: helm version --short
      register: helm_check
      failed_when: false
      changed_when: false

    - name: Parse installed Helm version
      set_fact:
        helm_installed_version: >-
          {{ helm_check.stdout | regex_search('v([0-9]+\.[0-9]+\.[0-9]+)', '\1') | first | default('') }}
      when: helm_check.rc == 0

    - name: Install Helm
      when: helm_check.rc != 0 or helm_installed_version != helm_version
      block:

        - name: Ensure required packages are present
          apt:
            name: [curl, tar]
            state: present
            update_cache: true

        - name: Create temp download directory
          tempfile:
            state: directory
            suffix: _helm_install
          register: helm_tmpdir

        - name: Download Helm tarball
          get_url:
            url: "{{ helm_binary_url }}"
            dest: "{{ helm_tmpdir.path }}/helm.tar.gz"
            mode: "0644"

        - name: Extract Helm binary
          unarchive:
            src: "{{ helm_tmpdir.path }}/helm.tar.gz"
            dest: "{{ helm_tmpdir.path }}"
            remote_src: true

        - name: Install Helm binary
          copy:
            src: "{{ helm_tmpdir.path }}/{{ helm_os }}-{{ helm_arch }}/helm"
            dest: "{{ helm_install_dir }}/helm"
            mode: "0755"
            owner: root
            group: root
            remote_src: true

        - name: Remove temp directory
          file:
            path: "{{ helm_tmpdir.path }}"
            state: absent

    - name: Verify Helm binary works
      command: helm version --short
      register: helm_version_check
      changed_when: false

    - name: Add Helm repos
      become: false
      block:

        - name: Add required Helm repositories
          command: >
            helm repo add {{ item.name }} {{ item.url }} --force-update
          loop: "{{ helm_repos }}"

        - name: Update all Helm repos
          command: helm repo update
          changed_when: false

        - name: List registered repos
          command: helm repo list
          register: repo_list
          changed_when: false

        - name: Show registered repos
          debug:
            msg: "{{ repo_list.stdout_lines }}"
```

Run with:
```bash
ansible-playbook install-helm.yml --limit dellb-k8client
```

**Result:** Helm 3.17.1 installed at `/usr/local/bin/helm` with three repos registered.

---

## Step 2 — Configure kubectl on k8client

Helm talks to Kubernetes through `kubectl`. Both need to be installed on `k8client` and configured with the cluster credentials.

### Install kubectl

```bash
sudo snap install kubectl --classic
```

### Copy kubeconfig from the master

The cluster credentials (kubeconfig) are created on the master during `kubeadm init`. Copy the content to k8client:

On **kubermaster**:
```bash
cat ~/.kube/config
```

On **k8client** — paste the output into the file:
```bash
mkdir -p ~/.kube
nano ~/.kube/config
chmod 600 ~/.kube/config
```

### Verify

```bash
kubectl get nodes
```

All nodes should show as `Ready`.

---

## Step 3 — Install the Foundation Stack

The stack consists of three components installed in strict order. Each depends on the one before it.

```
MetalLB → cert-manager → NGINX Gateway Fabric
(IPs)      (TLS certs)    (traffic routing)
```

### Why these three

- **MetalLB** — on bare metal there is no cloud provider to assign external IPs to LoadBalancer services. Without MetalLB, every service stays stuck at `EXTERNAL-IP: <pending>` forever. MetalLB watches for LoadBalancer services and assigns real LAN IPs from a configured pool.

- **cert-manager** — automatically issues and renews TLS certificates. Without it, every HTTPS service needs manually managed certificates. cert-manager acts as a certificate robot that integrates directly with the gateway.

- **NGINX Gateway Fabric** — the modern replacement for the deprecated Kubernetes Ingress resource. Implements the Gateway API standard (HTTPRoute, GRPCRoute) to route external traffic to services inside the cluster. Gets its external IP from MetalLB and its TLS certificates from cert-manager.

---

### Install MetalLB

```bash
helm install metallb metallb/metallb \
  --namespace metallb-system \
  --create-namespace \
  --wait
```

MetalLB is now running but has no IP pool yet. Apply the pool configuration as a post-install step — this is a common Helm pattern where the chart installs the operator and you configure it separately via CRDs:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: lan-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.10.100-192.168.10.120
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: lan-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
  - lan-pool
EOF
```

**Verify:**
```bash
kubectl get pods -n metallb-system
kubectl get ipaddresspool -n metallb-system
```

Expected result: one `controller` pod and one `speaker` pod per node, all Running. IP pool showing `192.168.10.100-192.168.10.120`.

---

### Install cert-manager

```bash
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true \
  --wait
```

Note: `--set crds.enabled=true` is required. cert-manager ships CRDs separately and will not install them unless explicitly told to. This is a deliberate design choice by the chart author to keep CRD lifecycle under the operator's control.

Create a self-signed ClusterIssuer — this is the post-install configuration step. It tells cert-manager how to issue certificates. For the lab, self-signed is sufficient. In production this would point to Let's Encrypt:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned
spec:
  selfSigned: {}
EOF
```

**Verify:**
```bash
kubectl get pods -n cert-manager
kubectl get clusterissuer
```

Expected result: three pods Running (`cert-manager`, `cainjector`, `webhook`). ClusterIssuer showing `READY: True`.

---

### Install NGINX Gateway Fabric

NGINX Gateway Fabric is distributed via an OCI registry, not a traditional Helm repo. This is a newer Helm feature where charts are stored in container registries. No `helm repo add` is needed — install directly from the URL.

First install the Gateway API CRDs (not included with Kubernetes by default):

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/standard-install.yaml
```

Then install the chart:

```bash
helm install ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric \
  --namespace nginx-gateway \
  --create-namespace \
  --wait
```

Create the GatewayClass and Gateway. The GatewayClass tells the controller which class name to watch for. The Gateway defines the actual listeners (ports 80 and 443). This is the moment MetalLB fires — creating a Gateway triggers a LoadBalancer service which MetalLB immediately assigns a real IP:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx
spec:
  controllerName: gateway.nginx.org/nginx-gateway-controller
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: main-gateway
  namespace: nginx-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    port: 80
    protocol: HTTP
  - name: https
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: main-gateway-tls
EOF
```

**Verify:**
```bash
kubectl get svc -n nginx-gateway
kubectl get gateway -n nginx-gateway
```

Expected result:
```
NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)
main-gateway-nginx     LoadBalancer   10.96.14.206   192.168.10.100   80,443

NAME           CLASS   ADDRESS          PROGRAMMED
main-gateway   nginx   192.168.10.100   True
```

`EXTERNAL-IP: 192.168.10.100` — MetalLB assigned the first IP from the pool.
`PROGRAMMED: True` — NGINX picked up the Gateway and is ready to route traffic.

---

## Final State

```bash
helm list -A
```

| Release | Namespace | Chart | Version | Status |
|---|---|---|---|---|
| metallb | metallb-system | metallb | 6.x | deployed |
| cert-manager | cert-manager | cert-manager | v1.20.0 | deployed |
| ngf | nginx-gateway | nginx-gateway-fabric | 2.4.2 | deployed |

### What is now reachable

`192.168.10.100:80` and `192.168.10.100:443` are live on the LAN. Any device on the `192.168.10.0/24` network can reach the cluster gateway at this address. Adding a new application to the cluster now only requires creating an `HTTPRoute` resource pointing to it — no infrastructure changes needed.

### Traffic flow

```
Browser / client on LAN
        ↓
192.168.10.100 (MetalLB IP on LAN)
        ↓
NGINX Gateway Fabric (routes by hostname/path)
        ↓
cert-manager (provides TLS certificate for HTTPS)
        ↓
Application service inside cluster
```

---

## Helm Concepts Learned

| Concept | Where it appeared |
|---|---|
| `helm repo add` | Adding metallb, jetstack repos |
| `helm repo update` | Refreshing repo indexes |
| OCI registry install | NGINX Gateway Fabric — no repo needed |
| `helm install` with `--namespace --create-namespace` | All three installs |
| `--set` flag for value overrides | cert-manager CRDs flag |
| `--wait` flag | All three installs — blocks until pods are Ready |
| Post-install CRD configuration | MetalLB IPAddressPool, cert-manager ClusterIssuer |
| Release tracking | Each install creates REVISION: 1, stored as a Secret |
| `helm list -A` | Viewing all releases across all namespaces |
