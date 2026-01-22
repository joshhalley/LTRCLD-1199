# Task 7: socat Practical Use Cases

[⬅️ Back to Main Menu](../index.md)

## Table of Contents

- [Accessing the Lab Containers](#accessing-the-lab-containers)
- [Use Case 1: Simple TCP Listener](#use-case-1-simple-tcp-listener)
- [Use Case 2: TCP Port Forwarding (Proxy) + Verbose/Debug](#use-case-2-tcp-port-forwarding-proxy-verbosedebug)
- [Use Case 3: UDP Send and Receive (IPv4 Explicit)](#use-case-3-udp-send-and-receive-ipv4-explicit)
- [Use Case 4: File Transfer Over TCP (TXT, One-Way)](#use-case-4-file-transfer-over-tcp-txt-one-way)
- [Use Case 5: Inline Traffic Inspection Using Two Router Containers](#use-case-5-inline-traffic-inspection-using-two-router-containers)

---

`socat` is a versatile data-transfer utility that can relay traffic between sockets, files, and processes.  
In this lab, `socat` is used to validate connectivity, simulate services, proxy traffic, generate traffic, transfer files, and inspect application flows running inside containers.

Here’s a **clean, lab-ready section** you can drop **immediately after the intro**.
It’s written in the same neutral, instructional tone you’ve been using elsewhere.

---

## Accessing the Lab Containers

For the following demonstrations, two **Swiss-Knife containers** are used.
Each container is deployed on a separate **Catalyst 8000v (C8Kv)** router using App-Hosting.

### Lab Setup Overview

| Component     | Management IP | Container IP   |
| ------------- | ------------- | -------------- |
| cat8kv-task-1 | `198.18.1.11` | `198.18.100.5` |
| cat8kv-task-2 | `198.18.1.12` | `198.18.101.5` |

---

### Step 1: SSH to both the Routers directly from the PC

Connect to the cat8kv-task-1 and cat8kv-task-2 hosting the swiss knife container.

### Step 2: Access the Swiss-Knife Container

Once logged into the router CLI, connect to the container shell:

```bash
app-hosting connect appid swiss_knife session /bin/bash
```

You will now be inside the Swiss-Knife container and can run `socat` commands directly.

---

### Notes

* All `socat` commands in this document are executed **inside the Swiss-Knife container**
* Container IPs are reachable end-to-end and are used for traffic generation, proxying, and inspection
* Multiple terminal sessions are recommended when running listeners and clients simultaneously

---

## Use Case 1: Simple TCP Listener

### Objective
Start a lightweight TCP service to receive and display incoming data.

### Command (Listener > Cat8Kv-task-1 > container)
```bash
socat -v TCP-LISTEN:8080,reuseaddr,fork STDOUT
```

### Test (Client > Cat8Kv-task-2 > container)

```bash
echo "hello from client" | socat - TCP:198.18.100.5:8080
```

### Expected Result

Incoming TCP connections are accepted and payloads are printed to the terminal.

---

## Use Case 2: TCP Port Forwarding (Proxy) + Verbose/Debug

### Objective

Forward traffic from a local TCP port to a remote service while logging connection and payload details.
We will SSH on port 9000 to the container's IP which will be redirected on port 22 to the router (cat8Kv-task-1) IP

### Command (Cat8Kv-task-1 > container)

```bash
socat -d -d -v TCP-LISTEN:9000,reuseaddr,fork TCP:198.18.100.1:22
```

### Optional (Hex dump of payloads)

```bash
socat -d -d -v -x TCP-LISTEN:9000,reuseaddr,fork TCP:198.18.100.1:22
```

### Test (Lab Ubuntu)

```bash
ssh -p 9000 admin@198.18.100.5
```
Give router's password

### Expected Result

Traffic received on port `9000` is forwarded to the destination router, while `socat` logs connection lifecycle and payloads.

---

## Use Case 3: UDP Send and Receive (IPv4 Explicit)

### Objective

Generate and receive UDP traffic between endpoints.

### Receiver (Cat8Kv-task-1 > container)

```bash
socat -d -d -v UDP4-LISTEN:5000,reuseaddr,fork STDOUT
```

### Sender (Cat8Kv-task-2 > container)

```bash
  echo "UDP Test From Cat8Kv-task-2 $(date)" | socat - UDP4:198.18.100.5:5000
```

### Expected Result

UDP payloads are received and displayed in real time.

---

## Use Case 4: File Transfer Over TCP (TXT, One-Way)

### Objective

Transfer a text file between containers using a raw TCP connection.

### Receiver (Cat8Kv-task-1 > container)

```bash
socat -d -d -u TCP4-LISTEN:6000,reuseaddr,fork OPEN:/tmp/received.txt,creat,trunc
```

### Sender (Cat8Kv-task-2 > container)

```bash
printf "hello-from-R1\n" > /tmp/send.txt
socat -d -d -u FILE:/tmp/send.txt TCP4:198.18.100.5:6000
```

### Verify (on receiver)

```bash
cat /tmp/received.txt
```

### Expected Result

The receiver writes the file to `/tmp/received.txt` successfully.

---

## Use Case 5: Inline Traffic Inspection Using Two Router Containers

### Objective

Inspect application traffic inline while forwarding it between two router-hosted containers.

### Lab Topology

* Cat8Kv-task-1 Container (Proxy / Inspection): `198.18.100.5`
* Cat8Kv-task-2 Container (Backend Service): `198.18.101.5`
* Lab Ubuntu (Client): `198.18.9.100`

---

### Step 1: Start Backend HTTP Server on R2 Container

On **Cat8Kv-task-2 container (`198.18.101.5`)**:

```bash
python3 -m http.server 8080 --bind 0.0.0.0
```

### Step 2: Start Inline Inspection Proxy on R1 Container

On **Cat8Kv-task-1 container (`198.18.100.5`)**:

```bash
socat -d -d -v TCP-LISTEN:8081,reuseaddr,fork TCP:198.18.101.5:8080
```

### Optional (Hex dump of payloads)

```bash
socat -d -d -v -x TCP-LISTEN:8081,reuseaddr,fork TCP:198.18.101.5:8080
```

### Step 3: Generate Traffic from Lab Ubuntu

From **Lab Ubuntu (`198.18.9.100`)**:

```bash
curl -v http://198.18.100.5:8081/
```

### Expected Result

* Ubuntu receives the HTTP response from Cat8Kv-task-2 Container via Cat8Kv-task-1 Container
* Cat8Kv-task-1 Container prints request/response payloads due to `-v` (and hex if `-x`)
* Cat8Kv-task-2 Container logs incoming requests via the python HTTP server

---

[⬅️ Return to Main Menu](../index.md)
