# Task 3: Kubernetes App Hosting using Virtual Kubelet

[⬅ Back to Main Menu](../index.md)

---

## Objective

In this task, you will deploy a **containerized application on a Cisco Catalyst 8000V (C8Kv)** router using **Kubernetes** and **Cisco Virtual Kubelet**.

The Kubernetes cluster is **already running**. You will:

- Validate Kubernetes readiness
- Prepare the router for IOx app-hosting
- Deploy Cisco Virtual Kubelet
- Run `hello-app` on the router
- Verify application access

---

## Table of Contents

1. [Kubernetes Cluster Validation](#1-kubernetes-cluster-validation)  
2. [Container Registry Verification](#2-container-registry-verification)  
3. [Router Preparation (cat8kv-task-3)](#3-router-preparation-cat8kv-task-3)  
4. [Build and Upload hello-app Image](#4-build-and-upload-hello-app-image)  
5. [Create Kubernetes Manifests](#5-create-kubernetes-manifests)  
6. [Deploy Virtual Kubelet and hello-app](#6-deploy-virtual-kubelet-and-hello-app)  
7. [Verification](#7-verification)  

---

## 1. Kubernetes Cluster Validation

* Login to the Lab Ubuntu (ssh 198.18.1.100)

Verify the Kubernetes node is healthy:

```bash
kubectl get nodes
```

Expected:

```text
NAME         STATUS   ROLES           AGE   VERSION
ubuntu-lab   Ready    control-plane   38h   v1.34.3+k3s1
```

---

## 2. Container Registry Verification

Confirm required repositories exist:

```bash
curl -s https://containers.dmz.cisco.com:5000/v2/_catalog
```

Expected:

```json
{"repositories":["cisco-virtual-kubelet","hello-app","mrtg","swiss-knife-alpine","wireshark"]}
```

Confirm tags:

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

Router:

* **Hostname:** cat8kv-task-3
* **IP:** 198.18.1.13

Login:

```bash
ssh admin@198.18.1.13
```

---

### 3.1 Verify IOX and RESTCONF

```text
show run | i iox|restconf
```
```text
iox
restconf
```

---

### 3.2 Enable and Disable App Hosting Signature Verification

```bash
conf t
app-hosting signed-verification
end
```
*Jan 13 12:00:22.429: %IM-6-VERIFICATION_MSG: R0/0: ioxman: app-hosting: App signature verification enabled successfully
```bash
conf t
no app-hosting signed-verification
end
```
*Jan 13 12:00:35.447: %IM-6-VERIFICATION_MSG: R0/0: ioxman: app-hosting: App signature verification disabled successfully

---

### 3.3 VirtualPortGroup Configuration

> If `show run int virtualportgroup0` fails, configure the interface directly.

```text
conf t
interface virtualportgroup0
ip address 198.18.102.1 255.255.255.0
end
```

---

### 3.4 DHCP Relay Configuration

```text
conf t
interface virtualportgroup0
ip helper-address 198.18.1.102
end
```

---

### 3.5 Enable SCP

Check existing:

```text
show run | i scp
```

Enable:

```text
conf t
ip scp server enable
end
```

---

### 3.6 Verify hello-app Is NOT Present Yet

```text
dir flash: | i hello.*tar
```

Expected: **No output**.

Exit router and return to `ubuntu-lab`.

---

## 4. Build and Upload hello-app Image

### 4.1 Pull hello-app from Registry

```bash
sudo docker pull containers.dmz.cisco.com:5000/hello-app:latest
```

---

### 4.2 Save as IOS XE TAR

```bash
sudo docker save containers.dmz.cisco.com:5000/hello-app:latest -o hello-app.iosxe.tar
sudo chmod 666 hello-app.iosxe.tar
```

---

### 4.3 Upload to Router Flash

```bash
scp hello-app.iosxe.tar admin@198.18.1.13:/flash:/hello-app.iosxe.tar
```

---

### 4.4 Create AppID to Verify Image

```text
cat8Kv-task-3# conf t
cat8Kv-task-3(config)# app-hosting appid hello_app_verify
cat8Kv-task-3(config)# end
```

---

### 4.5 Deploy hello-app image 

```text
cat8Kv-task-3# cat8Kv-task-3#app-hosting install appid hello_app_verify package flash:hello-app.iosxe.tar
```

---

## 5. Create Kubernetes Manifests

> Create **four YAML files** on `ubuntu-lab` exactly as shown below.

---

### 5.1 RBAC (One-Time) — Cluster Role Binding

```bash
kubectl create clusterrolebinding vk-admin-binding \
  --clusterrole=cluster-admin \
  --serviceaccount=default:default
```

---

### 5.2 File: `01_vk_configmap.yaml` (Remote Kubeconfig)

```bash
cat > 01_vk_configmap.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: remote-kubeconfig
  namespace: default
data:
  config: |
    apiVersion: v1
    kind: Config
    clusters:
    - name: local-cluster
      cluster:
        certificate-authority: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        server: https://kubernetes.default.svc:443
    contexts:
    - name: default-context
      context:
        cluster: local-cluster
        user: pod-service-account
        namespace: default
    current-context: default-context
    users:
    - name: pod-service-account
      user:
        tokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
EOF
```

Apply:

```bash
kubectl apply -f 01_vk_configmap.yaml
```

Expected:

```text
configmap/remote-kubeconfig created
```

---

### 5.3 File: `02_vk_deployment_config.yaml` (VK Device + Node Mapping)

```bash
cat > 02_vk_deployment_config.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: vk-config
  namespace: default
data:
  config.yaml: |
    device:
      name: cat8kv-router
      driver: XE
      address: "198.18.8.13"
      port: 443
      username: admin
      password: C1sco12345
      tls:
        enabled: true
        insecureSkipVerify: true
      networking:
        dhcpEnabled: true
        virtualPortGroup: "0"
        defaultVRF: ""

    kubelet:
      node_name: "cat8kv-node"
      namespace: ""
      update_interval: "30s"
      os: "Linux"
      node_internal_ip: "198.18.8.13"
EOF
```

Apply:

```bash
kubectl apply -f 02_vk_deployment_config.yaml
```

Expected:

```text
configmap/vk-config created
```

---

### 5.4 File: `03_vk_deployment.yaml` (Virtual Kubelet Deployment)

```bash
cat > 03_vk_deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cisco-virtual-kubelet
  labels:
    app: cisco-virtual-kubelet
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cisco-virtual-kubelet
  template:
    metadata:
      labels:
        app: cisco-virtual-kubelet
    spec:
      containers:
      - name: virtual-kubelet
        image: containers.dmz.cisco.com:5000/cisco-virtual-kubelet:1.0.0
        env:
        - name: KUBECONFIG
          value: "/etc/kubernetes/kubeconfig.yaml"
        volumeMounts:
        - name: kubeconfig-storage
          mountPath: "/etc/kubernetes"
          readOnly: true
        - name: vk-config-storage
          mountPath: "/etc/virtual-kubelet"
          readOnly: true
      volumes:
      - name: kubeconfig-storage
        configMap:
          name: remote-kubeconfig
          items:
          - key: "config"
            path: "kubeconfig.yaml"
      - name: vk-config-storage
        configMap:
          name: vk-config
          items:
          - key: "config.yaml"
            path: "config.yaml"
EOF
```

Apply:

```bash
kubectl apply -f 03_vk_deployment.yaml
```

Expected:

```text
deployment.apps/cisco-virtual-kubelet created
```

---

### 5.5 File: `04_vk_pod_hello-app.yaml` (hello-app Pod on C8Kv)

```bash
cat > 04_vk_pod_hello-app.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: iox-xe-hello-app-pod
  namespace: default
spec:
  nodeName: cat8kv-node
  containers:
  - name: test-app
    image: flash:/hello-app.iosxe.tar
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
EOF
```

Apply:

```bash
kubectl apply -f 04_vk_pod_hello-app.yaml
```

Expected:

```text
pod/iox-xe-hello-app-pod created
```

---

## 6. Deploy Virtual Kubelet and hello-app

Monitor the deployment:

```bash
kubectl get pods -o wide -w
```

Expected progression:

```text
cisco-virtual-kubelet-xxxxx   1/1   Running   10.0.0.x     ubuntu-lab
iox-xe-hello-app-pod          1/1   Running   198.18.102.x cat8kv-node
```

---

## 7. Verification

Use the Pod IP shown in the output (example: `198.18.102.3`) and validate the app:

```bash
curl http://198.18.102.3:8080
```

Expected:

```text
Hello, world!
Version: 1.0.0
Hostname: cvkxxxxxxxxxxxxxxxx
```

---

## Summary

✔ Pulled and packaged `hello-app` as an IOS XE TAR
✔ Uploaded the application TAR to router flash
✔ Deployed Cisco Virtual Kubelet into Kubernetes
✔ Scheduled a pod to the C8Kv node (`cat8kv-node`)
✔ Validated application reachability from the lab server

---

[⬅ Back to Main Menu](../index.md)

```
