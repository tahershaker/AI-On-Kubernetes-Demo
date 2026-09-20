# Demos

*A series of self-contained demos, each focused on one AI-on-Kubernetes concept or object at a time, built on top of the cluster installed earlier in this repo.*

---

## Description

Each folder below is a standalone demo. They build on each other loosely — later demos reuse a component from an earlier one (the MIG demo needs the GPU Operator, the vLLM demo needs the MIG split, the RAG demo need the LLM) — but each demo has its own README with its own prerequisites and steps.

This is for demo purposes only. **Do not use this in a production environment.**

---

## Prerequisites

- The 3-node cluster is installed and `kubectl` works from the master node — see [01-Install-Kubernetes/README.md](/01-Install-Kubernetes/README.md).

---

## Demos

| # | Demo | Focus |
|---|---|---|
| 01 | [Deploy The NVIDIA GPU Operator](/02-Demos/01-Install-GPU-Operator/) | Installing the GPU Operator via Helm to expose the H200 GPU to Kubernetes |
| 02 | [Configure MIG On The H200 GPU](/02-Demos/02-Configure-MIG/README.md) | Partitioning the H200 into isolated MIG instances with the GPU Operator's MIG manager |
| 03 | [Deploy an LLM With vLLM and Connect OpenWebUI](/02-Demos/03-Deploy-LLM-With-vLLM-OpenWebUI/README.md) | Serving an LLM with vLLM on a MIG slice, with OpenWebUI for browser-based chat access |
| 04 | [Build A Simple RAG (Vector DB + GPU Embedding Model)](/02-Demos/04-Deploy-Simple-RAG/README.md) | Retrieval-augmented generation with Chroma, a GPU-backed TEI embedding server, and vLLM |

---

Enjoy
