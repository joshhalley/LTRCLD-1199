# Task 5: MTR Path Analysis (My Traceroute)

[⬅️ Back to Main Menu](README.md)

## Table of Contents

- [1. Sample Output (For Reference)](#1-sample-output-for-reference)
- [2. How to Interpret This Output](#2-how-to-interpret-this-output)
- [3. Recommended Baseline Command](#3-recommended-baseline-command)
- [4. ICMP MTR (Default Mode)](#4-icmp-mtr-default-mode)
- [5. UDP MTR (Application-like Traffic)](#5-udp-mtr-application-like-traffic)
- [6. TCP MTR (Most Reliable for Apps)](#6-tcp-mtr-most-reliable-for-apps)
- [7. MTR Using URLs (Hostname Resolution)](#7-mtr-using-urls-hostname-resolution)
- [8. Loss Interpretation Cheat Sheet](#8-loss-interpretation-cheat-sheet)
- [9. Tool Selection Guidance](#9-tool-selection-guidance)
- [10. Key Takeaways](#10-key-takeaways)

---

This guide explains how to use **MTR (My Traceroute)** effectively in **modern, policy-driven networks** (SD-WAN, firewalls, cloud, app-hosting).

MTR combines:

* `ping` (loss, latency, jitter)
* `traceroute` (hop-by-hop path)

---

## 1. Sample Output (For Reference)

Example command:

```bash
mtr -r -n -c 10 8.8.8.8
```

Example output:

```text
HOST: swissknife                  Loss%   Snt   Last   Avg  Best  Wrst StDev
  1.|-- 192.168.100.1              0.0%    10    0.4   0.4   0.3   0.6   0.1
  2.|-- 198.18.6.10                0.0%    10    0.7   0.7   0.6   1.1   0.1
  3.|-- 198.18.1.1                 0.0%    10    1.4   1.4   1.2   1.5   0.1
  4.|-- 10.255.0.3                 0.0%    10    1.9   1.6   1.4   2.1   0.2
  5.|-- 10.1.27.9                  0.0%    10    1.9   1.9   1.6   2.3   0.2
  6.|-- ???                       100.0    10    0.0   0.0   0.0   0.0   0.0
  7.|-- 64.103.43.33               0.0%    10    2.1   2.0   1.9   2.3   0.1
  8.|-- 10.230.4.141               0.0%    10    6.0   6.1   5.9   6.2   0.1
  9.|-- 64.103.40.101              0.0%    10    6.4   7.2   6.3   8.6   0.9
 10.|-- 128.107.8.18               0.0%    10   27.2  27.6  26.9  28.1   0.5
 11.|-- ???                       100.0    10    0.0   0.0   0.0   0.0   0.0
 12.|-- 8.8.8.8                    0.0%    10    7.5   7.7   6.6  15.1   2.6
```

---

## 2. How to Interpret This Output

### Hop classification (important)

| Hop range | Meaning                             |
| --------- | ----------------------------------- |
| 1–3       | **Local / Container / Router edge** |
| 4–6       | **Underlay / SD-WAN / Core**        |
| 7–10      | **Service Provider / Backbone**     |
| 11        | **Google edge / cloud border**      |
| 12        | **Destination**                     |

---

### Key interpretation rules

#### Rule 1 – Destination matters most

* If **destination has 0% loss**, the path is healthy
* Ignore loss on intermediate hops unless it continues

#### Rule 2 – `???` is normal

* Routers are **not required** to respond to ICMP
* Firewalls, SP cores, and cloud edges often suppress replies

#### Rule 3 – Latency should increase gradually

* Sudden jumps that persist to destination indicate problems
* One noisy hop alone is not an issue

---

## 3. Recommended Baseline Command
* Login to the Cat8Kv-Task-1 (ssh 198.18.1.11)
* Connect to the swiss_knife container
```bash
app-hosting connect appid swiss_knife session /bin/bash
```

```bash
mtr -r -n -w -c 20 <destination>
```

Why:

* `-r` → report mode (non-interactive, container-safe)
* `-n` → numeric IPs (no DNS noise)
* `-w` → clean formatting
* `-c 20` → better statistics

---

## 4. ICMP MTR (Default Mode)

```bash
mtr -r -n 8.8.8.8
```

### Use cases

* Basic reachability
* Initial path discovery
* Quick health check

### Limitations

* ICMP is often:

  * Rate-limited
  * De-prioritized
  * Blocked

---

## 5. UDP MTR (Application-like Traffic)

### Syntax

```bash
mtr -r -n -u -P <port> <destination>
```

### Examples

#### DNS-like path (not DNS validation)

```bash
mtr -r -n -u -P 53 8.8.8.8
```

#### Voice / video / SD-WAN SLA

```bash
mtr -r -n -u -P 16384 <remote-ip>
```

### When to use UDP

* Voice / video analysis
* SD-WAN policy paths
* SLA validation

### Important warning

> UDP MTR does **not** send real application payloads
> Some destinations (e.g. DNS servers) will **not respond**

---

## 6. TCP MTR (Most Reliable for Apps)

### Syntax

```bash
mtr -r -n -T -P <port> <destination>
```

### Examples

#### HTTPS

```bash
mtr -r -n -T -P 443 8.8.8.8
```

### Why TCP MTR is powerful

* Mimics real application behavior
* Traverses:

  * Firewalls
  * NAT
  * SD-WAN policies
* Most reliable in enterprise networks

---

## 7. MTR Using URLs (Hostname Resolution)

```bash
mtr -r -T -P 443 www.google.com
```

What happens:

* DNS resolution happens **once**
* MTR traces to resolved IP
* Useful for:

  * SaaS apps
  * Cloud endpoints

### Best practice

```bash
mtr -r -n -T -P 443 <resolved-ip>
```

(Use `dig` first if DNS itself is under suspicion.)

---

## 8. Loss Interpretation Cheat Sheet

| Scenario                           | Meaning                   |
| ---------------------------------- | ------------------------- |
| Loss at hop N only                 | Ignore                    |
| Loss starts at hop N and continues | Real issue                |
| `???` but destination reachable    | Normal                    |
| ICMP fails, TCP works              | ICMP filtered             |
| UDP fails, TCP works               | Expected in many networks |

---

## 9. Tool Selection Guidance

| Goal           | Tool     |
| -------------- | -------- |
| Reachability   | `ping`   |
| Path + loss    | `mtr`    |
| App path       | `mtr -T` |
| DNS validation | `dig`    |
| HTTP testing   | `curl`   |

---

## 10. Key Takeaways 

* **MTR is a path visibility tool, not an application tester**
* **TCP MTR is the most reliable in real networks**
* **Loss only matters if it reaches the destination**
* **`???` is expected in SP and cloud environments**

---

[⬅️ Return to Main Menu](README.md)
