# Task 6: Web Testing & Troubleshooting (curl, wget, httpie)

[⬅️ Back to Main Menu](../index.md)

## Table of Contents

- [1) Basic Connectivity Test (GET)](#1-basic-connectivity-test-get)
- [2) Headers Only (`-I`) — “Is the web service up?”](#2-headers-only-i-is-the-web-service-up)
- [3) Verbose Mode (`-v`) — “Show me *what* fails and *where*”](#3-verbose-mode-v-show-me-what-fails-and-where)
- [4) Force a Specific IP While Keeping the Hostname (`--resolve`)](#4-force-a-specific-ip-while-keeping-the-hostname-resolve)
- [5) Test a Specific Port](#5-test-a-specific-port)
- [6) About `-k` (Insecure TLS)](#6-about-k-insecure-tls)
- [7) Application Timing (Great for “slow app” complaints)](#7-application-timing-great-for-slow-app-complaints)
- [8) API Calls](#8-api-calls)
- [9) Recommended “Quick Checks” (Copy/Paste)](#9-recommended-quick-checks-copypaste)
- [10) `wget` — File Download + Availability Checks](#10-wget-file-download-availability-checks)
- [11) HTTPie – API Calls](#11-httpie-api-calls)
- [12) Comparing curl, wget, and httpie](#12-comparing-curl-wget-and-httpie)
- [13) Tool Selection Cheat Sheet](#13-tool-selection-cheat-sheet)

---

## 1) Basic Connectivity Test (GET)
* Login to the Cat8Kv-Task-1 (ssh 198.18.1.11)
* Connect to the swiss_knife container
```bash
app-hosting connect appid swiss_knife session /bin/bash
```

```bash
curl https://www.google.com
```

### What this tells you

* **DNS** works (hostname resolved)
* **TCP/443** is reachable
* **TLS handshake** completes
* **HTTP request/response** succeeds

> Tip: This prints the full HTML body (very noisy). Use `-I` or `-v` for troubleshooting.

---

## 2) Headers Only (`-I`) — “Is the web service up?”

```bash
curl -I https://www.google.com
```

### What `-I` does

* Sends an HTTP **HEAD** request (or equivalent behavior depending on server/CDN)
* Returns **response headers only** (no HTML body)
* Fast and clean for reachability checks

### How to read the important headers 

* `HTTP/2 200` → request succeeded (HTTP status code **200 OK**)
* `date:` → server time when response was generated
* `server: gws` → Google Web Server (useful to confirm which edge served you)
* `content-type:` → response format (HTML / JSON / etc.)
* `cache-control:` / `expires:` → caching behavior (important when troubleshooting “stale” pages)
* `set-cookie:` → confirms the response was processed end-to-end (common for web apps)
* `alt-svc: h3=":443"` → server advertises HTTP/3 (QUIC) availability on 443

* Sample Output
```text
swissknife:/root# curl -I https://www.google.com                                        
HTTP/2 200 
content-type: text/html; charset=ISO-8859-1
content-security-policy-report-only: object-src 'none';base-uri 'self';script-src 'nonce-xhH9ePIU8GyadCXkRymmXg' 'strict-dynamic' 'report-sample' 'unsafe-eval' 'unsafe-inline' https: http:;report-uri https://csp.withgoogle.com/csp/gws/other-hp
accept-ch: Sec-CH-Prefers-Color-Scheme
p3p: CP="This is not a P3P policy! See g.co/p3phelp for more info."
date: Sat, 17 Jan 2026 09:13:32 GMT
server: gws
x-xss-protection: 0
x-frame-options: SAMEORIGIN
expires: Sat, 17 Jan 2026 09:13:32 GMT
cache-control: private
set-cookie: AEC=AaJma5vSZ67DfW3zRxjvMo2OPJBX8M7gxDpdqY1bohAMbgk5hZ_6wZ6Fz90; expires=Thu, 16-Jul-2026 09:13:32 GMT; path=/; domain=.google.com; Secure; HttpOnly; SameSite=lax
set-cookie: __Secure-ENID=30.SE=FpIEnE9GiX6iQ5M1s8hTb4nGNsb_lEmS6nR85Ju0c1Ben9O29_OynUu0xIKR_uVSqwiH2HfSNCzmffZUH24JHakx5NgKhyKbPAj6c4WZZlpnbozXYGNv1jBVq2R2HK9XEVCTOaaaZmePf45ClvoHG50Fmt24WoorGqNm-bmCjeWvQeIT1zsQznzGH0NucilhsFz2f2irKyx6E4ncRhQQb5l1eg; expires=Wed, 17-Feb-2027 01:31:50 GMT; path=/; domain=.google.com; Secure; HttpOnly; SameSite=lax
set-cookie: __Secure-BUCKET=CBI; expires=Thu, 16-Jul-2026 09:13:32 GMT; path=/; domain=.google.com; Secure; HttpOnly
alt-svc: h3=":443"; ma=2592000,h3-29=":443"; ma=2592000

swissknife:/root#
```

---

## 3) Verbose Mode (`-v`) — “Show me *what* fails and *where*”

```bash
curl -v https://www.google.com
```

### What `-v` adds 

Verbose mode prints the full transaction steps, which is exactly what you need when “it doesn’t work”:

* DNS resolution results (A/AAAA records)
* Which address curl tries first (**IPv6 then IPv4**)
* TCP connect attempts and failures
* TLS handshake details (TLS version, cipher, cert subject/issuer, validity)
* ALPN negotiation (HTTP/2 vs HTTP/1.1)
* Request headers sent + response headers received

**Note:**
Curl tries IPv6 first and failed with `Network unreachable`, then fell back to IPv4 and succeeded.
That’s a classic “no IPv6 routing” symptom in lab/enterprise networks.

---

## 4) Force a Specific IP While Keeping the Hostname (`--resolve`)

```bash
curl -v https://8.8.8.8 --resolve www.google.com:443:8.8.8.8
```

### What this is doing

* `--resolve` injects a **static DNS override** inside curl:

  * “When connecting to `www.google.com:443`, use IP `8.8.8.8`”
* This is extremely useful to isolate:

  * **DNS problems** (bypass DNS completely)
  * **Anycast / CDN behavior** (force a specific edge)
  * **Policy / routing differences** to a specific destination IP

> Lab use case: “DNS is broken” vs “path is blocked to the service”
>
> * If `--resolve` works but normal name doesn’t → DNS issue
> * If both fail → network / firewall / routing issue

---

## 5) Test a Specific Port

### 5.1 Internal service example (Non 80 or 443 port)

```bash
curl -v http://198.18.102.5:8080
```

This is a perfect lab example of:

* Port reachability
* HTTP server identity (`Server: lighttpd/1.4.82`)
* Valid HTTP status (`200 OK`)

* Use your **internal lab web app** for 8080/8443 tests

---

## 6) About `-k` (Insecure TLS)

You used:

```bash
curl -vk https://example.com:8443
```

### What `-k` does

* Skips TLS certificate verification (curl continues even if the CA chain is incomplete)

### When to use it

* Internal services using:

  * self-signed certs
  * private CA
  * incomplete chain
* Quick validation: “Is TLS reachable at all?”

### When *not* to use it

* Production validation (it can hide real cert problems)

> In your output, curl showed:
> `SSL certificate verify result: unable to get local issuer certificate (20), continuing anyway.`
> That’s exactly what `-k` is for in a lab.

---

## 7) Application Timing (Great for “slow app” complaints)

Command you ran:

```bash
curl -o /dev/null -s -w \
"DNS: %{time_namelookup}\nConnect: %{time_connect}\nTLS: %{time_appconnect}\nTTFB: %{time_starttransfer}\nTotal: %{time_total}\n" \
https://www.google.com
```

Example output :

```text
DNS: 0.011500
Connect: 0.019673
TLS: 0.045948
TTFB: 0.087206
Total: 0.088797
```

### How to interpret these numbers

* `DNS` → time to resolve the hostname
* `Connect` → time to establish TCP connection
* `TLS` → time to complete TLS handshake (HTTPS only)
* `TTFB` → **Time To First Byte** (server responsiveness + any middleboxes)
* `Total` → full request time (until first response completes)

**Troubleshooting clues**

* High **DNS** → resolver issue / reachability / latency to DNS
* High **Connect** → routing / firewall / packet loss / congestion
* High **TLS** → TLS inspection / slow handshake / CPU constraints
* High **TTFB** with low Connect/TLS → server/backend slowness

---

## 8) API Calls 

### 8.1 Simple GET that returns JSON

```bash
curl -s https://httpbin.org/get | head
```

### 8.2 GET with query parameters

```bash
curl -s "https://httpbin.org/get?site=swissknife&tool=curl"
```

### 8.3 POST JSON body

```bash
curl -s -X POST https://httpbin.org/post \
  -H "Content-Type: application/json" \
  -d '{"user":"test","pass":"test"}' | head
```
From the output notice that, **httpbin is echoing back exactly what it received**:

* The **`data`** field shows the raw request body (`{"user":"test","pass":"test"}`), confirming the POST payload arrived intact.
* The **`headers`** section reflects your request headers (for example `Content-Type: application/json`), confirming the request was processed as sent.

---

## 9) Recommended “Quick Checks” (Copy/Paste)

```bash
# 1) Is HTTPS reachable (headers only)?
curl -I https://www.google.com

# 2) Where does it fail (DNS / TCP / TLS / HTTP)?
curl -v https://www.google.com

# 3) Is it slow because of DNS, connect, TLS, or server?
curl -o /dev/null -s -w "DNS:%{time_namelookup}\nConnect:%{time_connect}\nTLS:%{time_appconnect}\nTTFB:%{time_starttransfer}\nTotal:%{time_total}\n" https://www.google.com

# 4) Is this a DNS problem or a routing/firewall problem?
curl -v https://1.1.1.1 --resolve example.com:443:1.1.1.1
```

---

## 10) `wget` — File Download + Availability Checks

`wget` is great when you want to **download files**, **validate reachability**, or **test stability** (resume support).

---

### 10.1 Basic download

```bash
wget --no-check-certificate https://hel1-speed.hetzner.com/100MB.bin
```

What to look for:

* DNS resolution + connect success
* Steady download progress
* File saved locally (good for throughput / path validation)

---

### 10.2 Headers / existence check only - `--spider`

```bash
wget --spider https://www.google.com
```

What it does :

* **Does NOT download** the page/file
* Sends an HTTP request and checks the response
* You got `200 OK` + “Remote file exists…” → the URL is reachable and responding

Use case:

* Quick “is it up?” check without pulling content (great in automation and troubleshooting)

---

### 10.3 Resume a download — `-c`

```bash
wget -c wget https://releases.ubuntu.com/22.04/ubuntu-22.04.5-live-server-amd64.iso
```

What `-c` does:

* If the download is interrupted, rerunning the same command **continues from where it stopped**
* Useful for:

  * unstable links
  * failover testing
  * validating SD-WAN / routing policy changes during transfer

---

### 10.4 Force IPv4 vs IPv6 — `-4` / `-6`

IPv4 forced :

```bash
wget -4 https://www.google.com
```

IPv6 forced :

```bash
wget -6 https://www.google.com
```

What this tells you:

* `-4` forces `wget` to use **A records (IPv4)** only
* `-6` forces `wget` to use **AAAA records (IPv6)** only
* Your `-6` failure `Network unreachable` confirms **no IPv6 routing** from that container/network, even though DNS can resolve IPv6 addresses

Use case:

* Quickly prove whether an issue is **IPv6 reachability** vs application/DNS
* Useful when apps behave differently over v4/v6, or when dual-stack policy is involved

---

## 11) HTTPie – API Calls

* HTTPie does the same job as curl.
* It presents the request and response in a more human-readable format (JSON-first, cleaner defaults).
---

### 11.1 Basic GET 

Use `--verify=no` : The container **doesn’t have the full CA chain installed**

```bash
http --verify=no https://httpbin.org/get
```

**What you’ll see echoed back**

* Your source IP/headers (in JSON)
* Any query params you pass
* Request metadata (helps validate what the app “sees”)

---

### 11.2 Headers only

```bash
http --verify=no -h https://httpbin.org/get
```

**Use case**

* Confirm **HTTP status**, `Content-Type`, and server headers without printing the body.

---

### 11.3 Verbose

```bash
http --verify=no --verbose https://httpbin.org/get
```
* HTTPie prints the request headers first (),
* It attempts the TLS connection and fails unless --verify=no is used.

---

### 11.4 POST JSON 

```bash
http --verify=no POST https://httpbin.org/post user=test pass=test
```

**Verification**

* httpbin will echo your payload back in the response as JSON (similar idea to curl’s `data/json` echo).

---
## 12) Comparing curl, wget, and httpie

| Tool   | Best For                          |
| ------ | --------------------------------- |
| curl   | Deep troubleshooting, timing, TLS |
| wget   | File downloads, availability      |
| httpie | APIs, JSON, readability           |

---
## 13) Tool Selection Cheat Sheet

| Question                   | Tool   |
| -------------------------- | ------ |
| Is host reachable?         | ping   |
| What path is used?         | mtr    |
| Is app reachable?          | curl   |
| Is DNS working?            | dig    |
| Is file accessible?        | wget   |
| Is API behaving correctly? | httpie |

---

[⬅️ Return to Main Menu](../index.md)
