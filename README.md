# AI-On-Kubernetes-Demo

*A series of self-contained demos that run AI workloads on a Kubernetes cluster, built to explain AI and Kubernetes concepts one object or feature at a time.*

---

## Description

This repo covers a set of hands-on demos for running AI on Kubernetes. Each demo lives in its own sub-repo and focuses on a single concept — for example, deploying the GPU Operator, configuring MIG, or serving an LLM with vLLM. Every demo is self-contained, with its own namespace and its own step-by-step README.

Outside the demos, there is one additional sub-repo that walks through installing RKE2 on the three cluster nodes. This is the base the demos build on top of.

---

## Intention of Use

This is for demo purposes only — not intended for production use. **Do not use this in a production environment.**

---

## Infrastructure

The demos run on a 3-node Kubernetes cluster (1 master, 1 non-GPU worker, 1 GPU worker) provisioned on **Nebius Cloud**. Provisioning the VMs is out of scope for this repo.

You can use **Nebius** or a different cloud. If you use a different cloud, you will need to adjust the code, commands, and files provided here to match it.

---

## Architecture 

![hl-arc](/Image/hl-arch.png)

---

## Node specifications

All nodes run Ubuntu. Each node has a private IP from the default VPC, and also has a public IP.

| Node | vCPU | RAM | GPU |
|---|---|---|---|
| Master Node | 2 | 8 GB | None |
| Worker Node 01 (non-GPU) | 4 | 16 GB | None |
| Worker Node 02 (GPU) | 16 | 200 GB | 1x H200 SXM |

---

## How to Use

1. Provision the infrastructure described above (or your own equivalent).
2. Install Kubernetes on top of it. See [01-Install-Kubernetes/README.md](/01-Install-Kubernetes/README.md)
  *this repo uses RKE2. If you want to install a different Kubernetes distribution, you can, but you will need to adjust the code, commands, and files accordingly.*
3. Once Kubernetes is installed, go to [02-Demos/README.md](/02-Demos/README.md) and work through the demos.

---

Enjoy
