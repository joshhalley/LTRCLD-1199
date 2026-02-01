# Task 1: App-Hosting Deployment on IOS-XE (Manual)

[⬅️ Back to Main Menu](../index.md)

## Table of Contents

## Table of Contents

- [Step 1: Check Container Registry and Retrieve Image](#step-1-check-container-registry-and-retrieve-image)
- [Step 2: Create TAR Image from Docker](#step-2-create-tar-image-from-docker)
- [Step 3: SCP File to cat8Kv-task-1](#step-3-scp-file-to-cat8kv-task-1)
- [Step 4: SCP File to cat8Kv-task-3](#step-4-scp-file-to-cat8kv-task-3)
- [Step 5: App Hosting Configuration on cat8Kv-task-1](#step-5-app-hosting-configuration-on-cat8kv-task-1)
- [Step 6: App Hosting Install on cat8Kv-task-1](#step-6-app-hosting-install-on-cat8kv-task-1)
- [Step 7: App Hosting Activate on cat8Kv-task-1](#step-7-app-hosting-activate-on-cat8kv-task-1)
- [Step 8: App Hosting Run on cat8Kv-task-1](#step-8-app-hosting-run-on-cat8kv-task-1)
- [Step 9: App Hosting Configuration on cat8Kv-task-3](#step-9-app-hosting-configuration-on-cat8kv-task-3)
- [Step 10: App Hosting Install on cat8Kv-task-3](#step-10-app-hosting-install-on-cat8kv-task-3)
- [Step 11: App Hosting Activate on cat8Kv-task-3](#step-11-app-hosting-activate-on-cat8kv-task-3)
- [Step 12: App Hosting Run on cat8Kv-task-3](#step-12-app-hosting-run-on-cat8kv-task-3)

---

This task focuses on deploying a containerized troubleshooting application on a Cisco IOS-XE router using the **App Hosting** feature.

The process includes:

* Pulling the image from a container registry
* Converting the image to a TAR package
* Copying the image to the router using SCP
* Installing and activating the application using App Hosting
* Testing the deployed application
* We will deploy 2 containers in this task

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

```bash
dcloud@ubuntu-lab:~$ curl -s http://198.18.5.101:5000/v2/_catalog
{"repositories":["hello-app","mrtg","swiss-knife-alpine","wireshark"]}
```


```bash
sudo docker pull 198.18.5.101:5000/swiss-knife-alpine:latest
```
```bash
sudo docker pull 198.18.5.101:5000/mrtg:latest
```

Verify the image was downloaded:

```bash
sudo docker images
```
```bash
dcloud@ubuntu-lab:~$ sudo docker images
REPOSITORY                             TAG       IMAGE ID       CREATED       SIZE
198.18.5.101:5000/swiss-knife-alpine   latest    987b29fae1f2   10 days ago   396MB
registry                               2         26b2eb03618e   2 years ago   25.4MB
dcloud@ubuntu-lab:~$ 
```
---

## Step 2: Create TAR Image from Docker

* Convert the Docker image into a TAR archive
* This file will be transferred to the router

```bash
sudo docker save 198.18.5.101:5000/swiss-knife-alpine:latest -o swiss-knife-alpine.tar
```

```bash
sudo docker save 198.18.5.101:5000/mrtg:latest -o mrtg.tar
```


Verify the TAR file exists:

```bash
ls -lh swiss-knife-alpine.tar
```

---

## Step 3: SCP File to cat8Kv-task-1

* Login to cat8Kv-task-1 using 198.18.1.11
* Enable SCP server

```text
conf t
ip scp server enable
end
```

* Copy the TAR image from your lab ubuntu to the router

```bash
sudo scp swiss-knife-alpine.tar admin@198.18.1.11:/flash:/swiss-knife-alpine.tar
```

Verify the file on the router:

```bash
dir bootflash: | include swiss-knife-alpine.tar
```

---

## Step 4: SCP File to cat8Kv-task-3

* Login to cat8Kv-task-3 using 198.18.1.13
* Enable SCP server

```text
conf t
ip scp server enable
end
```

* Copy the TAR image from your lab ubuntu to the router

```bash
sudo scp mrtg.tar admin@198.18.1.13:/flash:/mrtg.tar
```

Verify the file on the router:

```bash
dir bootflash: | include mrtg.tar
```
---

## Step 5: App Hosting Configuration on cat8kv-task-1

* Login to cat8Kv-task-1 using 198.18.1.11
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

## Step 6: App Hosting Install on cat8kv-task-1

* Install the application from the TAR file
* This step extracts and prepares the container
* Enable terminal monitor to see the logs for progress

```bash
app-hosting install appid swiss_knife package bootflash:swiss-knife-alpine.tar
```
In case the following error is seen, 
```text
ioxman: app-hosting: Failed to install swiss_knife: App signature validation is required. App signature file package.cert or package.sign not found in package
```
Enable and disable app hosting signature verification and run the install command again

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

Validate installation:

```bash
show app-hosting detail appid swiss_knife
```

---

## Step 7: App Hosting Activate on cat8kv-task-1

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

## Step 8: App Hosting Run on cat8kv-task-1

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

## Step 9: App Hosting Configuration on cat8kv-task-3

* Login to cat8Kv-task-3 using 198.18.1.13
* Enable IOx application framework 
* Create a virtual interface for the container and advertise in OSPF
* Configure network connectivity for the application
* Allocate IP address and gateway for the container

```bash
conf t

 iox

 int virtualportgroup0
 ip address 198.18.102.1 255.255.255.0
 ip ospf 1 area 0

app-hosting appid mrtg
 app-vnic gateway0 virtualportgroup 0 guest-interface 0
  guest-ipaddress 198.18.102.5 netmask 255.255.255.0
 app-default-gateway 198.18.102.1 guest-interface 0
 name-server0 8.8.8.8

end
```

Verify configuration:

```bash
show run | sec app-hosting
```

---

## Step 10: App Hosting Install on cat8kv-task-3

* Install the application from the TAR file
* This step extracts and prepares the container
* Enable terminal monitor to see the logs for progress

```bash
app-hosting install appid mrtg package bootflash:mrtg.tar
```
In case the following error is seen, 
```text
ioxman: app-hosting: Failed to install swiss_knife: App signature validation is required. App signature file package.cert or package.sign not found in package
```
Enable and disable app hosting signature verification and run the install command again

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

Validate installation:

```bash
show app-hosting detail appid mrtg
```

---

## Step 11: App Hosting Activate on cat8kv-task-3

* Activate the application
* Makes it ready to start

```bash
app-hosting activate appid mrtg
```

Verify state:

```bash
show app-hosting list
```

---

## Step 12: App Hosting Run on cat8kv-task-3

* Start the container application

```bash
app-hosting start appid mrtg
```

Ensure it is running:

```bash
show app-hosting detail appid mrtg
```

Connect to the container and explore

```bash
app-hosting connect appid mrtg session /bin/bash
```

---

[⬅️ Return to Main Menu](../index.md)
