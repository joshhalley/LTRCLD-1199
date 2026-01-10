# Welcome to Cisco Live Amsterdam 2026


# LTRCLD-1199 – Using Containers on Cisco Edge devices to Turbocharge Troubleshooting

Welcome to **LTRCLD-1199**, a hands-on lab focused on deploying and operating containerized troubleshooting tools on **Cisco IOS-XE Router** using:

- Native App-Hosting
- Terraform (Infrastructure as Code)
- Kubernetes (KIND + virtual kubelet)

In this lab you will:

- Package and transfer container images to IOS-XE routers  
- Deploy and manage applications using App-Hosting  
- Automate deployments with Terraform  
- Integrate C8Kv with a Kubernetes-based control plane  
- Validate and test multiple network tools 

---

## Lab Tasks

Use the top navigation bar to jump directly to each task, or use the links below:

- [Task 1: App Hosting Deployment on IOS-XE](task-1.md#task-1-app-hosting-deployment-on-ios-xe)  
- [Task 2: App Hosting Automation with Terraform](task-2.md#task-2-app-hosting-automation-with-terraform)  
- [Task 3: Kubernetes-Based Container Orchestration](task-3.md#task-3-kubernetes-based-container-orchestration)

---

## Lab Flow

1. **Review the topology**  
   - Open the [Topology](topology.md) page to understand the devices, addressing, and roles.

2. **Get access details**  
   - Open the [Lab Access](access.md) page for jump host, VPN, and credentials.

3. **Complete the tasks in order**
   - Task 1 – Manual deployment using App-Hosting  
   - Task 2 – Automating deployment with Terraform  
   - Task 3 – Integrating with Kubernetes

---

## Prerequisites

To get the most value out of this lab, you should be familiar with:

- Basic IOS-XE CLI navigation  
- Fundamentals of Docker containers  
- Basic understanding of Terraform and/or YAML  
- Very basic Kubernetes concepts (Pods, Nodes) – helpful but not required

---

## Lab Objectives

By the end of this lab, you will be able to:

- Deploy a containerized tool on C8Kv using App-Hosting  
- Use SCP and MD5 verification to safely transfer images  
- Automate app-hosting configuration with Terraform  
- Integrate C8Kv into a Kubernetes environment using KIND and virtual kubelet  
- Validate that tools are running and accessible for troubleshooting work
---

