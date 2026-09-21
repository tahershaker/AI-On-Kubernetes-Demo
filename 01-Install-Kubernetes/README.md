# Install RKE2 On The Cluster

*Installs RKE2 on the 3 provisioned nodes and brings up the Kubernetes cluster the demos in this repo run on.*

---

## Description

This sub-repo provides a step-by-step guide to install an [RKE2](https://docs.rke2.io/) Kubernetes cluster on the provisioned infrastructure. The provisioned infrastructure is 3 virtual machines, one of which has a GPU (H200 SXM). It is provisioned on `Nebius Cloud`, as described in the main repo's README — provisioning the infrastructure is out of scope for this sub-repo.

To follow the demos in the main repo, you can use RKE2 or any other Kubernetes distribution of your choice. If you choose a different distribution, some commands, code, or configuration steps will need to be adjusted to match the goal of each demo.

The focus of this sub-repo is to install and configure a Kubernetes cluster using RKE2, and confirm the cluster is up and running, as preparation for the demo activities in this repo.

This guide uses the [RKE2 quick start script](https://docs.rke2.io/install/quickstart) to bootstrap each node with its required role.

This guide builds a demo cluster and is not intended for a production environment. -- This is for demo purposes only. **Do not use this in a production environment.**


---

## Architecture 

The diagram below shows a high-level architecture of the provisioned infrastructure: 3 VM nodes, all connected to the same VPC network, each exposed to the internet and configured with both a private IP and a public IP.

![hl-arc](/Image/hl-arch.png)

---

## Environment

The tables below give a high-level summary of the infrastructure and the configuration used in this guide. You can change the configuration, but doing so means paying closer attention to the commands and code below, to make sure they match your changes.

**Node Roles, Hostnames & Resource Configuration**

| Node | Role | Hostname | CPU | Memory | GPU | Disk |
|---|---|---|---|---|---|---|
| Master Node | Control plane | `kube-ai-demo-master-01` | 2 | 8GB | None | 60GB |
| Worker Node 01 | Worker (non-GPU) | `kube-ai-demo-worker-no-gpu-01` | 2 | 16GB | None | 60GB |
| Worker Node 02 | Worker (GPU) | `kube-ai-demo-worker-gpu-02` | 16 | 20GB | 1xH200-SXM | 60GB |

**RKE2 Configuration Values**

| Setting | Value |
|---|---|
| Version | Latest |
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

## Configuration Flow

This guide follows this order:

1. Configure the hostname on each node (master & workers)
2. Create the RKE2 configuration file on each node (master & workers)
3. Install and start RKE2 on the master node
4. Set up `kubectl` access and verify the master node is ready
5. Install and start RKE2 on the worker nodes (non-GPU, then GPU)
6. Verify all 3 nodes have joined the cluster
7. Label the non-GPU worker node, so non-GPU workloads can be pinned to it
8. Install Helm on the master node

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

On the master node, replace `<Node_FQDN>` with the node's FQDN:

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

On the non-GPU worker node, replace `<Node_FQDN>` with the node's FQDN, replace `<MASTER_PRIVATE_IP>` with the master node's private IP:

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

On the GPU worker node, replace `<Node_FQDN>` with the node's FQDN, replace `<MASTER_PRIVATE_IP>` with the master node's private IP:

```bash
mkdir -p /etc/rancher/rke2/ && cat <<EOF > /etc/rancher/rke2/config.yaml
write-kubeconfig-mode: "0644"
node-name: <Node_FQDN>
server: https://<MASTER_PRIVATE_IP>:9345
token: AiDemoRKE2token!!5s84s9f9e3d2f2x3f1
EOF
```

---

### Step 7 — Install RKE2 on the master node using the RKE2 quick start script

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

### Step 11 — Install RKE2 on the non-GPU worker node using the RKE2 quick start script

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

### Step 13 — Install RKE2 on the GPU worker node using the RKE2 quick start script

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