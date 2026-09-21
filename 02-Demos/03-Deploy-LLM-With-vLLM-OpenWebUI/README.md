# Deploy an LLM With vLLM and Connect OpenWebUI

*Deploys an LLM using vLLM on the GPU node, then deploys OpenWebUI on the non-GPU node and connects it to vLLM's OpenAI-compatible API for browser-based chat access.*

---

## Description

This sub-repo provides a step-by-step guide to deploy an LLM using [vLLM](https://docs.vllm.ai/) as an OpenAI-compatible inference server, pinned to the big MIG instance (`4g.71gb`) created in [02-Configure-MIG](/02-Demos/02-Configure-MIG/README.md) on the GPU node. It then deploys [OpenWebUI](https://docs.openwebui.com/) on the non-GPU node and connects it to vLLM's API, giving a browser-based chat interface backed by the GPU.

The focus of this sub-repo is to serve a model from Hugging Face through vLLM, expose it as an OpenAI-compatible API inside the cluster, connect OpenWebUI to that API, and back both vLLM's model cache and OpenWebUI's data with PersistentVolumeClaims rather than node-local directories.

The model used is `Qwen/Qwen2.5-7B-Instruct` — openly licensed on Hugging Face with no gated-access approval needed, and small enough to fit comfortably within the storage and GPU budget of this demo.

*Note: RKE2 does not ship a default StorageClass (unlike K3s). This sub-repo installs [Rancher's local-path-provisioner](https://github.com/rancher/local-path-provisioner) to dynamically provision the PVCs used here — a simple, single-replica, node-local provisioner that is fine for a demo but not what a production cluster would use.*

This guide builds a demo cluster and is not intended for a production environment. -- This is for demo purposes only. **Do not use this in a production environment.**

---

## Prerequisites

- Logged in to the master node, with `kubectl` working: `kubectl get nodes` - All 3 nodes should show `Ready`.
- The MIG split from [02-Configure-MIG](/02-Demos/02-Configure-MIG/README.md) is applied, and `kube-ai-demo-worker-gpu-02` advertises `nvidia.com/mig-4g.71gb: 1`.
- `kube-ai-demo-worker-no-gpu-01` is labeled `workload-type=non-gpu` (see [01-Install-Kubernetes, Step 16](/01-Install-Kubernetes/README.md)).
- Outbound internet access from `kube-ai-demo-worker-gpu-02` — vLLM downloads the model weights from Hugging Face on first start (~15 GB for this model).
- At least ~30 GB free disk on the node(s) backing storage — the local-path-provisioner used here provisions PVCs as directories on the node, under `/opt/local-path-provisioner` by default.

---

## Configuration Flow

This guide follows this order:

1. Create the namespace for this demo
2. Install the local-path storage provisioner
3. Confirm the local-path StorageClass is available
4. Create the vLLM model cache PVC
5. Confirm the PVC is Pending (WaitForFirstConsumer binding)
6. Deploy vLLM on the big MIG GPU instance
7. Watch the vLLM pod come up
8. Confirm the vLLM PVC is now Bound
9. Confirm vLLM finished loading the model
10. Create a Service for vLLM
11. Test the vLLM API from inside the cluster
12. Create the OpenWebUI data PVC
13. Deploy OpenWebUI, pointed at the vLLM Service
14. Watch the OpenWebUI pod come up
15. Expose OpenWebUI with a NodePort Service
16. Open OpenWebUI in a browser
17. Send a test prompt

---

## How to Use

### Step 1 — Create the namespace

On the master node:

```bash
kubectl create namespace vllm-openwebui-demo
```

![step1](/02-Demos/03-Deploy-LLM-With-vLLM-OpenWebUI/Image/step-1.png)

---

### Step 2 — Install the local-path storage provisioner & Confirm the StorageClass is available

On the master node:

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.36/deploy/local-path-storage.yaml
kubectl get storageclass
```

![step2](/02-Demos/03-Deploy-LLM-With-vLLM-OpenWebUI/Image/step-2.png)

> You should see `local-path` listed.

> *Note: RKE2 does not bundle a default StorageClass out of the box. This installs the community-maintained Rancher local-path-provisioner, which creates a `local-path` StorageClass and dynamically provisions each PVC as a directory on whichever node the pod claiming it lands on. This is a simple, single-replica, node-local provisioner — fine for this demo, not what you'd use in production (no replication, tied to one node).*

---

### Step 3 — Create the vLLM model cache PVC & Confirm the PVC is Pending

On the master node:

This creates a dedicated PVC to be mounted into the vLLM pod to be created in Step 4, so it can use it as the Hugging Face cache directory — the downloaded model persists there instead of being re-downloaded on every pod restart.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: vllm-model-cache
  namespace: vllm-openwebui-demo
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 25Gi
EOF
kubectl get pvc -n vllm-openwebui-demo
```

![step3](/02-Demos/03-Deploy-LLM-With-vLLM-OpenWebUI/Image/step-3.png)

> Status should show `Pending`. *Note: `local-path` uses `WaitForFirstConsumer` binding — the underlying volume isn't created until a pod that mounts this PVC is actually scheduled, so it stays Pending until Step 7.*

*Note: mounted into the vLLM pod as the Hugging Face cache directory, so the model is downloaded once and reused across pod restarts. 25Gi comfortably covers this model's ~15 GB of weights with headroom, out of the ~30 GB budget.*

---

### Step 4 — Deploy vLLM on the big MIG instance

On the master node:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm
  namespace: vllm-openwebui-demo
  labels:
    app: vllm
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vllm
  template:
    metadata:
      labels:
        app: vllm
    spec:
      nodeSelector:
        kubernetes.io/hostname: kube-ai-demo-worker-gpu-02
      containers:
      - name: vllm
        image: vllm/vllm-openai:latest
        args:
          - "--model=Qwen/Qwen2.5-7B-Instruct"
          - "--served-model-name=qwen2.5-7b"
          - "--host=0.0.0.0"
          - "--port=8000"
          - "--gpu-memory-utilization=0.90"
          - "--max-model-len=8192"
          - "--enable-auto-tool-choice"
          - "--tool-call-parser=hermes"
        ports:
        - containerPort: 8000
        resources:
          limits:
            nvidia.com/mig-4g.71gb: 1
        volumeMounts:
        - name: hf-cache
          mountPath: /root/.cache/huggingface
        - name: shm
          mountPath: /dev/shm
      volumes:
      - name: hf-cache
        persistentVolumeClaim:
          claimName: vllm-model-cache
      - name: shm
        emptyDir:
          medium: Memory
          sizeLimit: 2Gi
EOF
```

![step4](/02-Demos/03-Deploy-LLM-With-vLLM-OpenWebUI/Image/step-4.png)

> *Note: `nvidia.com/mig-4g.71gb: 1` requests the big MIG instance specifically — the GPU Operator advertises each MIG size as its own resource, so this pod cannot land on a `1g.35gb` slice. `--gpu-memory-utilization` and `--max-model-len` are vLLM's own tuning flags; the values here are conservative starting points.*

---

### Step 5 — Watch the vLLM pod come up

On the master node:

```bash
kubectl get pods -n vllm-openwebui-demo -w
```

> First start pulls the vLLM image (~9 GB) and downloads the model weights — this can take several minutes. Press `Ctrl+C` once the pod shows `Running`.
> Also use this command to confirm the vllm pod is now running on the GPU node and not the Non-GPU node `kubectl get pods -n vllm-openwebui-demo -o wide`

![step5](/02-Demos/03-Deploy-LLM-With-vLLM-OpenWebUI/Image/step-5.png)

---

### Step 6 — Confirm the vLLM PVC is now Bound

On the master node:

```bash
kubectl get pvc -n vllm-openwebui-demo
```

![step6](/02-Demos/03-Deploy-LLM-With-vLLM-OpenWebUI/Image/step-6.png)

> Status should now show `Bound` — the volume was created once the pod above was scheduled onto `kube-ai-demo-worker-gpu-02`.

---

### Step 7 — Confirm vLLM finished loading the model

On the master node:

A `Running` pod status only means the container started — it doesn't mean vLLM has finished downloading the model into the PVC and loading it onto the GPU. Sending requests before that point just gets connection errors. This step tails the pod's logs to catch the actual readiness signal before moving on.


```bash
kubectl logs -n vllm-openwebui-demo -l app=vllm -f
```

![step7](/02-Demos/03-Deploy-LLM-With-vLLM-OpenWebUI/Image/step-7.png)

> Look for a line confirming the API server is up (`Application startup complete` / `Uvicorn running on http://0.0.0.0:8000`). Press `Ctrl+C` once you see it.

---

### Step 8 — Create a Service for vLLM

On the master node:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: vllm-service
  namespace: vllm-openwebui-demo
spec:
  selector:
    app: vllm
  ports:
  - port: 8000
    targetPort: 8000
EOF
```

> Use this command to check the creation of the service `kubectl get svc -n vllm-openwebui-demo`

![step-8](/02-Demos/03-Deploy-LLM-With-vLLM-OpenWebUI/Image/step-8.png)

---

### Step 9 — Test the vLLM API from inside the cluster

On the master node:

This checks vLLM's API on its own, before OpenWebUI enters the picture, so a later problem is easy to isolate to one side or the other. The below command runs a temporary pod using the `curl` image to send a request to vLLM's API and confirm it's up and serving — the pod deletes itself automatically once the command finishes (`--rm`). Testing from inside the cluster this way also confirms the Service's DNS name and port are reachable, the same way OpenWebUI will reach them.

```bash
kubectl run vllm-test -n vllm-openwebui-demo --rm -it --restart=Never --image=curlimages/curl -- curl -s http://vllm-service:8000/v1/models
```

> You should see a JSON response listing `qwen2.5-7b` as an available model — confirming vLLM is serving on the MIG slice and reachable over the cluster network.

![step9](/02-Demos/03-Deploy-LLM-With-vLLM-OpenWebUI/Image/step-9.png)

---

### Step 10 — Create the OpenWebUI data PVC

On the master node:

This creates a dedicated PVC to be mounted into the OpenWebUI pod in Step 11, so it can use it to persist its own database — chat history and settings — instead of losing them on every pod restart.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: openwebui-data
  namespace: vllm-openwebui-demo
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 3Gi
EOF
```

![step10](/02-Demos/03-Deploy-LLM-With-vLLM-OpenWebUI/Image/step-10.png)

---

### Step 11 — Deploy OpenWebUI, pointed at the vLLM Service

On the master node:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: openwebui
  namespace: vllm-openwebui-demo
  labels:
    app: openwebui
spec:
  replicas: 1
  selector:
    matchLabels:
      app: openwebui
  template:
    metadata:
      labels:
        app: openwebui
    spec:
      nodeSelector:
        workload-type: non-gpu
      containers:
      - name: openwebui
        image: ghcr.io/open-webui/open-webui:main
        env:
        - name: OPENAI_API_BASE_URLS
          value: "http://vllm-service.vllm-openwebui-demo.svc.cluster.local:8000/v1"
        - name: OPENAI_API_KEYS
          value: "none"
        - name: ENABLE_OLLAMA_API
          value: "false"
        - name: WEBUI_AUTH
          value: "false"
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: webui-data
          mountPath: /app/backend/data
      volumes:
      - name: webui-data
        persistentVolumeClaim:
          claimName: openwebui-data
EOF
```

![step11](/02-Demos/03-Deploy-LLM-With-vLLM-OpenWebUI/Image/step-11.png)

*Note: `nodeSelector: workload-type: non-gpu` uses the label set in the RKE2 install repo to keep this off the GPU node. `WEBUI_AUTH=false` skips OpenWebUI's login screen for this demo — not something to carry into production.*

---

### Step 12 — Watch the OpenWebUI pod come up

On the master node:

```bash
kubectl get pods -n vllm-openwebui-demo -w
```

> Press `Ctrl+C` once the pod shows `Running`.
> Also use this command to confirm the OpenWebUI pod is now running on the Non-GPU node and not the GPU node `kubectl get pods -n vllm-openwebui-demo -o wide`

![step12](/02-Demos/03-Deploy-LLM-With-vLLM-OpenWebUI/Image/step-12.png)

---

### Step 13 — Expose OpenWebUI with a NodePort Service

On the master node:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: openwebui-service
  namespace: vllm-openwebui-demo
spec:
  type: NodePort
  selector:
    app: openwebui
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30080
EOF
```

![step13](/02-Demos/03-Deploy-LLM-With-vLLM-OpenWebUI/Image/step-13.png)

---

### Step 14 — Open OpenWebUI in a browser

Get the public IP of the GPU node from your cloud provider console (the same IP you use to SSH into it), then browse to:

```
http://<worker-no-gpu-01-public-ip>:30080
```

![step14](/02-Demos/03-Deploy-LLM-With-vLLM-OpenWebUI/Image/step-14.png)

---

### Step 15 — Send a test prompt

In the model dropdown, select `qwen2.5-7b`, then send a prompt and confirm a response comes back.

This confirms the full path: browser → OpenWebUI (non-GPU worker, PVC-backed) → vLLM Service → vLLM pod on the `4g.71gb` MIG instance (PVC-backed) → H200 SXM.

![step15](/02-Demos/03-Deploy-LLM-With-vLLM-OpenWebUI/Image/step-15.png)

---

Enjoy
