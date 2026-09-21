# Configure MIG On The H200 GPU

*Configures Multi-Instance GPU (MIG) on the GPU available on the GPU node to split it into isolated, 3 GPU (1 big and 2 small), preparing the cluster to run multiple AI workloads on a single GPU.*


---

## Description

This sub-repo provides a step-by-step guide to partition the H200 SXM GPU on the GPU node using [Multi-Instance GPU (MIG)](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/index.html), configured through the NVIDIA GPU Operator's MIG manager.

The focus of this sub-repo is to check which MIG profiles the H200 (the GPU available in this Demo) supports, define a custom mixed configuration that splits the GPU into 1 big instance and 2 small instances, apply it through the GPU Operator, and confirm Kubernetes exposes all 3 instances as separate, schedulable resources.

The H200 141GB supports MIG profiles from `1g.18gb` up to `7g.141gb` ([NVIDIA MIG User Guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/supported-mig-profiles.html)). This demo applies a custom `mixed MIG strategy` layout that fills almost the whole card:
- 1 big instance: `4g.71gb` (4/7 compute, ~71 GB)
- 2 small instances: `1g.35gb` each (1/7 compute, ~35 GB each)

*Note: The MIG split used in this demo (1 big instance + 2 small instances) is for demo purposes only. In production, this exact split is unlikely to be the right one — a production MIG strategy needs to be worked out from the available hardware and the actual workloads it will run, not copied from a demo.*

This guide follows NVIDIA's own [MIG support in the GPU Operator guide](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-operator-mig.html).

This guide builds a demo cluster and is not intended for a production environment. -- This is for demo purposes only. **Do not use this in a production environment.**

---

## Prerequisites

- Logged in to the master node, with `kubectl` working: ```bash kubectl get nodes``` - All 3 nodes should show `Ready`.
- The NVIDIA GPU Operator is already installed and the GPU is exposed on GPU node (see [01-Install-GPU-Operator/README.md](/02-Demos/01-Install-GPU-Operator/README.md)).

---

## Configuration Flow

This guide follows this order:

1. Create the namespace for this demo
2. Confirm the GPU is currently exposed as a single, unpartitioned device
3. Check which MIG profiles the H200 GPU (available in this demo) supports
4. Set the GPU Operator's MIG strategy to mixed
5. Confirm the MIG strategy patch was applied
6. Create the custom MIG configuration file
7. Load the configuration into a ConfigMap
8. Point the MIG manager at the new ConfigMap
9. Label the GPU node to apply the profile
10. Watch the MIG manager apply the change
11. Confirm the MIG configuration succeeded
12. Confirm the node advertises 3 separate MIG instances
13. Deploy and check a test pod on the big instance
14. Deploy and check a test pod on the small instance
16. Clean up the test pods

---

## How to Use

### Step 1 — Confirm the GPU is currently exposed as a single, unpartitioned device

On the master node:

```bash
kubectl describe node kube-ai-demo-worker-gpu-02 | grep -A16 "Capacity:"
```

> You should see `nvidia.com/gpu: 1` and no `nvidia.com/mig-*` entries yet.

![step1](/02-Demos/02-Configure-MIG/Image/step-1.png)

---

### Step 2 — Check which MIG profiles the H200 supports

On the GPU worker node:

```bash
nvidia-smi mig -lgip
```

> This lists every MIG profile the GPU hardware supports, with the memory and compute (SM) fraction each one consumes. This is a hardware capability query — it works whether or not MIG mode is currently enabled, so it's safe to run before making any changes. Use this output to decide the split in Step 4; the layout in this demo (`4g.71gb` + `1g.35gb` + `1g.35gb`) is one valid combination out of what this command shows.

![step2](/02-Demos/02-Configure-MIG/Image/step-2.png)

---

### Step 3 — Set the GPU Operator's MIG strategy to mixed

On the master node:

The `MIG manager` is deployed as part of the `GPU Operator`, and by default it uses the `single` strategy. `single` only lets the GPU be split using one profile at a time — every instance on the GPU has to be the same size. In this demo we want to see different profile sizes side by side on the same GPU, so we need the `mixed` strategy instead, which allows different profiles to coexist on the same GPU.

The GPU Operator's components are configured through the `ClusterPolicy` custom resource (CRD) — the single object that controls how the Operator deploys and configures the driver, toolkit, device plugin, MIG manager, and the rest. We apply the strategy change by patching this `ClusterPolicy` directly, rather than editing it interactively.

```bash
kubectl patch clusterpolicies.nvidia.com/cluster-policy --type='json' \
  -p='[{"op":"replace", "path":"/spec/mig/strategy", "value":"mixed"}]'
```

*Note: `mixed` strategy is needed here because we're running one big instance and two small instances side by side on the same card — each size is then advertised as its own resource (e.g. `nvidia.com/mig-4g.71gb`, `nvidia.com/mig-1g.35gb`).*

> You can use this command to check if the patching succeeded `kubectl get clusterpolicies.nvidia.com/cluster-policy -o jsonpath='{.spec.mig.strategy}'`

![step3](/02-Demos/02-Configure-MIG/Image/step-3.png)

---

### Step 4 — Create the custom MIG configuration file

On the master node:

The GPU Operator ships a default ConfigMap (`default-mig-parted-config`) with a set of ready-made MIG layouts — things like `all-disabled`, `all-1g.18gb`, or `all-balanced`. None of these gives us the exact split this demo wants: `1 big instance plus 2 small instances` on the same GPU. So instead of picking one of the defaults, we write our own `mig-parted` config file.

This file follows the same format the MIG manager already understands: under `mig-configs`, we give our layout a name (`1big-2small`) and, for the GPU device on this node (`devices: [0]`), list the profiles and how many of each we want — one `4g.71gb` and two `1g.35gb`. Step 7 loads this file into a ConfigMap in the `gpu-operator` namespace, and Step 8 points the MIG manager at it so it knows this custom layout exists alongside the built-in ones.


```bash
cat <<EOF > custom-mig-config.yaml
version: v1
mig-configs:
  1big-2small:
    - devices: [0]
      mig-enabled: true
      mig-devices:
        "4g.71gb": 1
        "1g.35gb": 2
EOF
```

![step4](/02-Demos/02-Configure-MIG/Image/step-4.png)

> *Note: An alternative to creating a separate ConfigMap is editing the existing `default-mig-parted-config` ConfigMap directly and adding this layout to it — that skips the ClusterPolicy patch to `migManager.config.name` entirely. We use a separate ConfigMap here to keep the custom layout out of an Operator-managed object, which can get overwritten on upgrade.*

---

### Step 5 — Load the configuration into a ConfigMap in the gpu-operator namespace

On the master node:

Kubernetes objects reference config as ConfigMaps, not raw files, so we load the `custom-mig-config.yaml` file from Step 4 into one, in the same namespace as the GPU Operator. Step 6 points the MIG manager at it by name.


```bash
kubectl create configmap custom-mig-config -n gpu-operator --from-file=config.yaml=custom-mig-config.yaml
```

![step5](/02-Demos/02-Configure-MIG/Image/step-5.png)

---

### Step 6 — Point the GPU Operator's MIG manager at the new ConfigMap

On the master node:

```bash
kubectl patch clusterpolicies.nvidia.com/cluster-policy --type='json' \
  -p='[{"op":"replace", "path":"/spec/migManager/config/name", "value":"custom-mig-config"}]'
```

> You can use this command to check if the patching succeeded `kubectl get clusterpolicies.nvidia.com/cluster-policy -o jsonpath='{.spec.migManager.config.name}'`


![step6](/02-Demos/02-Configure-MIG/Image/step-6.png)

---

### Step 7 — Label the GPU node with the profile to apply

On the master node:

```bash
kubectl label node kube-ai-demo-worker-gpu-02 nvidia.com/mig.config=1big-2small --overwrite
```

>*Note: this label is what triggers the repartition. The MIG manager pod on that node will cordon it, drain any GPU workloads, and reconfigure the card. This can take a minute or two.*

![step7](/02-Demos/02-Configure-MIG/Image/step-7.png)

---

### Step 8 — Watch the MIG manager apply the change

On the master node:

```bash
kubectl get pods -n gpu-operator -l app=nvidia-mig-manager -w
```

> Wait for the pod to settle back into `Running`, then press `Ctrl+C`.

![step8](/02-Demos/02-Configure-MIG/Image/step-8.png)

---

### Step 9 — Confirm the configuration succeeded

On the master node:

```bash
kubectl get node kube-ai-demo-worker-gpu-02 -o jsonpath='{.metadata.labels.nvidia\.com/mig\.config\.state}'
```

> Expected output: `success`

![step9](/02-Demos/02-Configure-MIG/Image/step-9.png)

---

### Step 10 — Confirm the node now advertises 3 separate MIG instances

On the master node:

```bash
kubectl describe node kube-ai-demo-worker-gpu-02 | grep -A20 "Capacity:"
```

> You should see `nvidia.com/mig-4g.71gb: 1` and `nvidia.com/mig-1g.35gb: 2`, with `nvidia.com/gpu: 0`.

![step10](/02-Demos/02-Configure-MIG/Image/step-10.png)

---

### Step 11 — Deploy a test pod on the big instance

On the master node:

```bash
kubectl create ns mig-demo
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: mig-test-big
  namespace: mig-demo
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
        nvidia.com/mig-4g.71gb: 1
EOF
```

![step11](/02-Demos/02-Configure-MIG/Image/step-11.png)

---

### Step 12 — Check the big instance test pod's output

On the master node:

```bash
kubectl logs mig-test-big -n mig-demo
```

> You should see `nvidia-smi` reporting a single MIG device sized around 71 GB.

![step12](/02-Demos/02-Configure-MIG/Image/step-12.png)

---

### Step 13 — Deploy a test pod on the first small instance

On the master node:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: mig-test-small-1
  namespace: mig-demo
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
        nvidia.com/mig-1g.35gb: 1
EOF
```

![step13](/02-Demos/02-Configure-MIG/Image/step-13.png)

---

### Step 14 — Check the first small instance test pod's output

On the master node:

```bash
kubectl logs mig-test-small-1 -n mig-demo
```

> You should see `nvidia-smi` reporting a MIG device sized around 35 GB. Note the GPU UUID shown — Step 16 compares against it.

![step14](/02-Demos/02-Configure-MIG/Image/step-14.png)

---

### Step 15 — Clean up the test pods

On the master node:

```bash
kubectl delete pod mig-test-big mig-test-small-1 -n mig-demo
```

---

Enjoy
