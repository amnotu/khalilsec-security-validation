# Pentest Execution Log — Initial Baseline (khalilsec.no)

This document records the first round of testing performed before the
Caddy-layer hardening described in `02-web-scan.md` and
`03-findings-and-fixes.md`. It is preserved to show the path from the
initial state to the hardened state.

**Original test date:** 16 June 2026
**Final review date:** 6 July 2026
**Tester:** MHD Khalil Arabieh
**Target:** https://khalilsec.no (Local Compute Engine VM)

---

## 1. Reconnaissance

### 1.1 Port Scanning (nmap)

**Command:**

```bash
nmap -sV -sC -p 80,443 --script='http-title,http-headers,http-methods,http-server-header' 136.118.89.39
```

**Output excerpt:**

```text
PORT    STATE SERVICE  VERSION
80/tcp  open  http     Caddy httpd
| http-headers:
|   Location: https://khalilsec.no/
|   Server: Caddy
443/tcp open  ssl/https
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
```

**Finding:** Caddy was detected on ports 80 and 443. The scan reported
only standard methods: `GET`, `HEAD`, `POST`, and `OPTIONS`.

---

### 1.2 Directory Enumeration (gobuster)

**Commands:**

```bash
gobuster dir -u https://khalilsec.no -w /usr/share/dirb/wordlists/common.txt -x js,json,html,txt -k --exclude-length 868

gobuster dir -u https://khalilsec.no -w /usr/share/dirb/wordlists/common.txt -x php,asp,aspx,bak,old,txt,json,env,git,svn,swp,conf,yaml,yml -k --exclude-length 868 -t 50
```

**Output:**

```text
/assets               (Status: 301) [Size: 156] [--> /assets/]
```

**Finding:** Only `/assets/` was found during this baseline run. Other
paths returned the SPA `index.html` shell, including probes for `.env`,
`.git`, backup files, and similar paths.

At this point, the fallback behavior came from the SPA catch-all route.
There was not yet an explicit Caddy block for sensitive extensions. This
was later hardened with a `403 Forbidden` rule. See `02-web-scan.md`.

---

## 2. Vulnerability Assessment

### 2.1 Parameter Fuzzing (ffuf)

**Command:**

```bash
ffuf -u 'https://khalilsec.no/?FUZZ=test' -w /snap/seclists/1214/Discovery/Web-Content/burp-parameter-names.txt -fs 868 -t 50
```

**Output summary:**

```text
:: Progress: [6453/6453] :: All responses had size 868
```

**Finding:** All 6,453 tested parameters returned the same `index.html`
response. The application did not process arbitrary query parameters at
this layer. No reflected XSS or open redirect behavior was found through
this test.

---

### 2.2 HTTP Method Tampering (Before Fix)

**Commands:**

```bash
curl -X TRACE https://khalilsec.no -I
curl -X CONNECT https://khalilsec.no -I
```

**Output before fix:**

```text
HTTP/2 200
```

**Finding at the time:** `TRACE` and `CONNECT` returned `200` instead of
being denied. `TRACE` has historic relevance to Cross-Site Tracing. No
practical exploit was found in this setup, but allowing these methods was
unneeded attack surface.

**Resolution:** Caddy handlers were added to return `405 Method Not
Allowed` for both `TRACE` and `CONNECT`. See the HTTP method checks in
`02-web-scan.md`.

---

### 2.3 Sensitive File Exposure

**Commands:**

```bash
curl https://khalilsec.no/.env
curl 'https://khalilsec.no/static/../server.js'
```

**Output before fix:** Both returned the SPA `index.html` shell.

**Finding at the time:** This was catch-all behavior, not proof that
`.env` or `server.js` existed at those paths. The path traversal attempt
using `../` did not expose files. Sensitive extensions were later blocked
at the Caddy layer.

---

### 2.4 Host Header Spoofing

**Original test commands:**

```bash
curl -sI -H 'Host: admin.khalilsec.no' https://khalilsec.no/
curl -sI -H 'Host: evil-attacker.com' https://khalilsec.no/
```

**Output before fix:**

```text
HTTP/2 200
content-length: 0
```

**Finding at the time:** Unknown hostnames returned `200` with an empty
body. No content leakage, virtual host routing issue, or reflected host
injection was found. The behavior still made unrecognized hosts appear
accepted by the server.

**Risk:** A `200` response for arbitrary hosts can create confusion and
can help domain-fronting style abuse. HTTP/2 guidance also favors `421
Misdirected Request` for requests sent to the wrong origin.

**Remediation:** Catch-all Caddy blocks were added for HTTP and HTTPS:

```caddyfile
http:// {
    respond "421 Misdirected Request" 421
}

https:// {
    respond "421 Misdirected Request" 421
}
```

**Verification after fix:**

```bash
curl -sI -H 'Host: admin.khalilsec.no' https://khalilsec.no/ | head -1
curl -sI -H 'Host: evil-attacker.com' https://khalilsec.no/ | head -1
curl -sI -H 'Host: definitely-not-valid-12345.com' https://khalilsec.no/ | head -1
curl -sI -H 'Host: khalilsec.no' https://khalilsec.no/ | head -1
curl -sI -H 'Host: admin.khalilsec.no' http://khalilsec.no/ | head -1
```

**Expected output after fix:**

```text
HTTP/2 421
HTTP/2 421
HTTP/2 421
HTTP/2 200
HTTP/1.1 421 Misdirected Request
```

**Result:** Unrecognized hosts now return `421 Misdirected Request`.
The valid host continues to return `200`. The issue is resolved.

---

## 3. Client-Side Input Handling

### 3.1 Terminal Command Injection / XSS Test

The site includes an interactive terminal-style UI component. It accepts
free-text commands and matches them against a fixed set of supported
commands such as `help`, `about`, `skills`, and `cmatrix`.

**Payloads entered in the terminal input:**

```text
alert("XSS")
"; alert("XSS"); //
```

**Result:**

```text
Command not found: "alert("XSS")". Type "help" to fetch list of diagnostic instructions.
Command not found: ""; alert("XSS"); //". Type "help" to fetch list of diagnostic instructions.
```

**Finding:** No script execution occurred. Review of the component source
explains why: unmatched input is rendered as plain text through React JSX
(`{line.text}`). React escapes this output by default. The tested flow did
not use `innerHTML`, `dangerouslySetInnerHTML`, or `eval()`.

**Result:** No terminal command injection or client-side XSS was found in
this flow.
