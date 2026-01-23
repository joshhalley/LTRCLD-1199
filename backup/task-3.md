# Task 3: Kubernetes-Based Container Orchestration

[⬅️ Back to Main Menu](README.md)

## Table of Contents

- [Step 1: Retrieve Image from Container Registry](#step-1-retrieve-image-from-container-registry)
- [Step 2: Create TAR Image from Docker](#step-2-create-tar-image-from-docker)
- [Step 3: Activate SCP on Router](#step-3-activate-scp-on-router)
- [Step 4: SCP File to Router](#step-4-scp-file-to-router)
- [Step 5: Verify MD5 Hash](#step-5-verify-md5-hash)
- [Step 6: Install KIND](#step-6-install-kind)
- [Step 7: Install Virtual Kubelet Provider](#step-7-install-virtual-kubelet-provider)
- [Step 8: Create Manifest File](#step-8-create-manifest-file)
- [Step 9: Kubectl Apply](#step-9-kubectl-apply)
- [Step 10: Check Kubernetes Node and Pod Health](#step-10-check-kubernetes-node-and-pod-health)

---

This task introduces Kubernetes-based orchestration to manage containerized network tools running on **Cisco IOS XE Router** devices.

In this task, you will:

* Retrieve an image from a container registry
* Create a TAR package from Docker
* Transfer the file to the router using SCP
* Install **KIND (Kubernetes in Docker)**
* Install a **virtual kubelet provider** for C8Kv
* Create and apply a Kubernetes manifest
* Validate Kubernetes node and pod health
* Test the deployed monitoring tools

> ⚠️ *Note: Full Kubernetes integration tuning will be enhanced later by another team member.*

---

## Step 1: Retrieve Image from Container Registry

• Pull the container image used for Kubernetes deployment

```bash
docker pull myregistry.example.com/tools/k8s-net-tools:latest
```

Verify image:

```bash
docker images
```

---

## Step 2: Create TAR Image from Docker

• Convert the image to a TAR format for IOS-XE app hosting

```bash
docker save myregistry.example.com/tools/k8s-net-tools:latest -o k8s-net-tools.tar
```

Verify:

```bash
ls -lh k8s-net-tools.tar
```

---

## Step 3: Activate SCP on Router

• Enable SCP if not already active

```bash
conf t
ip scp server enable
end
write memory
```

Verify:

```bash
show running-config | include scp
```

---

## Step 4: SCP File to Router

• Copy TAR file to the router

```bash
scp k8s-net-tools.tar admin@10.10.10.1:bootflash:
```

Verify:

```bash
dir bootflash: | include k8s-net-tools
```

---

## Step 5: Verify MD5 Hash

• Validate file integrity

```bash
verify /md5 bootflash:k8s-net-tools.tar
```

Local comparison:

```bash
md5sum k8s-net-tools.tar
```

---

## Step 6: Install KIND

• Install Kubernetes in Docker (KIND) on your Linux / jump host

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

Create a cluster:

```bash
kind create cluster --name c8kv-lab
```

Verify:

```bash
kubectl cluster-info --context kind-c8kv-lab
```

---

## Step 7: Install Virtual Kubelet Provider

• Install the Virtual Kubelet to represent C8Kv as a Kubernetes node

```bash
kubectl apply -f https://raw.githubusercontent.com/virtual-kubelet/virtual-kubelet/main/deploy/virtual-kubelet.yaml
```

Verify node registration:

```bash
kubectl get nodes
```

Expected output:

```bash
c8kv-virtual-node   Ready
```

*(Name may vary depending on your configuration)*

---

## Step 8: Create Manifest File

• Create manifest file for your containerized tool

```bash
nano c8kv-tool.yaml
```

Example:

```bash
apiVersion: v1
kind: Pod
metadata:
  name: c8kv-net-tools
spec:
  containers:
  - name: net-tools
    image: myregistry.example.com/tools/k8s-net-tools:latest
    ports:
    - containerPort: 8080
```

Save and exit.

---

## Step 9: Kubectl Apply

• Deploy container via Kubernetes

```bash
kubectl apply -f c8kv-tool.yaml
```

Verify pod:

```bash
kubectl get pods -o wide
```

---

## Step 10: Check Kubernetes Node and Pod Health

• Check Kubernetes cluster status

```bash
kubectl get nodes
```

• Check pod state

```bash
kubectl get pods
```

• On the C8Kv router, verify application status

```bash
show app-hosting list
```

Expected state:

```bash
RUNNING
```

---

* [Main Menu](/README.md/#table-of-content)

---

[⬅️ Return to Main Menu](README.md)
