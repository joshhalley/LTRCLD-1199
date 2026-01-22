# Task 1: App-Hosting Deployment on IOS-XE (Manual)

[⬅️ Back to Main Menu](../index.md)

## Table of Contents

- [Step 1: Check Container Registry and retrieve image](#step-1-check-container-registry-and-retrieve-image)
- [Step 2: Create TAR Image from Docker](#step-2-create-tar-image-from-docker)
- [Step 3: SCP File to Router](#step-3-scp-file-to-router)
- [Step 4: Verify MD5 Hash](#step-4-verify-md5-hash)
- [Step 5: App Hosting Configuration](#step-5-app-hosting-configuration)
- [Step 6: App Hosting Install](#step-6-app-hosting-install)
- [Step 7: App Hosting Activate](#step-7-app-hosting-activate)
- [Step 8: App Hosting Run](#step-8-app-hosting-run)

---

This task focuses on deploying a containerized troubleshooting application on a Cisco IOS-XE router using the **App Hosting** feature.

The process includes:

* Pulling the image from a container registry
* Converting the image to a TAR package
* Copying the image to the router using SCP
* Installing and activating the application using App Hosting
* Testing the deployed application

---

## Step 1: Check Container Registry and retrieve image

* Login to the Lab Ubuntu (ssh 198.18.1.100)
* Check the docker registry 
* Pull the required image from a container registry
* This image will be used to create a TAR file for router deployment

```bash
curl -s http://198.18.5.101:5000/v2/_catalog
```
Sample output
root@ubuntu-lab:~# curl -s http://198.18.5.101:5000/v2/_catalog
{"repositories":["mrtg","swiss-knife-alpine","telegraf-alpine","wireshark"]}


```bash
docker pull 198.18.5.101:5000/swiss-knife-alpine:latest
```

Verify the image was downloaded:

```bash
docker images
```
root@ubuntu-lab:~# docker images
REPOSITORY                             TAG       IMAGE ID       CREATED      SIZE
198.18.5.101:5000/swiss-knife-alpine   latest    ef61409cede7   2 days ago   270MB
root@ubuntu-lab:~# 

---

## Step 2: Create TAR Image from Docker

* Convert the Docker image into a TAR archive
* This file will be transferred to the router

```bash
docker save 198.18.5.101:5000/swiss-knife-alpine:latest -o swiss-knife-alpine.tar
```

Verify the TAR file exists:

```bash
ls -lh swiss-knife-alpine.tar
```

---

## Step 3: SCP File to Router

* Copy the TAR image from your machine to the router
* The file will be stored in bootflash
* Initiate the copy from the router cat8Kv-task-1
* Login to cat8Kv-task-1 using 198.18.1.11

```bash
cat8Kv-task-1#copy scp: bootflash:
Address or name of remote host []? 198.18.9.100
Source username [admin]? root
Source filename []? swiss-knife-alpine.tar
Destination filename [swiss-knife-alpine.tar]? 
```

Verify the file on the router:

```bash
dir bootflash: | include swiss-knife-alpine.tar
```

---

## Step 4: Verify MD5 Hash

* Confirm file integrity on the router
* Ensures no corruption occurred during transfer

```bash
verify /md5 bootflash:swiss-knife-alpine.tar
```

Compare with local checksum:

```bash
md5sum swiss-knife-alpine.tar
```

---

## Step 5: App Hosting Configuration

* Enable IOx application framework 
* Create a virtual interface for the container and advertise in OSPF
* Configure network connectivity for the application
* Allocate IP address and gateway for the container

```bash
conf t

 iox

 int virtualportgroup0
 ip address 198.18.100.1 255.255.255.0
 ip ospf 1 area 0

app-hosting appid swiss_knife
 app-vnic gateway0 virtualportgroup 0 guest-interface 0
  guest-ipaddress 198.18.100.5 netmask 255.255.255.0
 app-default-gateway 198.18.100.1 guest-interface 0
 name-server0 8.8.8.8

end
```

Verify configuration:

```bash
show run | sec app-hosting
```

---

## Step 6: App Hosting Install

* Install the application from the TAR file
* This step extracts and prepares the container
* Enable terminal monitor to see the logs for progress

```bash
app-hosting install appid swiss_knife package bootflash:swiss-knife-alpine.tar
```
In case the following error is seen
App signature validation is required. App signature file package.cert or package.sign not found in package
enable and disable app hosting signature verification 

```bash
cat8Kv-task-1(config)#app-hosting signed-verification 
cat8Kv-task-1(config)#
*Jan 13 12:00:22.429: %IM-6-VERIFICATION_MSG: R0/0: ioxman: app-hosting: App signature verification enabled successfully
cat8Kv-task-1(config)#
cat8Kv-task-1(config)#no app-hosting signed-verification 
cat8Kv-task-1(config)#
*Jan 13 12:00:35.447: %IM-6-VERIFICATION_MSG: R0/0: ioxman: app-hosting: App signature verification disabled successfully
cat8Kv-task-1(config)#
```

Validate installation:

```bash
show app-hosting detail appid swiss_knife
```

---

## Step 7: App Hosting Activate

* Activate the application
* Makes it ready to start

```bash
app-hosting activate appid swiss_knife
```

Verify state:

```bash
show app-hosting list
```

---

## Step 8: App Hosting Run

* Start the container application

```bash
app-hosting start appid swiss_knife
```

Ensure it is running:

```bash
show app-hosting detail appid swiss_knife
```

Connect to the container and explore

```bash
app-hosting connect appid swiss_knife session /bin/bash
```

---

[⬅️ Return to Main Menu](../index.md)
