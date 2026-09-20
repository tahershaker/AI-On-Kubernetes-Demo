# Install RKE2 On The Cluster

*Installs RKE2 on the 3 provisioned nodes and brings up the Kubernetes cluster the demos in this repo run on.*

---

## Description

This sub-repo installs [RKE2](https://docs.rke2.io/) on the 3 nodes provisioned for this repo, and brings them up as a working Kubernetes cluster: 1 master and 2 workers (1 non-GPU, 1 GPU).

This repo uses RKE2. If you want to install a different Kubernetes distribution, you can — but you will need to adjust the steps, commands, and files below to match it.

This is for demo purposes only. **Do not use this in a production environment.**

---

## Environment

| Node | Role | Hostname |
|---|---|---|
| Master Node | Control plane | `kube-ai-demo-master-01` |
| Worker Node 01 | Worker (non-GPU) | `kube-ai-demo-worker-no-gpu-01` |
| Worker Node 02 | Worker (GPU) | `kube-ai-demo-worker-gpu-02` |

| Setting | Value |
|---|---|
| CNI | Calico |
| Cluster CIDR | `172.16.0.0/16` |
| Service CIDR | `172.17.0.0/16` |
| Install method | RKE2 quick install |

You can use your own hostnames or CIDRs if you prefer — just adjust the commands below to match.

---

## Prerequisites

- The 3 nodes are provisioned and you are logged in to each as `root` (or a user with `sudo`).
- Outbound internet access from each node.

---

## How to Use

### Step 1 — Set the hostname on the master node

On the master node:

```bash
sudo hostnamectl set-hostname --static kube-ai-demo-master-01
if grep -q "^preserve_hostname" /etc/cloud/cloud.cfg 2>/dev/null; then
  sudo sed -i 's/^preserve_hostname:.*/preserve_hostname: true/' /etc/cloud/cloud.cfg
else
  echo "preserve_hostname: true" | sudo tee -a /etc/cloud/cloud.cfg
fi
exec bash
```

📸 *Screenshot placeholder*

---

### Step 2 — Set the hostname on the non-GPU worker node

On the non-GPU worker node:

```bash
sudo hostnamectl set-hostname --static kube-ai-demo-worker-no-gpu-01
if grep -q "^preserve_hostname" /etc/cloud/cloud.cfg 2>/dev/null; then
  sudo sed -i 's/^preserve_hostname:.*/preserve_hostname: true/' /etc/cloud/cloud.cfg
else
  echo "preserve_hostname: true" | sudo tee -a /etc/cloud/cloud.cfg
fi
exec bash
```

📸 *Screenshot placeholder*

---

### Step 3 — Set the hostname on the GPU worker node

On the GPU worker node:

```bash
sudo hostnamectl set-hostname --static kube-ai-demo-worker-gpu-02
if grep -q "^preserve_hostname" /etc/cloud/cloud.cfg 2>/dev/null; then
  sudo sed -i 's/^preserve_hostname:.*/preserve_hostname: true/' /etc/cloud/cloud.cfg
else
  echo "preserve_hostname: true" | sudo tee -a /etc/cloud/cloud.cfg
fi
exec bash
```

📸 *Screenshot placeholder*

---

### Step 4 — Create the RKE2 config file on the master node

On the master node:

```bash
mkdir -p /etc/rancher/rke2/ && cat <<EOF > /etc/rancher/rke2/config.yaml
write-kubeconfig-mode: "0644"
node-name: kube-ai-demo-master-01
cni: "calico"
cluster-cidr: "172.16.0.0/16"
service-cidr: "172.17.0.0/16"
token: AiDemoRKE2token!!5s84s9f9e3d2f2x3f1
EOF
```

📸 *Screenshot placeholder*

---

### Step 5 — Create the RKE2 config file on the non-GPU worker node

On the non-GPU worker node, replace `<MASTER_PRIVATE_IP>` with the master node's private IP:

```bash
mkdir -p /etc/rancher/rke2/ && cat <<EOF > /etc/rancher/rke2/config.yaml
write-kubeconfig-mode: "0644"
node-name: kube-ai-demo-worker-no-gpu-01
server: https://<MASTER_PRIVATE_IP>:9345
token: AiDemoRKE2token!!5s84s9f9e3d2f2x3f1
EOF
```

📸 *Screenshot placeholder*

---

### Step 6 — Create the RKE2 config file on the GPU worker node

On the GPU worker node, replace `<MASTER_PRIVATE_IP>` with the master node's private IP:

```bash
mkdir -p /etc/rancher/rke2/ && cat <<EOF > /etc/rancher/rke2/config.yaml
write-kubeconfig-mode: "0644"
node-name: kube-ai-demo-worker-gpu-02
server: https://<MASTER_PRIVATE_IP>:9345
token: AiDemoRKE2token!!5s84s9f9e3d2f2x3f1
EOF
```

📸 *Screenshot placeholder*

---

### Step 7 — Install RKE2 on the master node

On the master node:

```bash
curl -sfL https://get.rke2.io | sudo sh -
```

📸 *Screenshot placeholder*

---

### Step 8 — Start RKE2 on the master node

On the master node:

```bash
sudo systemctl enable rke2-server.service --now
```

This can take a few minutes — Step 10 confirms it's ready.

📸 *Screenshot placeholder*

---

### Step 9 — Set up `kubectl` access on the master node

On the master node:

```bash
mkdir -p ~/.kube && sudo cp /etc/rancher/rke2/rke2.yaml ~/.kube/config && sudo chown $(id -u):$(id -g) ~/.kube/config && echo 'export PATH=$PATH:/var/lib/rancher/rke2/bin' >> ~/.bashrc && exec bash
```

📸 *Screenshot placeholder*

---

### Step 10 — Verify the master node is ready

On the master node:

```bash
kubectl get nodes
```

You should see `kube-ai-demo-master-01` in `Ready` state.

📸 *Screenshot placeholder*

---

### Step 11 — Install RKE2 on the non-GPU worker node

On the non-GPU worker node:

```bash
curl -sfL https://get.rke2.io | sudo INSTALL_RKE2_TYPE="agent" sh -
```

📸 *Screenshot placeholder*

---

### Step 12 — Start RKE2 on the non-GPU worker node

On the non-GPU worker node:

```bash
sudo systemctl enable rke2-agent.service --now
```

📸 *Screenshot placeholder*

---

### Step 13 — Install RKE2 on the GPU worker node

On the GPU worker node:

```bash
curl -sfL https://get.rke2.io | sudo INSTALL_RKE2_TYPE="agent" sh -
```

📸 *Screenshot placeholder*

---

### Step 14 — Start RKE2 on the GPU worker node

On the GPU worker node:

```bash
sudo systemctl enable rke2-agent.service --now
```

📸 *Screenshot placeholder*

---

### Step 15 — Verify the full cluster is up

On the master node:

```bash
kubectl get nodes
```

All 3 nodes should show, in `Ready` state: `kube-ai-demo-master-01`, `kube-ai-demo-worker-no-gpu-01`, `kube-ai-demo-worker-gpu-02`.

📸 *Screenshot placeholder*

---

### Step 16 — Label the non-GPU worker node

This pins non-GPU workloads to the non-GPU worker, keeping the GPU worker free for GPU workloads. On the master node:

```bash
kubectl label node kube-ai-demo-worker-no-gpu-01 workload-type=non-gpu
```

📸 *Screenshot placeholder*
