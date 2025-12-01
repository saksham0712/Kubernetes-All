# ⚙️ Kubernetes Worker Node Setup Guide (AWS EC2 + Containerd + Calico)

This guide walks through setting up **Kubernetes worker nodes** and connecting them to your master control plane using `kubeadm`.  
It includes detailed explanations of **what, why, and how** for each step — along with real-world **challenges faced and their fixes**.

---

## 🧠 Overview

A Kubernetes cluster consists of:
- **Control Plane (Master):** Runs the API server, scheduler, controller-manager, etcd, and manages the cluster state.
- **Worker Nodes:** Run your actual applications (Pods), managed by the master via the kubelet agent.

---

## 🪜 Step 1: System Preparation

**What:** Update OS and install base dependencies.  
**Why:** Ensures compatibility with Kubernetes packages and the latest kernel features.

```bash
sudo apt update -y && sudo apt upgrade -y
sudo apt install -y apt-transport-https ca-certificates curl gnupg lsb-release
```

✅ **Explanation:**  
- `apt-transport-https` allows package download via HTTPS.  
- `ca-certificates` ensures secure connections to repos.  
- `curl` and `gnupg` are needed to fetch and verify Kubernetes GPG keys.

---

## 🧩 Step 2: Configure Kernel Modules and Sysctl Parameters

**Why:** Kubernetes networking depends on certain kernel features for inter-pod communication and packet forwarding.

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
```

**Explanation:**
- `overlay` enables container overlay networking.  
- `br_netfilter` allows bridged network traffic to be filtered through iptables.  
- `ip_forward=1` ensures packets can move between pods and nodes.

---

## 🐳 Step 3: Install and Configure containerd

**What:** Install containerd (container runtime).  
**Why:** It’s the low-level runtime that runs Kubernetes containers and manages isolation.

```bash
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

✅ **How it works:**
- `SystemdCgroup = true` ensures kubelet and containerd use the same resource control mechanism.
- This prevents “cgroup driver mismatch” issues that often cause pod startup failures.

---

## 🚀 Step 4: Install Kubernetes Binaries

**What:** Install the Kubernetes components — `kubelet`, `kubeadm`, `kubectl`.  
**Why:** These tools help the node communicate with the cluster.

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update -y
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable kubelet
```

✅ **Explanation:**
- `kubelet`: Node agent that runs pods.
- `kubeadm`: CLI tool to join cluster.
- `kubectl`: Used to manage cluster (optional on workers).

---

## 🔑 Step 5: Join the Cluster

Use the join command from your **master node output** (replace IP, token, and hash):

```bash
sudo kubeadm join 172.31.42.55:6443 --token xpc4jn.5v0po5snbefx8owy   --discovery-token-ca-cert-hash sha256:14911b12d5dfb5fd7768557f7500925eb6006bdeb422e7a9e0327389783af803
```

**What happens here:**
- The node authenticates itself using the token.  
- Master validates it via CA hash.  
- The kubelet service registers this worker node into the cluster.

---

## 🧱 Step 6: Verify Node Status (on Master)

Run this on the master node:

```bash
kubectl get nodes -o wide
```

Expected Output:

| NAME | STATUS | ROLES | AGE | VERSION |
|------|---------|-------|------|----------|
| master | Ready | control-plane | 30m | v1.31.x |
| worker-1 | Ready | <none> | 2m | v1.31.x |

If it shows `NotReady`, wait 1–2 minutes for Calico to allocate pod network interfaces.

---

## 🏷️ Step 7: Label the Worker Node (Optional)

```bash
kubectl label node worker-1 node-role.kubernetes.io/worker=worker
```

This helps you easily identify nodes during scheduling or debugging.

---

## 🧪 Step 8: Test Deployment

Deploy a sample Nginx pod to ensure scheduling works:

```bash
kubectl run nginx --image=nginx --port=80
kubectl get pods -o wide
```

✅ **If `STATUS=Running`, your worker node setup is complete!**

---

# 💥 Common Challenges and Fixes

| Problem | Cause | Fix |
|----------|--------|-----|
| `[ERROR IsPrivilegedUser]` | Tried running `kubeadm join` without `sudo` | Always run `sudo kubeadm join ...` |
| `cp: cannot stat '/etc/kubernetes/admin.conf'` | Worker node doesn’t have `admin.conf` (exists only on master) | Skip this step — not required for worker |
| Node stuck in `NotReady` | Calico network pods initializing | Wait ~1 min or restart kubelet/containerd |
| `The connection to the server was refused` | kubelet couldn’t reach master | Check firewall/Security Group for port 6443 |
| `CrashLoopBackOff` in calico-node | Cgroup mismatch or network CIDR mismatch | Reapply Calico manifest with correct CIDR |
| Worker doesn’t appear in `kubectl get nodes` | Token expired | Generate new join token on master: `kubeadm token create --print-join-command` |

---

# ✅ Final Verification Checklist

| Check | Command | Expected Result |
|--------|----------|-----------------|
| Node joined | `kubectl get nodes` | Worker listed as Ready |
| Services | `sudo systemctl status kubelet` | Active (running) |
| Network | `kubectl get pods -n kube-system` | All Calico pods Running |
| Token validity | `kubeadm token list` | Token present |
| Pod scheduling | `kubectl run nginx` | Pod Running |

---

🎉 **Congratulations, Papa Ji!**  
Your worker node is now successfully connected to your Kubernetes control plane — and your cluster is officially multi-node and production-ready.  
You’ve also mastered debugging real-world issues that most engineers get stuck on! 🚀
