# Task 3: Kubernetes App Hosting using Virtual Kubelet

[⬅️ Back to Main Menu](../index.md)

---

## Objective

In this task, you will deploy a **containerized application on a Cisco Catalyst 8000V (C8Kv)** router using **Kubernetes** and the **Cisco Virtual Kubelet Provider**.

The Kubernetes cluster is **already running**.

You will:

- Validate Kubernetes readiness
- Prepare the router for IOx app-hosting
- Deploy Cisco Virtual Kubelet
- Run `hello-app` on the router
- Verify application access

---

## Table of Contents

1. [Kubernetes Cluster Validation](#1-kubernetes-cluster-validation)  
1. [Container Registry Verification](#2-container-registry-verification)  
1. [Router Preparation (cat8kv-task-3)](#3-router-preparation-cat8kv-task-3)  
1. [Build and Upload hello-app Image](#4-build-and-upload-hello-app-image)  
1. [Create Kubernetes Manifests](#5-create-kubernetes-manifests)  
1. [Deploy Virtual Kubelet and hello-app](#6-deploy-virtual-kubelet-and-hello-app)  
1. [Verification](#7-verification)  

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

**Note:** The cluster Kubeconfig is available under ```~/.kube/config```

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

### 3.1 DHCP Relay Configuration

```text
conf t
interface virtualportgroup0
ip helper-address 198.18.1.102
end
```

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

### 5.3 File: `02_vk_deployment_config-r3.yaml` (VK Device + Node Mapping)

```bash
cat > 02_vk_deployment_config_r3.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: vk-config-r3
  namespace: default
data:
  config.yaml: |
    device:
      name: cat8kv-router-r3
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
      node_name: "cat8kv-node-r3"
      namespace: ""
      update_interval: "30s"
      os: "Linux"
      node_internal_ip: "198.18.8.13"
EOF
```

Apply:

```bash
kubectl apply -f 02_vk_deployment_config_r3.yaml
```

Expected:

```text
configmap/vk-config-r3 created
```

---

### 5.4 File: `03_vk_deployment_r3.yaml` (Virtual Kubelet Deployment)

```bash
cat > 03_vk_deployment-r3.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cisco-virtual-kubelet-r3
  labels:
    app: cisco-virtual-kubelet-r3
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cisco-virtual-kubelet-r3
  template:
    metadata:
      labels:
        app: cisco-virtual-kubelet-r3
    spec:
      nodeSelector:
        kubernetes.io/hostname: ubuntu-lab
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
          name: vk-config-r3
          items:
          - key: "config.yaml"
            path: "config.yaml"
EOF
```

Apply:

```bash
kubectl apply -f 03_vk_deployment-r3.yaml
```

Expected:

```text
deployment.apps/cisco-virtual-kubelet-r3 created
```

---

### 5.5 File: `04_vk_pod_hello-app-r3.yaml` (hello-app Pod on C8Kv)

```bash
cat > 04_vk_pod_hello-app-r3.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: iox-xe-hello-app-pod-r3-0
  namespace: default
spec:
  nodeName: cat8kv-node-r3
  containers:
  - name: test-app
    image: flash:/hello-app.iosxe.tar
    resources:
      requests:
        memory: "4Mi"
        cpu: "50m"
      limits:
        memory: "8Mi"
        cpu: "75m"
EOF
```

Apply:

```bash
kubectl apply -f 04_vk_pod_hello-app-r3.yaml
```

Expected:

```text
pod/iox-xe-hello-app-pod-r3-0 created
```

---

## 6. Deploy Virtual Kubelet and hello-app

Monitor the deployment:

```bash
kubectl get pods -o wide -w
```

Expected progression:

```text
dcloud@ubuntu-lab:~$ kubectl get pods -o wide -w
NAME                                     READY   STATUS              RESTARTS   AGE   IP          NODE          NOMINATED NODE   READINESS GATES
cisco-virtual-kubelet-7b58bd86db-mvnr2   1/1     Running             0          54s   10.0.0.61   ubuntu-lab    <none>           <none>
iox-xe-hello-app-pod-r3-0                     0/1     ContainerCreating   0          8s    0.0.0.0     cat8kv-node-r3   <none>           <none>
iox-xe-hello-app-pod-r3-0                     0/1     ContainerCreating   0          13s   0.0.0.0     cat8kv-node-r3   <none>           <none>
iox-xe-hello-app-pod-r3-0                     0/1     ContainerCreating   0          18s   0.0.0.0     cat8kv-node-r3   <none>           <none>
iox-xe-hello-app-pod-r3-0                     0/1     ContainerCreating   0          23s   198.18.102.175   cat8kv-node-r3   <none>           <none>
iox-xe-hello-app-pod-r3-0                     0/1     ContainerCreating   0          28s   198.18.102.175   cat8kv-node-r3   <none>           <none>
iox-xe-hello-app-pod-r3-0                     0/1     ContainerCreating   0          34s   198.18.102.175   cat8kv-node-r3   <none>           <none>

```

Expected output after successful deployment

```text
dcloud@ubuntu-lab:~$ kubectl get pods -o wide -w
NAME                                     READY   STATUS    RESTARTS   AGE     IP               NODE          NOMINATED NODE   READINESS GATES
cisco-virtual-kubelet-7b58bd86db-mvnr2   1/1     Running   0          8m9s    10.0.0.61        ubuntu-lab    <none>           <none>
iox-xe-hello-app-pod-r3-0                1/1     Running   0          7m23s   198.18.102.175   cat8kv-node-r3   <none>           <none>
```

---

## 7. Verification

Use the Pod IP shown in the output (example: `198.18.102.175`) and validate the app:

```bash
curl http://198.18.102.X:8080
```

Expected:

```text
Hello, world!
Version: 1.0.0
Hostname: cvkxxxxxxxxxxxxxxxx
```

---

## Key Takeways
- ✅  Pulled and packaged `hello-app` as an IOS XE TAR
- ✅  Uploaded the application TAR to router flash
- ✅  Deployed Cisco Virtual Kubelet into Kubernetes
- ✅  Scheduled a pod to the C8Kv node (`cat8kv-node`)
- ✅  Validated application reachability from the lab server

---

[⬅️ Return to Main Menu](../index.md)

```
