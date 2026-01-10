# Task 2: App Hosting Automation with Terraform

This task focuses on deploying and managing a containerized application on a **Cisco IOS XE Router** router using **Terraform (Infrastructure as Code)**.

You will:

* Retrieve the container image from a registry
* Convert it to a TAR package
* Transfer it to the router using SCP
* Review and use an existing Terraform project
* Deploy the app-hosting configuration via Terraform
* Verify the container is running on the C8Kv
* Test the installed tools

**Steps in this task:**

* [Prerequisites: Router Preparation](#prerequisites-router-preparation)
* [Step 1: Retrieve Image from Container Registry](#step-1-retrieve-image-from-container-registry)
* [Step 2: Create TAR Image from Docker](#step-2-create-tar-image-from-docker)
* [Step 3: SCP File to Router](#step-3-scp-file-to-router)
* [Step 4: Verify MD5 Hash](#step-4-verify-md5-hash)
* [Step 5: Verify Tool Versions](#step-5-verify-tool-versions)
* [Step 6: Clone and Build the App-Hosting Terraform Provider](#step-6-clone-and-build-the-app-hosting-terraform-provider)
* [Step 7: Configure Terraform to Use the Local Provider](#step-7-configure-terraform-to-use-the-local-provider)
* [Step 8: Create the Terraform Working Directory](#step-8-create-the-terraform-working-directory)
* [Step 9: Review and Paste main.tf](#step-9-review-and-paste-maintf)
* [Step 10: Terraform Init](#step-10-terraform-init)
* [Step 11: Terraform Plan and Apply](#step-11-terraform-plan-and-apply)
* [Step 12: Verify Deployment on the Router](#step-12-verify-deployment-on-the-router)
* [Step 13: Test Tool A](#step-13-test-tool-a)
* [Step 14: Test Tool B](#step-14-test-tool-b)
* [Step 15: Test Tool C](#step-15-test-tool-c)

---
## Prerequisites: Router Preparation

Before starting **Task-2**, the C8000v router must be prepared for **RESTCONF** access and **IOX App-Hosting**.
Terraform uses RESTCONF APIs to manage App-Hosting resources, and the lab container images are **unsigned**, so signature verification must be disabled.


Run the following commands on the **C8000v router**:

```ios
conf t
!
ip http secure-server
restconf
!
iox
 app-hosting signature-verification
 no app-hosting signature-verification
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
curl -k -u <USERNAME>:<PASSWORD> -H "Accept: application/yang-data+json" \
https://<ROUTER_IP>/restconf/data/Cisco-IOS-XE-native:native/iox
```

Expected response:

```json
{
  "Cisco-IOS-XE-native:iox": {}
}
```

Once these checks succeed, proceed to **Task-2: Deploy App-Hosting Using Terraform**.

---

## Step 1: Retrieve Image from Container Registry

• Pull the required image from the container registry
• This image will be used by the Terraform-managed app-hosting workflow

```code
docker pull 198.18.5.101:5000/swiss-knife:task-2
```

Validate the download:

```code
docker images
```

---

## Step 2: Create TAR Image from Docker

• Convert the Docker image to a TAR package
• This is the image format required for IOS-XE app hosting

```code
docker save 198.18.5.101:5000/swiss-knife:task-2 -o swiss-knife-task-2.tar
```

Verify the TAR file exists:

```code
ls -lh swiss-knife-task-2.tar
```

---


## Step 3: SCP File to Router

* Copy the TAR image from your machine to the router
* The file will be stored in bootflash
* Initiate the copy from the router cat8Kv-task-2

```code
cat8Kv-task-2#copy scp: bootflash:
Address or name of remote host []? 198.18.9.100
Source username [admin]? root
Source filename []? swiss-knife-task-2.tar
Destination filename [swiss-knife-task-2.tar]? 
```

Verify the file on the router:

```code
dir bootflash: | include swiss-knife-task-2.tar
```

---

## Step 4: Verify MD5 Hash

* Confirm file integrity on the router
* Ensures no corruption occurred during transfer

```code
verify /md5 bootflash:swiss-knife-task-1.tar
```

Compare with local checksum:

```code
md5sum swiss-knife-task-1.tar
```


## Step 5: Verify Tool Versions

Terraform and Go are already installed on the participant VM.
Verify the versions before proceeding.

### Verify Terraform Version

```bash
terraform version
```

Expected output (version may vary):

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
chmod +x terraform-provider-ciscoapphosting
```

---

## Step 7: Configure Terraform to Use the Local Provider

Terraform must be instructed to use the **local provider** instead of the public registry.

Create the Terraform CLI configuration file:

```bash
mkdir -p ~/.config
nano ~/.config/terraformrc
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

## Step 9: Paste and Review `main.tf`

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

  image    = "bootflash:swiss-knife-task-2.tar"

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

On the C8000v router, verify the application and networking:

```bash
show app-hosting list
show run interface VirtualPortGroup0
show run | section ospf
```

You should see:

* App state: **RUNNING**
* `VirtualPortGroup0` configured with IP address
* `router ospf 1` with a network statement for the VPG IP



## Step 13: Test Tool A

• Test the exposed service from your local machine

```code
curl http://10.1.1.2:8080/health
```

Expected output:

```code
OK
```

---

## Step 14: Test Tool B

• Verify connectivity to the container

```code
ping 10.1.1.2
```

Expected:
5/5 success

---

## Step 15: Test Tool C

• Connect to the container shell for live troubleshooting

```code
app-hosting connect appid netops-toolkit /bin/bash
```

Inside container:

```code
ifconfig
tcpdump -i eth0
```

Exit with:

```code
exit
```

---

* [Main Menu](/README.md/#table-of-content)

---

