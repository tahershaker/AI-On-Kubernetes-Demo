# Build A Simple RAG (Vector DB + GPU Embedding Model)

*Builds a Retrieval-Augmented Generation (RAG) pipeline using explicit, individually deployed building blocks instead of a RAG framework — a vector database, a GPU-backed embedding server, and a small custom retrieval API — grounding vLLM's answers in a fictional company handbook, with questions asked live from OpenWebUI.*

---

## Description

This sub-repo provides a step-by-step guide to building a Retrieval-Augmented Generation (RAG) setup using explicit, individually deployed building blocks instead of a RAG framework such as LangChain or LlamaIndex.

The focus of this sub-repo is to stand up a vector database ([Chroma](https://docs.trychroma.com/)), a GPU-backed embedding server ([Text Embeddings Inference](https://github.com/huggingface/text-embeddings-inference), running on one of the two `1g.35gb` MIG instances created in [02-Configure-MIG](/02-Demos/02-Configure-MIG/README.md)), load a fictional 10-page company handbook for "AI-Demo-Lab" into that vector database, and expose a small custom API that performs retrieval before handing a grounded prompt to vLLM. That API is then connected into OpenWebUI (from [03-Deploy-vLLM-OpenWebUI](/02-Demos/03-Deploy-vLLM-OpenWebUI/README.md)) as a second model, so a retrieval-grounded answer can be compared directly against the raw model's answer, live from the browser.

The components used to build this demo are:
- **TEI** — a GPU-backed embedding server, pinned to one of the two `1g.35gb` MIG slices created in [02-Configure-MIG](/02-Demos/02-Configure-MIG/README.md), serving the `BAAI/bge-small-en-v1.5` embedding model.
- **Chroma** — a real vector database, deployed on the non-GPU worker, PVC-backed so vectors survive a pod restart.
- **RAG API** — a small custom service, written in plain Python, that wraps vLLM behind the same `/v1/chat/completions` shape. On every request it embeds the incoming question via TEI, retrieves the closest chunks from Chroma, and forwards a grounded prompt to vLLM — this is what OpenWebUI actually talks to for retrieval-augmented answers.
- **vLLM** — unchanged from [03-Deploy-LLM-With-vLLM-OpenWebUI](/02-Demos/03-Deploy-LLM-With-vLLM-OpenWebUI/README.md), still on the `4g.71gb` MIG slice, used only for the final answer generation.

*Note: The manual, step-by-step deployment in this demo is for demo purposes only. In production, you would most likely not run a fixed set of `kubectl apply` commands against a hardcoded local file — you would have a fully automated pipeline pulling source documents from object storage (e.g. S3-compatible storage), with a tool like Kubeflow orchestrating ingestion, chunking, and embedding on a schedule or on new-document triggers, rather than a person running it by hand.*

*The knowledge base used in this demo (a fictional company handbook) and its chunking strategy (a fixed-size sliding window, written in plain Python) are also for demo purposes only. In production, the source data, chunk size, overlap, and retrieval strategy all need to be worked out from the actual documents and workload, not copied from a demo.*

This guide builds a demo cluster and is not intended for a production environment. This is for demo purposes only. **Do not use this in a production environment.**

---

## Architecture

This demo wires together five pieces, each doing one job:

1. **Chunking and ingestion (Python)** — a plain Python script splits the fictional AI-Demo-Lab handbook into fixed-size, overlapping chunks (no text-splitting library), sends each chunk to the embedding server, and stores the resulting vectors in Chroma. This runs once, as a Kubernetes Job.
2. **TEI (embedding server)** — Hugging Face's Text Embeddings Inference server, running the `BAAI/bge-small-en-v1.5` model on one of the two `1g.35gb` MIG slices. Converts text into vectors — handbook chunks at ingestion time, questions at query time.
3. **Chroma (vector database)** — stores the chunk vectors and, given a question's vector, returns the closest matching chunks by similarity search.
4. **RAG API (Python)** — a small custom service that mimics vLLM's own API shape (`/v1/chat/completions`). On every request it embeds the incoming question via TEI, retrieves the closest chunks from Chroma, builds a prompt containing only those chunks, and forwards it to vLLM.
5. **vLLM and OpenWebUI** — unchanged from demo 03. vLLM generates the final answer from the augmented prompt; OpenWebUI is the browser interface, configured with two selectable models — the raw `qwen2.5-7b` (no retrieval) and `ai-demo-lab-rag` (retrieval-grounded) — so the two can be compared side by side.

Data flow for a single question, end to end:

|---|
|Browser (OpenWebUI) |
|  - RAG API (embeds the question via TEI) |
|  - Chroma (returns the closest handbook chunks) |
|  - RAG API (builds a grounded prompt from those chunks) |
|  - vLLM (generates the answer) |
|  - back to OpenWebUI |

---

## Prerequisites

- Logged in to the master node, with `kubectl` working: `kubectl get nodes` - All 3 nodes should show `Ready`.
- The MIG split from [02-Configure-MIG](/02-Demos/02-Configure-MIG/README.md) is applied, and `kube-ai-demo-worker-gpu-02` advertises `nvidia.com/mig-1g.35gb: 2` (this demo uses one of the two slices; the other stays free).
- The vLLM deployment from [03-Deploy-vLLM-OpenWebUI](/02-Demos/03-Deploy-vLLM-OpenWebUI/README.md) is running and reachable at `vllm-service.vllm-openwebui-demo.svc.cluster.local:8000`, serving `qwen2.5-7b`.
- `kube-ai-demo-worker-no-gpu-01` is labeled `workload-type=non-gpu` (see [01-Install-Kubernetes, Step 16](/01-Install-Kubernetes/README.md)).
- Outbound internet access from `kube-ai-demo-worker-gpu-02` — TEI downloads the embedding model (~130 MB) from Hugging Face on first start.
- Outbound internet access from `kube-ai-demo-worker-no-gpu-01` — the Job installs `chromadb` and `requests` with `pip` on first run.

---

## Before We Start — Confirm the base model has no knowledge of AI-Demo-Lab

Before building anything, it's worth showing the starting point: the vLLM model deployed in [03-Deploy-vLLM-OpenWebUI](/02-Demos/03-Deploy-vLLM-OpenWebUI/README.md) has no knowledge of this fictional company, since it was never trained or given any information about it. This is the "before" this demo compares against.

Open OpenWebUI (already deployed in demo 03) at: `http://<worker-no-gpu-01-public-ip>:30080`

Make sure `qwen2.5-7b` is selected in the model dropdown, then ask:

`How many days of paid annual leave do employees get?`


The model has nothing to answer this question with — it doesn't know which company you mean, and even if you name AI-Demo-Lab explicitly, it has no data about it. Expect either a request for clarification or a generic, made-up answer, not the real figure (25 days). Keep this in mind — the same question gets asked again later, once retrieval is wired in, and the answer should then be grounded and correct.

![step0](/02-Demos/04-Deploy-Simple-RAG/Image/step-0.png)

---

## How to Use

### Step 1 — Create the namespace

On the master node:

```bash
kubectl create namespace simple-rag-demo
```

---

### Step 2 — Create the Chroma PVC

On the master node:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: chroma-data
  namespace: simple-rag-demo
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 2Gi
EOF
```

![step2](/02-Demos/04-Deploy-Simple-RAG/Image/step-2.png)

---

### Step 3 — Deploy Chroma on the non-GPU worker

On the master node:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chroma
  namespace: simple-rag-demo
  labels:
    app: chroma
spec:
  replicas: 1
  selector:
    matchLabels:
      app: chroma
  template:
    metadata:
      labels:
        app: chroma
    spec:
      nodeSelector:
        workload-type: non-gpu
      containers:
      - name: chroma
        image: chromadb/chroma
        ports:
        - containerPort: 8000
        volumeMounts:
        - name: chroma-data
          mountPath: /data
      volumes:
      - name: chroma-data
        persistentVolumeClaim:
          claimName: chroma-data
EOF
```

*Note: `chromadb/chroma` stores its data at `/data` inside the container — mapped here to the PVC so the vector store survives a pod restart.*

![step3](/02-Demos/04-Deploy-Simple-RAG/Image/step-3.png)

---

### Step 4 — Watch the Chroma pod come up

On the master node:

```bash
kubectl get pods -n simple-rag-demo -l app=chroma -w
```

Press `Ctrl+C` once the pod shows `Running`.

Also use this command to confirm the Chroma pod is now running on the Non-GPU node and not the GPU node ```bash kubectl get pods -n simple-rag-demo -l app=chroma -o wide```

![step4](/02-Demos/04-Deploy-Simple-RAG/Image/step-4.png)

---

### Step 5 — Create a Service for Chroma

On the master node:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: chroma-service
  namespace: simple-rag-demo
spec:
  selector:
    app: chroma
  ports:
  - port: 8000
    targetPort: 8000
EOF
```

![step5](/02-Demos/04-Deploy-Simple-RAG/Image/step-5.png)

---

### Step 6 — Deploy TEI (embedding server) on the small MIG instance

On the master node:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tei
  namespace: simple-rag-demo
  labels:
    app: tei
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tei
  template:
    metadata:
      labels:
        app: tei
    spec:
      nodeSelector:
        kubernetes.io/hostname: kube-ai-demo-worker-gpu-02
      containers:
      - name: tei
        image: ghcr.io/huggingface/text-embeddings-inference:hopper-1.9
        args:
          - "--model-id=BAAI/bge-small-en-v1.5"
          - "--port=80"
        ports:
        - containerPort: 80
        resources:
          limits:
            nvidia.com/mig-1g.35gb: 1
EOF
```

*Note: `hopper-1.9` matches the H200's Hopper compute architecture — check the [current tag list](https://github.com/huggingface/text-embeddings-inference/pkgs/container/text-embeddings-inference) before running, since Hugging Face ships new tags regularly. `nvidia.com/mig-1g.35gb: 1` requests one of the two small MIG slices specifically, leaving the other free for other workloads.*

![step6](/02-Demos/04-Deploy-Simple-RAG/Image/step-6.png)

---

### Step 7 — Watch the TEI pod come up

On the master node:

```bash
kubectl get pods -n simple-rag-demo -l app=tei -w
```

First start downloads the embedding model. Press `Ctrl+C` once the pod shows `Running`.

![step7](/02-Demos/04-Deploy-Simple-RAG/Image/step-7.png)

---

### Step 8 — Create a Service for TEI

On the master node:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: tei-service
  namespace: simple-rag-demo
spec:
  selector:
    app: tei
  ports:
  - port: 80
    targetPort: 80
EOF
```

![step8](/02-Demos/04-Simple-RAG/Image/step-8.png)

---

### Step 9 — Test the embedding endpoint

On the master node:

```bash
kubectl run tei-test -n simple-rag-demo --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s -X POST http://tei-service/embed -H "Content-Type: application/json" -d '{"inputs":"hello world"}'
```

You should see a JSON array of floating-point numbers — confirming the embedding model is serving on the GPU and reachable over the cluster network.

![step9](/02-Demos/04-Deploy-Simple-RAG/Image/step-9.png)

---

### Step 10 — Create the AI-Demo-Lab company handbook file

On the master node. This is the knowledge base the demo retrieves from — a fictional company handbook covering company background and internal policies. 

Copy the content of the `ai-demo-lab-handbook.txt` file located in [The Docs Section](/02-Demos/04-Deploy-Simple-RAG/Docs/ai-demo-lab-handbook.txt) and past it in the below command to create this file on the master node.

```bash
cat <<'EOF' > ai-demo-lab-handbook.txt
<Past-The-Content-Of-The-Text-File-Here>
EOF
```

![step10](/02-Demos/04-Deploy-Simple-RAG/Image/step-10.png)

---

### Step 11 — Load the handbook into a ConfigMap

On the master node:

```bash
kubectl create configmap rag-docs -n simple-rag-demo --from-file=handbook.txt=ai-demo-lab-handbook.txt
```

![step11](/02-Demos/04-Deploy-Simple-RAG/Image/step-11.png)

---

### Step 12 — Create the ingestion script and load it into a ConfigMap

On the master node. This script only chunks the handbook, embeds it, and stores it in Chroma — it does not handle a question or call vLLM, since that part now lives in the RAG API created in Step 16:

```bash
cat <<'EOF' > rag-ingest.py
import requests
import chromadb

DOC_FILE = "/data/docs/handbook.txt"
TEI_URL = "http://tei-service.simple-rag-demo.svc.cluster.local/embed"
CHROMA_HOST = "chroma-service.simple-rag-demo.svc.cluster.local"
CHROMA_PORT = 8000
CHUNK_SIZE = 800
CHUNK_OVERLAP = 100

def chunk_text(text, size=CHUNK_SIZE, overlap=CHUNK_OVERLAP):
    chunks = []
    start = 0
    while start < len(text):
        end = start + size
        chunk = text[start:end].strip()
        if chunk:
            chunks.append(chunk)
        start = end - overlap
    return chunks

def embed(texts, batch_size=32):
    embeddings = []
    for i in range(0, len(texts), batch_size):
        batch = texts[i:i + batch_size]
        resp = requests.post(TEI_URL, json={"inputs": batch})
        resp.raise_for_status()
        embeddings.extend(resp.json())
    return embeddings

print("Connecting to Chroma...")
client = chromadb.HttpClient(host=CHROMA_HOST, port=CHROMA_PORT)
collection = client.get_or_create_collection("ai-demo-lab-handbook")

with open(DOC_FILE) as f:
    document = f.read()

chunks = chunk_text(document)
print(f"Split handbook into {len(chunks)} chunks")

print("Embedding chunks on the GPU embedding server...")
chunk_embeddings = embed(chunks)

collection.upsert(
    ids=[f"chunk-{i}" for i in range(len(chunks))],
    embeddings=chunk_embeddings,
    documents=chunks,
)
print(f"Stored {len(chunks)} chunk embeddings in Chroma")
print("Ingestion complete.")
EOF
kubectl create configmap rag-ingest-script -n simple-rag-demo --from-file=rag-ingest.py=rag-ingest.py
kubectl get configmap rag-ingest-script -n simple-rag-demo
```

*Note: `chunk_text` is a plain sliding-window splitter — fixed-size character chunks with a small overlap so a fact split across a chunk boundary usually still appears whole in at least one chunk. No text-splitting library, no framework.*

![step12](/02-Demos/04-Deploy-Simple-RAG/Image/step-12.png)

---

### Step 13 — Deploy the ingestion Job

On the master node:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: rag-ingest
  namespace: simple-rag-demo
spec:
  backoffLimit: 0
  template:
    spec:
      restartPolicy: Never
      nodeSelector:
        workload-type: non-gpu
      containers:
      - name: ingest
        image: python:3.11-slim
        command: ["sh", "-c", "pip install --quiet chromadb requests && python /data/script/rag-ingest.py"]
        volumeMounts:
        - name: docs
          mountPath: /data/docs
        - name: script
          mountPath: /data/script
      volumes:
      - name: docs
        configMap:
          name: rag-docs
      - name: script
        configMap:
          name: rag-ingest-script
EOF
```

![step13](/02-Demos/04-Deploy-Simple-RAG/Image/step-13.png)

---

### Step 14 — Watch the ingestion Job run to completion

On the master node:

```bash
kubectl get pods -n simple-rag-demo -l job-name=rag-ingest -w
```

Wait until the pod shows `Completed`, then press `Ctrl+C`.

![step14](/02-Demos/04-Deploy-Simple-RAG/Image/step-14.png)

---

### Step 15 — Check the ingestion output

On the master node:

```bash
kubectl logs -n simple-rag-demo -l job-name=rag-ingest
```

You should see the chunk count and a confirmation that the embeddings were stored in Chroma.

![step15](/02-Demos/04-Deploy-Simple-RAG/Image/step-15.png)

---

### Step 16 — Create the RAG API script and load it into a ConfigMap

On the master node. This is the piece that makes OpenWebUI retrieval-aware: it exposes the same `/v1/chat/completions` shape vLLM uses, but embeds the question, retrieves context from Chroma, and forwards an augmented prompt to vLLM before returning the answer:

```bash
cat <<'EOF' > rag-api.py
import json
import time
import requests
import chromadb
from flask import Flask, request, jsonify, Response

TEI_URL = "http://tei-service.simple-rag-demo.svc.cluster.local/embed"
CHROMA_HOST = "chroma-service.simple-rag-demo.svc.cluster.local"
CHROMA_PORT = 8000
VLLM_URL = "http://vllm-service.vllm-openwebui-demo.svc.cluster.local:8000/v1/chat/completions"
VLLM_MODEL = "qwen2.5-7b"
RAG_MODEL_NAME = "ai-demo-lab-rag"
TOP_K = 3

app = Flask(__name__)
chroma_client = chromadb.HttpClient(host=CHROMA_HOST, port=CHROMA_PORT)
collection = chroma_client.get_or_create_collection("ai-demo-lab-handbook")

def embed(texts):
    resp = requests.post(TEI_URL, json={"inputs": texts})
    resp.raise_for_status()
    return resp.json()

@app.route("/v1/models", methods=["GET"])
def models():
    return jsonify({
        "object": "list",
        "data": [{"id": RAG_MODEL_NAME, "object": "model", "owned_by": "simple-rag-demo"}],
    })

@app.route("/v1/chat/completions", methods=["POST"])
def chat_completions():
    body = request.get_json()
    messages = body.get("messages", [])
    question = next((m["content"] for m in reversed(messages) if m["role"] == "user"), "")
    stream = body.get("stream", False)

    question_embedding = embed([question])[0]
    results = collection.query(query_embeddings=[question_embedding], n_results=TOP_K)
    retrieved_chunks = results["documents"][0]
    context = "\n\n".join(retrieved_chunks)

    prompt = f"""Answer the question using only the context below. If the answer is not in the context, say you don't know.

Context:
{context}

Question: {question}
"""

    vllm_response = requests.post(
        VLLM_URL,
        json={
            "model": VLLM_MODEL,
            "messages": [{"role": "user", "content": prompt}],
            "max_tokens": 300,
            "temperature": 0.1,
        },
    )
    vllm_response.raise_for_status()
    answer = vllm_response.json()["choices"][0]["message"]["content"]

    if not stream:
        return jsonify({
            "id": "ragchat-1",
            "object": "chat.completion",
            "created": int(time.time()),
            "model": RAG_MODEL_NAME,
            "choices": [{
                "index": 0,
                "message": {"role": "assistant", "content": answer},
                "finish_reason": "stop",
            }],
        })

    def generate():
        chunk = {
            "id": "ragchat-1",
            "object": "chat.completion.chunk",
            "created": int(time.time()),
            "model": RAG_MODEL_NAME,
            "choices": [{"index": 0, "delta": {"role": "assistant", "content": answer}, "finish_reason": None}],
        }
        yield f"data: {json.dumps(chunk)}\n\n"
        done_chunk = {
            "id": "ragchat-1",
            "object": "chat.completion.chunk",
            "created": int(time.time()),
            "model": RAG_MODEL_NAME,
            "choices": [{"index": 0, "delta": {}, "finish_reason": "stop"}],
        }
        yield f"data: {json.dumps(done_chunk)}\n\n"
        yield "data: [DONE]\n\n"

    return Response(generate(), mimetype="text/event-stream")

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8001)
EOF
kubectl create configmap rag-api-script -n simple-rag-demo --from-file=rag-api.py=rag-api.py
```

*Note: OpenWebUI normally streams a model's response token by token (`stream: true`). This script doesn't do real token-by-token streaming — it waits for vLLM's full answer, then sends it back as a single Server-Sent-Events chunk in the format OpenWebUI expects, followed by `[DONE]`. This keeps the script simple while still working correctly inside OpenWebUI; the only visible difference is the answer appears all at once instead of typing itself out.*

![step16](/02-Demos/04-Deploy-Simple-RAG/Image/step-16.png)

---

### Step 17 — Deploy the RAG API

On the master node:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rag-api
  namespace: simple-rag-demo
  labels:
    app: rag-api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rag-api
  template:
    metadata:
      labels:
        app: rag-api
    spec:
      nodeSelector:
        workload-type: non-gpu
      containers:
      - name: rag-api
        image: python:3.11-slim
        command: ["sh", "-c", "pip install --quiet flask chromadb requests && python /app/script/rag-api.py"]
        ports:
        - containerPort: 8001
        volumeMounts:
        - name: script
          mountPath: /app/script
      volumes:
      - name: script
        configMap:
          name: rag-api-script
EOF
```

*Note: this is a Deployment, not a Job — unlike the ingestion step, the RAG API needs to stay running to answer questions from OpenWebUI on demand.*

---

### Step 18 — Watch the RAG API pod come up

On the master node:

```bash
kubectl get pods -n simple-rag-demo -l app=rag-api -w
```

Press `Ctrl+C` once the pod shows `Running`.

![step18](/02-Demos/04-Deploy-Simple-RAG/Image/step-18.png)

---

### Step 19 — Create a Service for the RAG API

On the master node:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: rag-api-service
  namespace: simple-rag-demo
spec:
  selector:
    app: rag-api
  ports:
  - port: 8001
    targetPort: 8001
EOF
```

![step19](/02-Demos/04-Deploy-Simple-RAG/Image/step-19.png)

---

### Step 20 — Test the RAG API directly

On the master node:

```bash
kubectl run rag-api-test -n simple-rag-demo --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s -X POST http://rag-api-service:8001/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"ai-demo-lab-rag","messages":[{"role":"user","content":"How many days of paid annual leave do employees get?"}]}'
```

You should see a JSON response whose answer states 25 days — confirming retrieval and generation work end to end before wiring this into OpenWebUI.

![step20](/02-Demos/04-Deploy-Simple-RAG/Image/step-20.png)

---

### Step 21 — Add the RAG API connection through OpenWebUI's Admin UI

OpenWebUI persists its configuration — including the list of connected OpenAI-compatible backends — into `config.json` inside its data directory, which lives on the `openwebui-data` PVC created in demo 03. Because that file was already written the first time OpenWebUI booted, updating `OPENAI_API_BASE_URLS` as an environment variable on the Deployment has no effect: the persisted config always takes priority over the env var on every later restart. The supported way to add a second backend after that point is through the UI:

1. In OpenWebUI, click your profile icon (bottom-left) → **Admin Panel**.
2. Go to **Settings → Connections**.
3. Under the OpenAI API section, click **+** to add another connection.
4. Set URL to `http://rag-api-service.simple-rag-demo.svc.cluster.local:8001/v1` and Key to any non-empty value (e.g. `none`) — the RAG API doesn't check it.
5. Save.

---

### Step 23 — Open OpenWebUI and select the RAG model

Browse to the same address used in demo 03:

```
http://<worker-no-gpu-01-public-ip>:30080
```

Open the model dropdown — you should now see two models: `qwen2.5-7b` (direct, no retrieval) and `ai-demo-lab-rag` (retrieval-grounded). Select `ai-demo-lab-rag`.

![step23](/02-Demos/04-Deploy-Simple-RAG/Image/step-23.png)

---

### Step 24 — Send a test prompt and confirm a grounded answer

Type a question into the chat, for example:

```
How many days of paid annual leave do employees get?
```

The answer should state 25 days, sourced from Section 6 of the handbook. Try a second question from a different section, for example:


```
What is the target time to first response for a Critical incident affecting a Tier 1 customer?
```

This should answer 4 hours, sourced from Section 10 — confirming retrieval is pulling different chunks depending on the question, not just repeating the same context every time.

![step24-2](/02-Demos/04-Simple-RAG/Image/step-24.png)

---

### Step 25 — Clean up the ingestion Job

On the master node. The ingestion Job has already done its job (loading the handbook into Chroma) and does not need to stay around:

```bash
kubectl delete job rag-ingest -n simple-rag-demo
```

*(Chroma, TEI, the RAG API, vLLM, and OpenWebUI are all left running as standing services for this demo.)*

---

Enjoy
