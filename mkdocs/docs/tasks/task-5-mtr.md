# Task 5: MTR Path Analysis (My Traceroute)

[⬅ Back to Main Menu](../index.md)

---

## Objective

In this task, you will use **MTR (My Traceroute)** from the **Swiss-Knife container** to analyze:

- Network path visibility
- Packet loss behavior
- Latency trends across the path

This task focuses on **real troubleshooting workflows**, not command syntax memorization.

---

## Page Index

- [Step 1: Baseline Path Test (ICMP)](#step-1-baseline-path-test-icmp)
- [Step 2: UDP-Based Path Test](#step-2-udp-based-path-test)
- [Step 3: TCP-Based Path Test (Most Realistic)](#step-3-tcp-based-path-test-most-realistic)
- [Step 4: MTR Using Hostnames](#step-4-mtr-using-hostnames)
- [How to Interpret MTR Output](#how-to-interpret-mtr-output)
- [Loss Interpretation Cheat Sheet](#loss-interpretation-cheat-sheet)
- [Key Takeaways](#key-takeaways)

---

## Step 1: Baseline Path Test (ICMP)

Connect to **Cat8Kv-Task-1** and access the Swiss-Knife container:

```bash
ssh 198.18.1.11
app-hosting connect appid swiss_knife session /bin/bash
```

Run a baseline ICMP MTR to a public destination:

```bash
mtr -r -n -c 20 8.8.8.8
```

**Why this matters**

* Establishes basic reachability
* Shows hop-by-hop latency
* Acts as a baseline before deeper analysis

### MTR – Commonly Used Options

| Option | Meaning | Why it’s used |
|------|--------|--------------|
| `-n` | Numeric output (no DNS lookup) | Faster results, avoids DNS delays |
| `-r` | Report mode (no interactive UI) | Script-friendly, clean output |
| `-c <count>` | Number of probes to send | Control test duration |
| `-w` | Wide report format | Prevents column wrapping, improves readability |
| `-T` | Use TCP probes | Test application-like paths |
| `-u` | Use UDP probes | Test non-ICMP traffic behavior |

---

## Step 2: UDP-Based Path Test

Run MTR using UDP to simulate application-like traffic.

### DNS-like traffic example

```bash
mtr -r -n -u -P 53 8.8.8.8
```

### Voice / video / SLA-style traffic

```bash
mtr -r -n -u -P 16384 8.8.4.4
```

**When to use UDP MTR**

* SD-WAN SLA validation
* Voice / video path testing
* Comparing ICMP vs policy-based paths

> Note: UDP MTR does not send real application payloads.
> Some destinations may not respond — this is expected.

---

## Step 3: TCP-Based Path Test (Most Realistic)

TCP MTR is the **most reliable** method in enterprise and cloud networks.

### HTTPS path test

```bash
mtr -r -n -T -P 3000 198.18.5.101
```

### Alternate public target

```bash
mtr -r -n -T -P 443 208.67.220.220
```

**Why TCP MTR is preferred**

* Traverses firewalls and NAT
* Matches real application behavior
* Aligns with SD-WAN and security policies

---

## Step 4: MTR Using Hostnames

You can also run MTR directly to hostnames:

```bash
mtr -r -T -P 443 www.cisco.com
```

**What happens**

* DNS is resolved once
* MTR traces to the resolved IP
* Useful for SaaS and cloud troubleshooting

**Best practice**

```bash
dig www.google.com
mtr -r -n -T -P 443 <resolved-ip>
```

---

## How to Interpret MTR Output

### Sample Output (Reference)

```text
HOST: swissknife                  Loss%   Snt   Last   Avg  Best  Wrst StDev
  1.|-- 192.168.100.1              0.0%    20    0.4   0.4   0.3   0.6   0.1
  2.|-- 198.18.6.10                0.0%    20    0.7   0.7   0.6   1.1   0.1
  3.|-- 198.18.1.1                 0.0%    20    1.4   1.4   1.2   1.5   0.1
  4.|-- 10.255.0.3                 0.0%    20    1.9   1.6   1.4   2.1   0.2
  5.|-- ???                       100.0    20    0.0   0.0   0.0   0.0   0.0
  6.|-- 8.8.8.8                    0.0%    20    7.5   7.7   6.6  15.1   2.6
```

### Key interpretation rules

* **Destination matters most**

  * If destination shows 0% loss → path is healthy
* **`???` hops are normal**

  * ICMP is often filtered or rate-limited
* **Latency trends matter**

  * Gradual increase is normal
  * Sudden jump that continues indicates an issue

---

## Loss Interpretation Cheat Sheet

| Scenario                        | Meaning         |
| ------------------------------- | --------------- |
| Loss at one hop only            | Ignore          |
| Loss continues to destination   | Real problem    |
| `???` but destination reachable | Normal behavior |
| ICMP fails, TCP works           | ICMP filtered   |
| UDP fails, TCP works            | Expected        |

---

## Key Takeaways

* MTR provides **path visibility**, not application testing
* TCP MTR is the **most reliable in real networks**
* Ignore loss unless it reaches the destination
* `???` hops are expected in SP and cloud environments

---

[⬅ Return to Main Menu](../index.md)

---
