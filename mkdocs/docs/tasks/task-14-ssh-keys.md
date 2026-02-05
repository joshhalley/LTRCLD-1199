# Task14: Passwordless Access using SSH Keys

[⬅️ Back to Main Menu](../index.md)

---

## Table of Contents

* [Objective](#objective)
* [Prerequisites](#prerequisites)
* [Step 1: Create SSH Key on Ubuntu](#step-1-create-ssh-key-on-ubuntu)
* [Step 2: Copy Public Key from Ubuntu](#step-2-copy-public-key-from-ubuntu)
* [Step 3: Configure SSH Public Key on Cat8Kv Routers](#step-3-configure-ssh-public-key-on-cat8kv-routers)
* [Step 4: Ensure SSH is Enabled on the Router](#step-4-ensure-ssh-is-enabled-on-the-router)
* [Step 5: SSH from Ubuntu using the Key](#step-5-ssh-from-ubuntu-using-the-key)
* [Step 6: Simplify Access using SSH Config](#step-6-simplify-access-using-ssh-config)
* [Step 7: Execute Router Commands Directly from Ubuntu](#step-7-execute-router-commands-directly-from-ubuntu)

---

## Objective

In this task, you will configure **passwordless SSH access** from the `ubuntu-lab` host to multiple **Catalyst 8000v (Cat8Kv)** routers using **SSH public key authentication**. This enables secure, automation-friendly access without interactive password prompts.

---

## Prerequisites

* Ubuntu host with user `dcloud`
* SSH client installed (default on Ubuntu)
* Administrative access to Cat8Kv routers

---

## Step 1: Create SSH Key on Ubuntu

Log in to the Ubuntu system and switch to the `dcloud` user:

```bash
su - dcloud
```

Generate an RSA SSH key pair:

```bash
ssh-keygen -t rsa -b 1024 -f ~/.ssh/c8kv_admin_key
```

When prompted for a passphrase, press **Enter** to leave it empty (or set one if required).

This creates the following files:

* `~/.ssh/c8kv_admin_key` – **Private key**
* `~/.ssh/c8kv_admin_key.pub` – **Public key**

---

## Step 2: Copy Public Key from Ubuntu

Display the public key:

```bash
cat ~/.ssh/c8kv_admin_key.pub
```

Example output:

```text
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQ... dcloud@ubuntu
```

Copy the **entire line**, including `ssh-rsa` and the comment at the end.

---

## Step 3: Configure SSH Public Key on Cat8Kv Routers

Repeat the following steps on **all three Cat8Kv routers**.

From the router CLI:

```text
conf t
ip ssh pubkey-chain
 username admin
  key-string
   ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQ... dcloud@ubuntu
  exit
 exit
end
write memory
```

⚠️ **Important:**

* Paste the key on **one single line**
* Do **not** add line breaks or extra spaces

---

## Step 4: Ensure SSH is Enabled on the Router

Still on the router CLI, ensure SSH prerequisites are configured:

```text
conf t
ip domain name dmz.cisco.com
crypto key generate rsa modulus 2048
ip ssh version 2
end
```

> If RSA keys already exist, the router may skip regeneration.

---

## Step 5: SSH from Ubuntu using the Key

From the Ubuntu host:

```bash
ssh -i ~/.ssh/c8kv_admin_key admin@198.18.6.11
```

You should log in **without being prompted for a password**.

---

## Step 6: Simplify Access using SSH Config

To avoid specifying the key and IP address every time, configure SSH aliases.

Edit the SSH config file:

```bash
nano ~/.ssh/config
```

Add the following entries:

```text
Host c8kv-task-1
  HostName 198.18.6.11
  User admin
  IdentityFile ~/.ssh/c8kv_admin_key

Host c8kv-task-2
  HostName 198.18.7.12
  User admin
  IdentityFile ~/.ssh/c8kv_admin_key

Host c8kv-task-3
  HostName 198.18.8.13
  User admin
  IdentityFile ~/.ssh/c8kv_admin_key
```

Now connect using a simple command:

```bash
ssh c8kv-task-1
```

### Sample Output

```text
The authenticity of host '198.18.7.12 (198.18.7.12)' can't be established.
RSA key fingerprint is SHA256:A0uoPV8ZDbz2G3l5jAB9+/lFI7sX9hDyVRvInE9Uiq0.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '198.18.7.12' (RSA) to the list of known hosts.

cat8Kv-task-2#
```

---

## Step 7: Execute Router Commands Directly from Ubuntu

You can now execute IOS-XE commands non-interactively from Ubuntu.

Example:

```bash
ssh c8kv-task-1 show ip int brief
```

### Sample Output

```text
Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet4       198.18.1.11     YES NVRAM  up                    up
GigabitEthernet5       198.18.9.11     YES NVRAM  up                    up
GigabitEthernet6       198.18.6.11     YES NVRAM  up                    up
Connection to 198.18.6.11 closed by remote host.
```

---

✔️ At this point, passwordless SSH access is fully configured for all Cat8Kv routers, enabling fast CLI access and seamless automation from the Ubuntu host.
