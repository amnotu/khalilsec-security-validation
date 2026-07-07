# TLS & Application-Layer Scanning

**Target:** https://khalilsec.no
**Original tools:** sslscan, gobuster, nikto, curl
**Final review date:** 6 July 2026

---

## TLS Configuration — sslscan

**Command:**

```bash
sslscan khalilsec.no
```

**Findings:**

| Check | Result |
|---|---|
| Legacy protocols: SSLv2, SSLv3, TLS 1.0, TLS 1.1 | Disabled |
| TLS 1.2 / TLS 1.3 | Enabled |
| Cipher suites | AEAD only: GCM and ChaCha20-Poly1305 |
| Weak ciphers | No CBC, RC4, NULL, or export suites found |
| Key exchange | X25519 / P-256 with forward secrecy |
| Heartbleed | Not vulnerable |
| Compression | Disabled |
| Renegotiation | Disabled |
| Certificate | Valid ECDSA P-256 certificate, short-lived and auto-renewed |

**Result:** No TLS weaknesses were found in the original scan. The setup
matches Caddy's modern automatic HTTPS defaults.

---

## Content Discovery — gobuster

**Command:**

```bash
gobuster dir -u https://khalilsec.no -w [SecLists wordlist] \
  -x php,js,json,env,config,bak,git,txt -t 20 --exclude-length 902
```

**Findings:** The application is a single-page app with catch-all
routing. After filtering the SPA shell baseline, sensitive-extension
paths such as `.env`, `.git`, and `.bak` returned `403 Forbidden`.

**Relevant Caddy rule:**

```caddyfile
@dotfiles {
    path_regexp \.(env|git|svn|swp|bak|old|log|map)$
}
handle @dotfiles {
    respond 403
}
```

**Assessment:** Sensitive file extension blocking is explicit at the edge.
No exposed sensitive files were found.

---

## Vulnerability Scan — nikto

**Command:**

```bash
nikto -h https://khalilsec.no
```

**Findings:** Nikto ran 6,544 checks. It reported one issue and 16
connection errors.

**Manual review:** Nikto flagged a missing `X-Frame-Options` header, but
manual `curl -I` verification showed `X-Frame-Options: SAMEORIGIN`.

**Assessment:** The Nikto item was treated as a false positive after
manual verification.

---

## Additional Checks

### HTTP to HTTPS Redirect

**Command:**

```bash
curl -I http://khalilsec.no
```

**Result:** `308 Permanent Redirect` to `https://khalilsec.no/`.

**Assessment:** No content was served over plain HTTP.

---

### CORS Origin Reflection Test

**Command:**

```bash
curl -sI -H 'Origin: https://evil.com' https://khalilsec.no
```

**Result:** `Access-Control-Allow-Origin` stayed fixed at
`https://khalilsec.no`.

**Assessment:** The server did not reflect arbitrary origins.

---

### HTTP Methods

**Command:**

```bash
curl -X OPTIONS -i https://khalilsec.no
```

**Result:** `204 No Content`; allowed methods were limited to `GET`,
`POST`, and `OPTIONS`. `CONNECT` and `TRACE` were explicitly blocked with
`405 Method Not Allowed`.

**Assessment:** No unexpected write or tunnel methods were exposed.

---

### Cookie Security

No cookies were set on a standard `GET` request.

**Assessment:** Cookie flags were not applicable at the time of testing.
Revisit this if session or auth features are added.

---

### robots.txt and security.txt

Both files fell through to the SPA shell during the original check.

**Assessment:** This is not a vulnerability. Adding a `security.txt` file
with a disclosure contact is still recommended for a security-focused site.

---

### HTTP/2 and HTTP/3

**Command:**

```bash
curl -I --http2 https://khalilsec.no
```

**Result:** HTTP/2 negotiated successfully. The server also advertised
HTTP/3 through `alt-svc`. HTTP/3 was not independently tested because the
local curl build did not support it.

---

## Reverse Proxy Configuration Review

The Caddyfile for `khalilsec.no` was reviewed directly.

```caddyfile
@dotfiles {
    path_regexp \.(env|git|svn|swp|bak|old|log|map)$
}
handle @dotfiles {
    respond 403
}

@connect {
    method CONNECT
}
handle @connect {
    respond 405
}

@trace {
    method TRACE
}
handle @trace {
    respond 405
}
```

**Assessment:** The observed hardening is deliberate. Sensitive extension
blocking, method denial, and server header stripping are all handled at
the reverse proxy layer.

---

## Final Status

No exploitable TLS, CORS, file exposure, or method tampering issue was
confirmed after the hardening changes.
