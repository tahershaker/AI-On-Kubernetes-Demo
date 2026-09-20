# Install RKE2 On The Cluster

*Installs RKE2 on the 3 provisioned nodes and brings up the Kubernetes cluster the demos in this repo run on.*

---

## Description

This sub-repo installs [RKE2](https://docs.rke2.io/) on the 3 nodes provisioned for this repo, and brings them up as a working Kubernetes cluster: 1 master and 2 workers (1 non-GPU, 1 GPU).

This repo uses RKE2. If you want to install a different Kubernetes distribution, you can — but you will need to adjust the steps, commands, and files below to match it.

This is for demo purposes only. **Do not use this in a production environment.**

---

## Architecture 

![hl-arc](/Image/hl-arch.png)

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

![step1](/01-Install-Kubernetes/Image/step-1.png)

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

---

### Step 4 — Create the RKE2 config file on the master node

On the master node:

```bash
mkdir -p /etc/rancher/rke2/ && cat <<EOF > /etc/rancher/rke2/config.yaml
write-kubeconfig-mode: "0644"
node-name: <Node_FQDN>
cni: "calico"
cluster-cidr: "172.16.0.0/16"
service-cidr: "172.17.0.0/16"
token: AiDemoRKE2token!!5s84s9f9e3d2f2x3f1
EOF
```

![step4](/01-Install-Kubernetes/Image/step-4.png)

---

### Step 5 — Create the RKE2 config file on the non-GPU worker node

On the non-GPU worker node, replace `<MASTER_PRIVATE_IP>` with the master node's private IP:

```bash
mkdir -p /etc/rancher/rke2/ && cat <<EOF > /etc/rancher/rke2/config.yaml
write-kubeconfig-mode: "0644"
node-name: <Node_FQDN>
server: https://<MASTER_PRIVATE_IP>:9345
token: AiDemoRKE2token!!5s84s9f9e3d2f2x3f1
EOF
```

---

### Step 6 — Create the RKE2 config file on the GPU worker node

On the GPU worker node, replace `<MASTER_PRIVATE_IP>` with the master node's private IP:

```bash
mkdir -p /etc/rancher/rke2/ && cat <<EOF > /etc/rancher/rke2/config.yaml
write-kubeconfig-mode: "0644"
node-name: <Node_FQDN>
server: https://<MASTER_PRIVATE_IP>:9345
token: AiDemoRKE2token!!5s84s9f9e3d2f2x3f1
EOF
```

---

### Step 7 — Install RKE2 on the master node

On the master node:

```bash
curl -sfL https://get.rke2.io | sudo sh -
```

![step7](/01-Install-Kubernetes/Image/step-7.png)

---

### Step 8 — Start RKE2 on the master node

On the master node:

```bash
sudo systemctl enable rke2-server.service --now
```

This can take a few minutes — Step 10 confirms it's ready.

![step8](/01-Install-Kubernetes/Image/step-8.png)

---

### Step 9 — Set up `kubectl` access on the master node

On the master node:

```bash
mkdir -p ~/.kube && sudo cp /etc/rancher/rke2/rke2.yaml ~/.kube/config && sudo chown $(id -u):$(id -g) ~/.kube/config && echo 'export PATH=$PATH:/var/lib/rancher/rke2/bin' >> ~/.bashrc && exec bash
```

![step9](/01-Install-Kubernetes/Image/step-9.png)

---

### Step 10 — Verify the master node is ready

On the master node:

```bash
kubectl get nodes
```

You should see `kube-ai-demo-master-01` in `Ready` state.

![step10](/01-Install-Kubernetes/Image/step-10.png)

---

### Step 11 — Install RKE2 on the non-GPU worker node

On the non-GPU worker node:

```bash
curl -sfL https://get.rke2.io | sudo INSTALL_RKE2_TYPE="agent" sh -
```

![step11](/01-Install-Kubernetes/Image/step-11.png)

---

### Step 12 — Start RKE2 on the non-GPU worker node

On the non-GPU worker node:

```bash
sudo systemctl enable rke2-agent.service --now
```

![step12](/01-Install-Kubernetes/Image/step-12.png)

---

### Step 13 — Install RKE2 on the GPU worker node

On the GPU worker node:

```bash
curl -sfL https://get.rke2.io | sudo INSTALL_RKE2_TYPE="agent" sh -
```

---

### Step 14 — Start RKE2 on the GPU worker node

On the GPU worker node:

```bash
sudo systemctl enable rke2-agent.service --now
```

---

### Step 15 — Verify the full cluster is up

On the master node:

```bash
kubectl get nodes
```

All 3 nodes should show, in `Ready` state: `kube-ai-demo-master-01`, `kube-ai-demo-worker-no-gpu-01`, `kube-ai-demo-worker-gpu-02`.

![step15](/01-Install-Kubernetes/Image/step-15.png)

---

### Step 16 — Label the non-GPU worker node

This label is what lets a non-GPU workload's nodeSelector be pinned to this node, keeping the GPU worker free for GPU workloads — the label on its own does nothing until a pod spec selects it. On the master node:

```bash
kubectl label node kube-ai-demo-worker-no-gpu-01 workload-type=non-gpu
kubectl get node kube-ai-demo-worker-no-gpu-01 --show-labels
```

*Note: The label alone does not move anything — it just tags the node. The pinning happens on the workload side: any deployment or pod that includes nodeSelector: { workload-type: non-gpu } in its spec will only be scheduled onto this node. Demos in this repo that deploy non-GPU workloads will use this selector to keep them off the GPU node.*

![step16](/01-Install-Kubernetes/Image/step-16.png)

---

### Step 17 — Install Helm on the master node

You'll need Helm for demos that deploy via Helm charts (e.g. the GPU Operator). On the master node:

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 -o get_helm.sh && chmod +x get_helm.sh && ./get_helm.sh && rm -f get_helm.sh
```

![step17](/01-Install-Kubernetes/Image/step-17.png)

---

## Next Action

Kubernetes is now up and running on all 3 nodes. You can start the demos — refer to the [02-Demos README](/02-Demos/README.md).

---

Enjoy