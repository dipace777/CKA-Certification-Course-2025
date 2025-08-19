# Day 54: Build a Multi-Node Kubernetes Cluster with kubeadm + Calico Operator | CKA Course 2025

## Video reference for Day 54 is the following:


---
## ⭐ Support the Project  
If this **repository** helps you, give it a ⭐ to show your support and help others discover it! 

---

## Table of Contents

* [Lecture Prerequisites](#lecture-prerequisites)

  * [What to focus on (5-minute checklist)](#what-to-focus-on-5-minute-checklist)
* [Demo: Build a Multi-Node Kubernetes Cluster with kubeadm (v1.32)](#demo-build-a-multi-node-kubernetes-cluster-with-kubeadm-v132)
  * [Demo Pre-requisites](#demo-pre-requisites)

    * [1) Networking & Security Groups](#1-networking--security-groups)

      * [control-plane-sg (inbound)](#control-plane-sg-inbound)
      * [data-plane-sg (inbound)](#data-plane-sg-inbound)
    * [2) SSH key pair](#2-ssh-key-pair)
    * [3) Instances](#3-instances)
    * [4) Hostnames (clarity only)](#4-hostnames-clarity-only)
    * [5) How to follow the steps (read this first)](#5-how-to-follow-the-steps-read-this-first)
  * [Step 1: Disable swap & set kernel networking $ALL NODES$](#step-1-disable-swap--set-kernel-networking-all-nodes)
  * [Step 2: Install and configure containerd $ALL NODES$](#step-2-install-and-configure-containerd-all-nodes)
  * [Step 3: Install kubeadm, kubelet, kubectl (v1.32) $ALL NODES$](#step-3-install-kubeadm-kubelet-kubectl-v132-all-nodes)
  * [Step 4: Initialize the control plane $CONTROL PLANE ONLY$](#step-4-initialize-the-control-plane-control-plane-only)
  * [Step 5: Install Calico CNI via Operator (defaults) $CONTROL PLANE ONLY$](#step-5-install-calico-cni-via-operator-defaults-control-plane-only)
  * [Step 6: Join the workers $WORKER ONLY$](#step-6-join-the-workers-worker-only)
  * [Step 7: Verify & quick demo $any node with kubeconfig$](#step-7-verify--quick-demo-any-node-with-kubeconfig)
* [If something flaps (quick stabilizers)](#if-something-flaps-quick-stabilizers)
* [Reset to a clean slate (lab-safe)](#reset-to-a-clean-slate-lab-safe)
* [Conclusion](#conclusion)
* [References](#references)

---

## Lecture Prerequisites

Before this demo, do a quick refresher on Kubernetes basics from **Day 7**:

* **Watch:** *Kubernetes Architecture (Day 7)* — concise walkthrough of control plane & node components
  👉 [https://www.youtube.com/watch?v=-9Cslu8PTjU\&ab\_channel=CloudWithVarJosh](https://www.youtube.com/watch?v=-9Cslu8PTjU&ab_channel=CloudWithVarJosh)
* **Skim:** Day 7 notes & examples (repo)
  👉 [https://github.com/CloudWithVarJosh/CKA-Certification-Course-2025/tree/main/Day%2007](https://github.com/CloudWithVarJosh/CKA-Certification-Course-2025/tree/main/Day%2007)

#### What to focus on (5-minute checklist)

* Control plane pieces: **API server, etcd, controller manager, scheduler**
* Node pieces: **kubelet, kube-proxy, container runtime**
* Networking at a glance: **Pods, Services**, and why we need a **CNI** (Calico)

If those terms ring a bell, you’re ready for Day 54.

---

# Demo: Build a Multi-Node Kubernetes Cluster with kubeadm (v1.32)

In this demo we’ll stand up a small, real cluster on VMs: one control-plane node and two worker nodes, all on Ubuntu with containerd and Calico for networking. I’ll show it on **cloud** VMs, but **any VMs work** (on-prem, home lab, or a laptop hypervisor) as long as they can reach each other and you can SSH in. We’ll set up the control plane first, make sure it’s good, then add the workers and deploy a tiny test app end-to-end. Every command block says exactly where to run it — **\[ALL NODES]**, **\[CONTROL PLANE ONLY]**, or **\[WORKER ONLY]**. If you’re not on AWS, treat “security groups” as your firewall rules/NSGs with the same ports.


---

## Demo Pre-requisites

### 1) Networking & Security Groups

![Alt text](/images/54a.png)

Create two SGs: **control-plane-sg** and **data-plane-sg** (same VPC). Allow **all egress** on both.

#### control-plane-sg (inbound)

| Purpose        | Protocol / Port | Source            | When                                        |
| -------------- | --------------- | ----------------- | ------------------------------------------- |
| SSH            | TCP 22          | Your IP           | Always                                      |
| Kubernetes API | TCP 6443        | **data-plane-sg** | Always                                      |
| VXLAN overlay  | **UDP 4789**    | **data-plane-sg** | Recommended (Calico defaults run on CP too) |

> If you will **never** run pods on the control plane (and pin Calico components away from it), you can omit **UDP 4789** here. For most demos, keep it.

---

#### data-plane-sg (inbound)

| Purpose         | Protocol / Port | Source               | When                                              |
| --------------- | --------------- | -------------------- | ------------------------------------------------- |
| SSH             | TCP 22          | Your IP              | Always                                            |
| Kubelet API     | TCP 10250       | **control-plane-sg** | Always                                            |
| VXLAN overlay   | **UDP 4789**    | **data-plane-sg**    | Always (worker ↔ worker)                          |
| VXLAN overlay   | **UDP 4789**    | **control-plane-sg** | Always (CP ↔ worker)                              |
| NodePort (demo) | TCP 30000–32767 | Your IP              | Optional (only for testing NodePort from outside) |

---

**Azure/GCP:** use NSGs / VPC firewall rules with the **same ports/protocols**.
**Also ensure** instances have outbound internet (NAT GW or public EIP) so images can pull.
**HA note (optional):** multi-CP clusters also need: **TCP 2379–2380** (etcd), **TCP 10257** (controller-manager), **TCP 10259** (scheduler) between control-plane nodes.


---

### 2) SSH key pair

**Recommended:** **create the key pair in your cloud provider’s console.**
This is the easiest path because the provider injects your **public key** at boot—no manual copy to `~/.ssh/authorized_keys`.

* **AWS (easy path):** EC2 → **Key Pairs** → **Create key pair** → Type: **ED25519** → download `.pem`

  ```bash
  chmod 400 ~/Downloads/kubeadm-demo.pem
  ssh -i ~/Downloads/kubeadm-demo.pem ubuntu@<PUBLIC_IP_OR_DNS>
  ```
* **Azure/GCP:** Create an SSH key in the VM create flow (Azure “SSH public key”, GCP “SSH Keys/OS Login”). The platform injects it automatically.

**Alternative (local key you generate):**
If you generate a key locally, you must **import the public key** to your cloud account **or** copy it into each VM’s `authorized_keys`.

```bash
# generate locally
ssh-keygen -t ed25519 -C "kubeadm-demo" -f ~/.ssh/kubeadm-demo
chmod 400 ~/.ssh/kubeadm-demo


# copy to a running VM using ssh-copy-id (requires you can already SSH to it)
ssh-copy-id -i ~/.ssh/kubeadm-demo.pub ubuntu@<PUBLIC_IP_OR_DNS>

# then connect using your private key
ssh -i ~/.ssh/kubeadm-demo ubuntu@<PUBLIC_IP_OR_DNS>
```

> **TL;DR:** **Use the provider-generated key pair if possible**—it skips the whole “copy the public key into `authorized_keys`” step.


When connecting:

```bash
ssh -i ~/.ssh/kubeadm-demo ubuntu@<PUBLIC_IP_OR_DNS>
```

---

### 3) Instances

* **OS:** Ubuntu **22.04 or 24.04** LTS (amd64)
* **Sizing:**

  * Control plane: **2 vCPU, 4–8 GB RAM**, 20 GB disk (8 GB works for quick demos)
  * Workers: **1–2 vCPU, 2–4 GB RAM**, 8–20 GB disk
* Place all nodes in the **same VPC** and **routable subnets**.

> kubeadm requires **≥2 vCPU** on the control plane (you *can* bypass with `--ignore-preflight-errors=NumCPU`, but keep 2 vCPU for a clean demo).

---

### 4) Hostnames (clarity only)

On each node (adjust name):

```bash
sudo hostnamectl set-hostname control-plane   # or worker-1 / worker-2
exec bash    # reload the current shell so hostname/prompt and new env settings take effect (use 'sudo reboot' if it still doesn’t reflect)
```

*(Optional)* Add friendly entries to `/etc/hosts` if you want to `ping` by name; Kubernetes doesn’t require it.

---

### 5) How to follow the steps (read this first)

* **Only run commands where the heading says.** Each block is labeled **\[ALL NODES]**, **\[CONTROL PLANE ONLY]**, or **\[WORKER ONLY]**. Stick to that label.
* **Recommended order:** finish the **control plane** first, then do the **workers**.

  * In the control-plane section there’s a short “check” at the end. **Run that check.** If it looks good (no errors), move on to the worker section.
* **Stay organized:** keep one terminal tab per node, copy/paste exactly, and replace placeholders (like `<CP_PRIVATE_IP>`) with your values.
* **Use `sudo` when shown** and don’t skip steps—even if something “looks done.”

---



## Step 1: Disable swap & set kernel networking \[ALL NODES]

```bash
# Disable swap (required by kubelet)
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Kernel modules & sysctls for container networking
echo -e "overlay\nbr_netfilter" | sudo tee /etc/modules-load.d/k8s.conf
sudo modprobe overlay && sudo modprobe br_netfilter

cat <<'EOF' | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF
sudo sysctl --system
```

**Why:**

* **Swap off** → kubelet’s resource management assumes no swap.
* **`overlay`** → enables overlay filesystem used by container images.
* **`br_netfilter` + bridge sysctls** → let iptables see bridged traffic (pods/Services).
* **`ip_forward=1`** → allow the node to route pod traffic.

---

## Step 2: Install and configure containerd \[ALL NODES]

```bash
# Install containerd
sudo apt-get update && sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null

# Align containerd with kubelet expectations
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo sed -i 's#sandbox_image = ".*"#sandbox_image = "registry.k8s.io/pause:3.9"#' /etc/containerd/config.toml

sudo systemctl daemon-reload
sudo systemctl enable --now containerd
sudo systemctl status containerd
```

**Why:**

* **containerd** is the container runtime kubelet talks to.
* **`SystemdCgroup=true`** matches kubelet’s cgroup driver, avoiding cgroup errors.
* **Pause image (`pause:3.9`)** consistency prevents needless pod sandbox churn.

---

## Step 3: Install kubeadm, kubelet, kubectl (v1.32) \[ALL NODES]

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key \
 | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' \
 | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update && sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable kubelet

# Verify
kubeadm version
kubelet --version
kubectl version
```

**Why:**

* **kubeadm** bootstraps the cluster; **kubelet** runs pods; **kubectl** is the CLI.
* Holding versions keeps your demo stable.

> *Note:* `kubectl` on workers is optional—useful for troubleshooting. In this demo we install it everywhere.

### Bash completion (optional, helpful)

```bash
sudo apt-get update && sudo apt-get install -y bash-completion
echo 'source /usr/share/bash-completion/bash_completion' >> ~/.bashrc
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -F __start_kubectl k' >> ~/.bashrc
source ~/.bashrc
```

---

## Step 4: Initialize the control plane \[CONTROL PLANE ONLY]

```bash
# Replace with your control plane node's private IP
sudo kubeadm init \
  --control-plane-endpoint=<CP_PRIVATE_IP>:6443 \
  --apiserver-advertise-address=<CP_PRIVATE_IP> \
  --pod-network-cidr=192.168.0.0/16
```

```bash
# Kubeconfig for the current user
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**Why:**

* **`--control-plane-endpoint`** is the address other nodes/clients use.
* **`--apiserver-advertise-address`** is the local IP the API server binds to.
* **Pod CIDR** must match your CNI (we’ll use 192.168.0.0/16 with Calico).

---

## Step 5: Install Calico CNI via **Operator** (defaults) \[CONTROL PLANE ONLY]

```bash
# 1) Install the operator and CRDs
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.2/manifests/operator-crds.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.2/manifests/tigera-operator.yaml

# 2) Apply a minimal Installation CR (no calico-apiserver)
cat <<'EOF' | kubectl apply -f -
apiVersion: operator.tigera.io/v1        # Calico Operator API group/version
kind: Installation                        # Cluster-wide Calico install spec
metadata:
  name: default                           # Must be named 'default' (singleton)
spec:
  calicoNetwork:                          # Calico networking settings
    ipPools:
    - cidr: 192.168.0.0/16                # Pod CIDR (matches kubeadm --pod-network-cidr)
      natOutgoing: Enabled                # SNAT pod→external traffic at node egress
      blockSize: 26                       # Per-node IP block size (/26 = 64 pod IPs)
      encapsulation: VXLANCrossSubnet     # Use VXLAN; skip encapsulation within same L2 subnet
      nodeSelector: all()                 # Apply this pool to all nodes
EOF
```

**Why:**

* This is the **official, operator-managed install**—simple now and easier to upgrade later.
* The provided `custom-resources.yaml` applies **sensible defaults** that work with kubeadm clusters (including the `192.168.0.0/16` pod network used here), so **no customization is required**.
* Once installed, Calico provides **networking for pods and Services**, allowing system pods (like CoreDNS) and your apps to start communicating.

---

## Step 6: Join the workers \[WORKER ONLY]

```bash
# On the control plane, print the fresh join command:
kubeadm token create --print-join-command
```

Run the printed command **exactly** on each worker, for example:

```bash
sudo kubeadm join <CP_PRIVATE_IP>:6443 --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

**Why:**

* This securely bootstraps the worker, registers it, and downloads cluster certs.

---

## Step 7: Verify & quick demo \[any node with kubeconfig]

```bash
# See nodes and system pods
kubectl get nodes -o wide
kubectl -n kube-system get pods -o wide

# Tiny demo app + Service
kubectl create deploy web --image=nginx --replicas=3
kubectl expose deploy/web --port=80 --type=NodePort

# Discover the NodePort
kubectl get svc web -o wide
kubectl get svc web -o jsonpath='{.spec.ports[0].nodePort}'; echo

# Test from your machine (security groups must allow it)
curl -I http://<worker-1-public-or-private-ip>:<nodeport>
curl -I http://<worker-2-public-or-private-ip>:<nodeport>
```

---

## If something flaps (quick stabilizers)

```bash
# Align containerd with kubelet and refresh once
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo sed -i 's#sandbox_image = ".*"#sandbox_image = "registry.k8s.io/pause:3.9"#' /etc/containerd/config.toml
sudo systemctl restart containerd && sudo systemctl restart kubelet

# Nudge kube-proxy once (harmless if already healthy)
kubectl -n kube-system rollout restart ds/kube-proxy
```

---

## Reset to a clean slate (lab-safe)

> **Order matters:** reset **workers first**, then the **control plane**.

```bash
# WORKERS
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d /var/lib/cni /var/lib/kubelet/pki ~/.kube
for i in cni0 vxlan.calico tunl0; do sudo ip link del "$i" 2>/dev/null || true; done
sudo systemctl restart containerd

# CONTROL PLANE
sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes /var/lib/etcd /etc/cni/net.d /var/lib/cni /var/lib/kubelet/pki ~/.kube
for i in cni0 vxlan.calico tunl0; do sudo ip link del "$i" 2>/dev/null || true; done
sudo systemctl restart containerd
```

---

## Conclusion

You just built a clean, repeatable multi-node Kubernetes **v1.32** cluster with **containerd** and **Calico**. We brought up the control plane first, confirmed it was healthy, then joined workers and validated cross-node traffic with a NodePort service. Along the way you aligned containerd’s cgroups with kubelet, set essential kernel networking flags, and applied Calico in the encapsulation mode you prefer (IP-in-IP or VXLAN) with the right firewall/SG openings.

---

## References

* Installing kubeadm (pkgs.k8s.io repo, version-pinned) — official guide: [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
* Container runtimes — required kernel modules/sysctls and runtime guidance: [https://kubernetes.io/docs/setup/production-environment/container-runtimes/](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
* Configure kubelet ↔ runtime cgroup driver (systemd) — why `SystemdCgroup=true` matters: [https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/)
* Change Kubernetes package repository (pin/minor selection) — using per-minor repos like v1.32: [https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/change-package-repository/](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/change-package-repository/)
* Kubernetes Services & NodePort — what a Service is and how NodePort exposes it: [https://kubernetes.io/docs/concepts/services-networking/service/](https://kubernetes.io/docs/concepts/services-networking/service/)
* `kubeadm reset` — what gets cleaned on control plane/worker nodes: [https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-reset/](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-reset/)
* Calico system/network requirements — ports & protocols: [https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements](https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements)
* Calico overlay modes — overview of IP-in-IP vs VXLAN: [https://docs.tigera.io/calico/latest/networking/configuring/vxlan-ipip](https://docs.tigera.io/calico/latest/networking/configuring/vxlan-ipip)
* Calico install (operator/manifest) — on-prem/self-managed flow: [https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises)
