
## Step 1 — Create SSH key on Ubuntu (as `dcloud`)

On the **Ubuntu box**:

```bash
su - dcloud
ssh-keygen -t rsa -b 1024 -f ~/.ssh/c8kv_admin_key
```

Just press **Enter** for passphrase (or set one if you want).

This creates:

* `~/.ssh/c8kv_admin_key`        (private)
* `~/.ssh/c8kv_admin_key.pub`    (public)

---

## Step 2 — Copy **public key** to the routers

Show the public key:

```bash
cat ~/.ssh/c8kv_admin_key.pub
```

It will look like:

```
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQ... dcloud@ubuntu
```

Copy **the entire line**.

---

## Step 3 — Configure SSH key on the all the 3 Cat8Kv routers (IOS-XE)

On the router CLI:

```text
conf t
ip ssh pubkey-chain
username admin
key-string
<ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQ... dcloud@ubuntu>
end
write memory
```

⚠️ Paste the key **in one line**, no line breaks.

---

## Step 4 — Ensure SSH is enabled on router

Still on the router:

```text
conf t
ip domain name dmz.cisco.com
crypto key generate rsa modulus 2048
ip ssh version 2
end
```

---

## Step 5 — SSH from Ubuntu using the key

From Ubuntu:

```bash
ssh -i ~/.ssh/c8kv_admin_key admin@198.18.6.11
```

You should log in **without password** 🎉

---

## Step 6 Simplify with SSH config

On Ubuntu:

```bash
nano ~/.ssh/config
```

Add:

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

Now connect with:

```bash
ssh c8kv-task-1
```
Sample Output

```bash
dcloud@ubuntu-lab:~$ ssh c8kv-task-2
The authenticity of host '198.18.7.12 (198.18.7.12)' can't be established.
RSA key fingerprint is SHA256:A0uoPV8ZDbz2G3l5jAB9+/lFI7sX9hDyVRvInE9Uiq0.
This host key is known by the following other names/addresses:
    ~/.ssh/known_hosts:1: [hashed name]
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '198.18.7.12' (RSA) to the list of known hosts.

cat8Kv-task-2#
```

---

## Step 6 Execute commands on routers directly from Ubuntu

```bash
ssh c8kv-task-1 show ip int brief
```
Sample Output

```bash
dcloud@ubuntu-lab:~$ ssh c8kv-task-1 show ip int brief


Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet4       198.18.1.11     YES NVRAM  up                    up      
GigabitEthernet5       198.18.9.11     YES NVRAM  up                    up      
GigabitEthernet6       198.18.6.11     YES NVRAM  up                    up      Connection to 198.18.6.11 closed by remote host.
```
---