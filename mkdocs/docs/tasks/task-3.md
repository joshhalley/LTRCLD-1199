# Task 3: Kubernetes App Hosting using Virtual Kubelet

[⬅ Back to Main Menu](../index.md)

---

## Objective

In this task, you will deploy a **containerized application on a Cisco Catalyst 8000V (C8Kv)** router using **Kubernetes** and **Cisco Virtual Kubelet**.

To save time during the lab, the Kubernetes cluster and supporting infrastructure are **already deployed and running**. You will focus on:

- Validating Kubernetes readiness
- Preparing the Cisco C8Kv router for app-hosting
- Deploying Cisco Virtual Kubelet
- Running a containerized application (`hello-app`) on the router
- Verifying application reachability

---

## Table of Contents

1. [Kubernetes Cluster Validation](#1-kubernetes-cluster-validation)
2. [Container Registry Verification](#2-container-registry-verification)
3. [Router Preparation (cat8kv-task-3)](#3-router-preparation-cat8kv-task-3)
4. [Build and Upload hello-app Image](#4-build-and-upload-hello-app-image)
5. [Virtual Kubelet Deployment](#5-virtual-kubelet-deployment)
6. [Deploy hello-app on C8Kv](#6-deploy-hello-app-on-c8kv)
7. [Verification and Validation](#7-verification-and-validation)

---

## 1. Kubernetes Cluster Validation

Verify that Kubernetes is up and running:

```bash
kubectl get nodes
```

Expected output:

```text
NAME         STATUS   ROLES           AGE   VERSION
ubuntu-lab   Ready    control-plane   38h   v1.34.3+k3s1
```

---

## 2. Container Registry Verification

Confirm the required images are available in the registry:

```bash
curl -s https://containers.dmz.cisco.com:5000/v2/_catalog
```

Expected output:

```json
{
  "repositories": [
    "cisco-virtual-kubelet",
    "hello-app",
    "mrtg",
    "swiss-knife-alpine",
    "wireshark"
  ]
}
```

Verify image tags:

```bash
curl -s https://containers.dmz.cisco.com:5000/v2/cisco-virtual-kubelet/tags/list
curl -s https://containers.dmz.cisco.com:5000/v2/hello-app/tags/list
```

Expected:

```json
{"name":"cisco-virtual-kubelet","tags":["1.0.0"]}
{"name":"hello-app","tags":["latest"]}
```

---

## 3. Router Preparation (cat8kv-task-3)

Target router:

```text
cat8kv-task-3
IP Address: 198.18.1.13
```

### 3.1 Verify IOX and RESTCONF

```bash
ssh admin@198.18.1.13
```

```text
cat8Kv-task-3# show run | i iox|restconf
iox
restconf
```

---

### 3.2 Disable App Hosting Signature Verification

```text
cat8Kv-task-3# app-hosting verification disable
```

```text
cat8Kv-task-3(config)# no app-hosting signed-verification
```

---

### 3.3 Configure Virtual Port Group

```text
cat8Kv-task-3(config)# interface virtualportgroup0
cat8Kv-task-3(config-if)# ip address 198.18.102.1 255.255.255.0
```

---

### 3.4 DHCP Relay Configuration

```text
cat8Kv-task-3(config)# interface virtualportgroup0
cat8Kv-task-3(config-if)# ip helper-address 198.18.1.102
```

---

### 3.5 Enable SCP

```text
cat8Kv-task-3(config)# ip scp server enable
```

---

### 3.6 Verify hello-app Is Not Present

```text
cat8Kv-task-3# dir flash: | i hello.*tar
```

(No output expected)

Exit the router and return to **ubuntu-lab**.

---

## 4. Build and Upload hello-app Image

### 4.1 Pull the Docker Image

```bash
sudo docker pull containers.dmz.cisco.com:5000/hello-app:latest
```

---

### 4.2 Create IOS XE Compatible TAR

```bash
sudo docker save containers.dmz.cisco.com:5000/hello-app:latest -o hello-app.iosxe.tar
sudo chmod 666 hello-app.iosxe.tar
```

---

### 4.3 Upload Image to Router

```bash
scp hello-app.iosxe.tar admin@198.18.1.13:/flash:/hello-app.iosxe.tar
```

---

## 5. Virtual Kubelet Deployment

### 5.1 Create Cluster Role Binding

```bash
kubectl create clusterrolebinding vk-admin-binding \
  --clusterrole=cluster-admin \
  --serviceaccount=default:default
```

---

### 5.2 Remote Kubeconfig ConfigMap

```bash
kubectl apply -f 01_vk_configmap.yaml
```

---

### 5.3 Virtual Kubelet Configuration

```bash
kubectl apply -f 02_vk_deployment_config.yaml
```

---

### 5.4 Deploy Virtual Kubelet

```bash
kubectl apply -f 03_vk_deployment.yaml
```

Verify deployment:

```bash
kubectl get pods -o wide
```

---

## 6. Deploy hello-app on C8Kv

Apply the pod manifest:

```bash
kubectl apply -f 04_vk_pod_hello-app.yaml
```

Monitor provisioning:

```bash
kubectl get pods -o wide -w
```

Expected state:

```text
iox-xe-hello-app-pod   1/1   Running   198.18.102.3   cat8kv-node
```

---

## 7. Verification and Validation

Access the application using the assigned IP:

```bash
curl http://198.18.102.3:8080
```

Expected output:

```text
Hello, world!
Version: 1.0.0
Hostname: cvkxxxxxxxxxxxxxxxx
```

---

## Summary

✔ Kubernetes workload scheduled on a **Cisco C8Kv router**
✔ Application deployed using **Virtual Kubelet**
✔ IOS XE App Hosting integrated with Kubernetes
✔ End-to-end application reachability verified

---

[⬅ Back to Main Menu](../index.md)

```