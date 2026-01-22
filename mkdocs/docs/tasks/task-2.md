# Task 2: App-Hosting Automation with Terraform

[⬅️ Back to Main Menu](../index.md)

## Table of Contents

- [Prerequisites: Router Preparation](#prerequisites-router-preparation)
- [Step 1: Check Container Registry and retrieve image](#step-1-check-container-registry-and-retrieve-image)
- [Step 2: Create TAR Image from Docker](#step-2-create-tar-image-from-docker)
- [Step 3: SCP File to Router](#step-3-scp-file-to-router)
- [Step 4: Verify MD5 Hash](#step-4-verify-md5-hash)
- [Step 5: Verify Tool Versions](#step-5-verify-tool-versions)
- [Step 6: Clone the App-Hosting Terraform Provider](#step-6-clone-the-app-hosting-terraform-provider)
- [Step 7: Configure Terraform to Use the Local Provider](#step-7-configure-terraform-to-use-the-local-provider)
- [Step 8: Create the Terraform Working Directory](#step-8-create-the-terraform-working-directory)
- [Step 9: Paste and Review `main.tf` to install the swiss-knife container](#step-9-paste-and-review-maintf-to-install-the-swiss-knife-container)
- [Step 10: Terraform Init](#step-10-terraform-init)
- [Step 11: Terraform Plan and Apply](#step-11-terraform-plan-and-apply)
- [Step 12: Verify Deployment on the Router](#step-12-verify-deployment-on-the-router)
- [Step 13: Install Wireshark container](#step-13-install-wireshark-container)
- [Step 14: Terraform Init](#step-14-terraform-init)
- [Step 15: Terraform Plan and Apply](#step-15-terraform-plan-and-apply)
- [Step 16: Verify Deployment on the Router](#step-16-verify-deployment-on-the-router)

---

This task focuses on deploying and managing a containerized application on a **Cisco IOS XE Router** router using **Terraform (Infrastructure as Code)**.

You will:

* Retrieve the container image from a registry
* Convert it to a TAR package
* Transfer it to the router using SCP
* Review and use an existing Terraform project
* Deploy the app-hosting configuration via Terraform
* Verify the container is running on the C8Kv
* Test the installed tools

---
## Prerequisites: Router Preparation

Before starting **Task-2**, the C8000v router must be prepared for **RESTCONF** access and **IOX App-Hosting**.
Terraform uses RESTCONF APIs to manage App-Hosting resources, and the lab container images are **unsigned**, so signature verification must be disabled.

Run the following commands on the **cat8Kv-task-2** SSH 198.18.1.12:

```ios
conf t
!
ip http secure-server
restconf
!
iox
 app-hosting signed-verification 
 !
 !
 !
 no app-hosting signed-verification 
!
end
wr mem
```

After completing the configuration, verify:

```ios
show ip http server status
show iox-service
show app-hosting infra
```

From the **Lab Ubuntu VM**, verify RESTCONF connectivity before running Terraform:

```bash
curl -k -u admin:C1sco12345 -H "Accept: application/yang-data+json" \
https://198.18.9.12/restconf/data/Cisco-IOS-XE-native:native/iox
```

Expected response:

```json
{
  "Cisco-IOS-XE-native:iox": {}
}
```

Once these checks succeed, proceed to deployning the APP via Terraform.

---

## Step 1: Check Container Registry and retrieve image

* Login to the Lab Ubuntu (ssh 198.18.1.100)
* Check the docker registry 
* Pull the required image from a container registry
* This image will be used to create a TAR file for router deployment
* In this task we will deploy 2 containers in cat8Kv-task-2, wireshark and swiss_knife (pulled in task 1)

```bash
curl -s http://198.18.5.101:5000/v2/_catalog
```
Sample Ouptut
{"repositories":["mrtg","swiss-knife-alpine","telegraf-alpine","wireshark"]}

```bash
docker pull 198.18.5.101:5000/wireshark:latest
```

Validate the download:

```bash
docker images
```

---

## Step 2: Create TAR Image from Docker

• Convert the Docker image to a TAR package
• This is the image format required for IOS-XE app hosting

```bash
docker save 198.18.5.101:5000/wireshark:latest -o wireshark.tar
```

Verify the TAR file exists:

```bash
ls -lh wireshark.tar
```

---

## Step 3: SCP File to Router

* Copy the wireshark and swiss_knife TAR images from your machine to the router
* The file will be stored in bootflash
* Initiate the copy from the router cat8Kv-task-2
* Login to cat8Kv-task-3 using 198.18.1.12

Copy swiss knife container
```bash
cat8Kv-task-2#copy scp: bootflash:
Address or name of remote host []? 198.18.9.100
Source username [admin]? root
Source filename []? swiss-knife-alpine.tar
Destination filename [swiss-knife-alpine.tar]? 
```
Copy wireshark container
```bash
cat8Kv-task-2#copy scp: bootflash:
Address or name of remote host []? 198.18.9.100
Source username [admin]? root
Source filename []? wireshark.tar
Destination filename [wireshark.tar]? 
```

Verify the file on the router:

```bash
dir bootflash: | include tar
```

---

## Step 4: Verify MD5 Hash

* Confirm file integrity on the router
* Ensures no corruption occurred during transfer

```bash
verify /md5 bootflash:wireshark.tar
```

Compare with local checksum:

```bash
md5sum wireshark.tar
```

## Step 5: Verify Tool Versions

Terraform and Go are already installed on the Lab Ubuntu (SSH 198.18.1.100)
Verify the versions before proceeding.

### Verify Terraform Version

```bash
terraform version
```

Expected output (ignore the out of date message):

```text
Terraform v1.6.6
```

> ℹ️ Terraform 1.6.6 is supported for this lab.

---

### Verify Go Version

```bash
go version
```

Expected output:

```text
go version go1.18.1 linux/amd64
```

---

## Step 6: Clone the App-Hosting Terraform Provider

Clone the Terraform provider repository from the **instructor VM**:

```bash
git clone ssh://lab@198.18.5.101/home/lab/terraform-provider-ciscoapphosting
```

Enter the **lab user password** when prompted. password is "lab"

Build the provider binary:

```bash
cd terraform-provider-ciscoapphosting
go build -o terraform-provider-ciscoapphosting

```

---

## Step 7: Configure Terraform to Use the Local Provider

Terraform must be instructed to use the **local provider** instead of the public registry.

Create the Terraform CLI configuration file:

```bash
nano ~/.terraformrc
```

Paste the following content:

```hcl
provider_installation {
  dev_overrides {
    "local/ciscoapphosting" = "/root/terraform-provider-ciscoapphosting"
  }
  direct {}
}
```

Save and exit.

> ⚠️ A warning about *provider development overrides* is expected and normal.

---

## Step 8: Create the Terraform Working Directory

Create a directory for the App-Hosting deployment:

```bash
mkdir ~/terraform-c8kv-apphosting
cd ~/terraform-c8kv-apphosting
```

Create a single Terraform file:

```bash
nano main.tf
```

---

## Step 9: Paste and Review `main.tf` to install the swiss-knife container

Paste the following configuration **as provided** (modify IP addresses only if instructed):

```hcl
terraform {
  required_providers {
    ciscoapphosting = {
      source  = "local/ciscoapphosting"
      version = "0.1.0"
    }
  }
}

provider "ciscoapphosting" {
  username = "admin"
  password = "C1sco12345"
  debug    = true
}

resource "ciscoapphosting_app" "swiss_knife" {
  host     = "198.18.9.12"
  name     = "swiss_knife"
  platform = "c8000v"

  image    = "bootflash:swiss-knife-alpine.tar"

  vpg_id   = 0
  vpg_ip   = "198.18.101.1"
  vpg_mask = "255.255.255.0"

  guest_ip      = "198.18.101.5"
  guest_netmask = "255.255.255.0"
  guest_gateway = "198.18.101.1"
  nameserver0   = "8.8.8.8"

  docker = true
}
```

Review the file:

```bash
cat main.tf
```

Confirm:

* Correct router IP
* Correct image name on `bootflash`
* `vpg_id = 0`

---

## Step 10: Terraform Init

Initialize the Terraform project:

```bash
terraform init
```

Expected output includes:

```text
Warning: Provider development overrides are in effect
```

✅ This is expected
❌ Do not attempt to remove this warning

---

## Step 11: Terraform Plan and Apply

Review the execution plan:

```bash
terraform plan
```
Enable `terminal monitor` on cat8kv-task-2 to observe the app hosting logs

Apply the configuration:

```bash
terraform apply
```

When prompted, type:

```text
yes
```

Expected output:

```text
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

---

## Step 12: Verify Deployment on the Router

On the cat8Kv-task-2 router, verify the application and networking:

```bash
show app-hosting list
show run interface VirtualPortGroup0
show run | section ospf
```

You should see:

* App state: **RUNNING**
* `VirtualPortGroup0` configured with IP address
* `router ospf 1` with a network statement for the VPG IP

Connect to the container and explore

```bash
app-hosting connect appid swiss_knife session /bin/bash
```

## Step 13: Install Wireshark container

Update and Review `main.tf` \
Paste the following configuration **as provided** at the end of main.tf created above (modify IP addresses only if instructed):

```bash
nano main.tf
```

```hcl

resource "ciscoapphosting_app" "wireshark" {
  host     = "198.18.9.12"
  name     = "wireshark"
  platform = "c8000v"

  image    = "bootflash:wireshark.tar"

  vpg_id   = 0
  vpg_ip   = "198.18.101.1"
  vpg_mask = "255.255.255.0"

  guest_ip      = "198.18.101.6"
  guest_netmask = "255.255.255.0"
  guest_gateway = "198.18.101.1"
  nameserver0   = "8.8.8.8"

  docker = true
}
```

Review the file:

```bash
cat main.tf
```

Confirm:

* Correct router IP
* Correct image name on `bootflash`
* `vpg_id = 0`

---

## Step 14: Terraform Init

Initialize the Terraform project:

```bash
terraform init
```

Expected output includes:

```text
Warning: Provider development overrides are in effect
```

✅ This is expected
❌ Do not attempt to remove this warning

---

## Step 15: Terraform Plan and Apply

Review the execution plan:

```bash
terraform plan
```

Apply the configuration:

```bash
terraform apply
```

When prompted, type:

```text
yes
```

Expected output:

```text
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

---

## Step 16: Verify Deployment on the Router

On the cat8Kv-task-2 router, verify the application and networking:

```bash
show app-hosting list
```

You should see:

* App state: **RUNNING**
* Now we have 2 containers running

Connect to the container and explore

```bash
app-hosting connect appid wireshark session /bin/bash
```

---

[⬅️ Return to Main Menu](../index.md)
