# Deploy The NVIDIA GPU Operator

*Installs the NVIDIA GPU Operator on the GPU worker node via Helm, checking first whether the NVIDIA driver and CUDA toolkit are already present on the node so the Operator can be told to skip re-installing them.*

---

## Description

This demo deploys the [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/index.html) using its Helm chart. The GPU Operator automates everything Kubernetes needs to expose and use the GPU on `kube-ai-demo-worker-gpu-02`: the NVIDIA driver, the container toolkit, the device plugin, DCGM monitoring, and node feature discovery.

Before installing, this demo checks the GPU node for a pre-existing NVIDIA driver and CUDA toolkit. If either is already present, the Helm install is configured to skip that component so the Operator does not try to install its own copy on top of it.

This is for demo purposes only. **Do not use this in a production environment.**

---

## Prerequisites

- Logged in to the master node, with `kubectl` working:

```bash
kubectl get nodes
```

All 3 nodes should show `Ready`.

- SSH access to the GPU worker node (`kube-ai-demo-worker-gpu-02`) for the driver and CUDA checks.
- Helm installed on the master node (see [01-Install-Kubernetes/README.md](/01-Install-Kubernetes/README.md), Step 17).

---

## How to Use

### Step 1 — Create the namespace

On the master node:

```bash
kubectl create namespace gpu-operator
```

![step1](/02-Demos/01-Install-GPU-Operator/Image/step-1.png)

---

### Step 2 — Check if the NVIDIA driver is already installed on the GPU node

On the GPU worker node:

```bash
nvidia-smi
```

- Output showing a driver version and GPU table → driver **is installed**.
- `command not found` → driver **is not installed**.

Note the result — it decides the flag used in Step 7.

![step2](/02-Demos/01-Install-GPU-Operator/Image/step-2.png)

---

### Step 3 — Check if the CUDA toolkit is already installed on the GPU node

On the GPU worker node:

```bash
nvcc --version
```

- Output showing a CUDA version → toolkit **is installed**.
- `command not found` → toolkit **is not installed**.

Note the result — it decides the flag used in Step 7.

![step3](/02-Demos/01-Install-GPU-Operator/Image/step-3.png)


> **Note:** Before installing, cross-check the driver, CUDA, OS and Kubernetes versions found in Steps 2–3 against the [GPU Operator Platform Support matrix](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/platform-support.html) and [Release Notes](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/release-notes.html). Confirm:
> - The driver branch (e.g. R580) is listed as supported.
> - The Kubernetes version (`kubectl get nodes -o wide`) falls inside the supported range for your OS.
> - The GPU model is listed as validated for the GPU Operator version you're installing.
>
> If the driver is already installed on the node (as checked in Step 2), the OS/kernel precompiled-driver compatibility does not matter — the Operator will be told to skip installing its own driver (`driver.enabled=false` in Step 7), so it just uses what's already running.

---

### Step 4 — Add the NVIDIA Helm repository and update Helm repositories

On the master node:

```bash
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update
```

![step4](/02-Demos/01-Install-GPU-Operator/Image/step-4.png)

---

### Step 5 — Confirm the GPU Operator chart is available

On the master node:

```bash
helm search repo nvidia/gpu-operator
```

You should see the chart listed with its version.

![step5](/02-Demos/01-Install-GPU-Operator/Image/step-5.png)

---

### Step 6 — Install the GPU Operator

On the master node. Pick the command below based on what Steps 2 and 3 found.

**If both the driver and the CUDA toolkit are already installed on the node:**

```bash
helm install gpu-operator nvidia/gpu-operator \
  -n gpu-operator \
  --set driver.enabled=false \
  --set toolkit.enabled=false
```

**If only the driver is already installed:**

```bash
helm install gpu-operator nvidia/gpu-operator \
  -n gpu-operator-demo \
  --set driver.enabled=false
```

**If neither is installed:**

```bash
helm install gpu-operator nvidia/gpu-operator \
  -n gpu-operator-demo
```

![step6](/02-Demos/01-Install-GPU-Operator/Image/step-6.png)

---

### Step 7 — Watch the Operator pods come up

On the master node:

```bash
kubectl get pods -n gpu-operator -w
```

Wait until every pod shows `Running` or `Completed` (validator pods complete and exit). Press `Ctrl+C` once stable.

![step7](/02-Demos/01-Install-GPU-Operator/Image/step-7.png)

---

### Step 8 — Confirm the GPU is exposed on the node

On the master node:

```bash
kubectl describe node kube-ai-demo-worker-gpu-02 | grep -A5 "Capacity:"
```

You should see `nvidia.com/gpu: 1` under capacity and allocatable.

![step8](/02-Demos/01-Install-GPU-Operator/Image/step-8.png)

---

### Step 9 — Deploy a test pod to validate GPU access

On the master node:

```bash
kubectl create namespace gpu-operator-demo
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: gpu-operator-test
  namespace: gpu-operator-demo
spec:
  restartPolicy: Never
  nodeSelector:
    kubernetes.io/hostname: kube-ai-demo-worker-gpu-02
  containers:
  - name: cuda-check
    image: nvidia/cuda:12.4.1-base-ubuntu22.04
    command: ["nvidia-smi"]
    resources:
      limits:
        nvidia.com/gpu: 1
EOF
```

![step9](/02-Demos/01-Install-GPU-Operator/Image/step-9.png)

---

### Step 10 — Check the test pod output

On the master node:

```bash
kubectl logs gpu-operator-test -n gpu-operator-demo
```

You should see the same GPU table as in Step 2 — the pod is scheduled onto the GPU node and can see the H200 through Kubernetes.

![step10](/02-Demos/01-Install-GPU-Operator/Image/step-10.png)

---

### Step 12 — Clean up the test pod

On the master node:

```bash
kubectl delete pod gpu-operator-test -n gpu-operator-demo
```

---

Enjoy
