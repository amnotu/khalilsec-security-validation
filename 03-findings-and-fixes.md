# Findings & Fixes

This document summarizes the issues found during testing, the fixes made,
and the checks used to confirm each fix.

**Target:** https://khalilsec.no
**Final review date:** 6 July 2026

---

## Summary Table

| # | Finding | Severity | Status |
|---|---|---|---|
| 1 | Fabricated compliance-claim headers | Informational / Credibility | Fixed |
| 2 | Duplicate and conflicting Content-Security-Policy | Low | Fixed |
| 3 | Overly permissive CORS logic in the application layer | Low | Fixed |
| 4 | Unknown Host headers returned `200` | Low | Fixed |

---

## Finding 1: Fabricated Compliance-Claim Headers

**What I found:** The application set custom HTTP headers such as
`X-GDPR-Protocol`, `X-HIPAA-Compliance`, `X-PCI-DSS-Compliance`,
`X-SOC2-Audit`, and `X-ISMS-Audit`. These suggested formal regulatory or
audit compliance.

**Why it matters:** These are not standard security headers. The site had
not gone through those audits. For a security portfolio, inaccurate audit
claims hurt trust.

**Fix:** Removed all five headers from the Express middleware.

**Verification:**

```bash
curl -sI https://khalilsec.no | grep -i 'gdpr\|hipaa\|isms\|pci-dss\|soc2'
```

**Expected result:** No output.

**Status:** Fixed.

---

## Finding 2: Duplicate and Conflicting Content-Security-Policy

**What I found:** Both Caddy and Express set a `Content-Security-Policy`
header. Caddy had the stricter policy. Express had a weaker policy with
`'unsafe-inline'`, `'unsafe-eval'`, and broad `https://*` sources.

**Why it matters:** Two CSP sources are fragile. If the proxy changes or
the app runs without Caddy, the weaker app policy could become active.

**Fix:** Removed the CSP header from Express. Caddy is now the single
source of truth for CSP.

**Verification:**

```bash
curl -sI https://khalilsec.no | grep -i content-security-policy
```

**Expected result:** Exactly one CSP header, matching the strict proxy
policy.

**Status:** Fixed.

---

## Finding 3: Overly Permissive CORS Logic in the Application Layer

**What I found:** Express CORS logic used substring and suffix checks such
as `origin.includes("europe-west")` and `origin.endsWith("run.app")`. It
also had a wildcard fallback for requests without an `Origin` header.

**Why it matters:** Loose matching can allow origins the developer did not
intend to trust. Caddy masked this in production, but the app layer was
still wrong on its own.

**Fix:** Replaced loose matching with an exact-match allowlist. Removed
the wildcard fallback.

**Verification:**

```bash
curl -sI -H 'Origin: https://evil.com' https://khalilsec.no | grep -i access-control-allow-origin
curl -sI -H 'Origin: https://khalilsec.no' https://khalilsec.no | grep -i access-control-allow-origin
```

**Expected result:** Both checks return `https://khalilsec.no`. The proxy
policy stays fixed and does not reflect attacker-controlled origins.

**Status:** Fixed.

---

## Finding 4: Unknown Host Headers Returned `200`

**What I found:** Requests with arbitrary `Host` headers returned
`HTTP/2 200` with an empty body.

**Why it matters:** No sensitive content was exposed, but unknown hosts
should not appear accepted. This can create confusion and may help abuse
cases such as domain-fronting attempts.

**Fix:** Added Caddy catch-all handlers that return `421 Misdirected
Request` for hosts not matching `khalilsec.no` or `www.khalilsec.no`.

**Verification:**

```bash
curl -sI -H 'Host: admin.khalilsec.no' https://khalilsec.no/ | head -1
curl -sI -H 'Host: evil-attacker.com' https://khalilsec.no/ | head -1
curl -sI -H 'Host: khalilsec.no' https://khalilsec.no/ | head -1
```

**Expected result:** Unknown hosts return `421`. The valid host returns
`200`.

**Status:** Fixed.

---

## Additional Testing: No Vulnerabilities Found

### Custom Rate Limiter

**Command:**

```bash
for i in {1..35}; do
  curl -s -o /dev/null -w "%{http_code}
"     -X POST https://khalilsec.no/api/password-check     -H 'Content-Type: application/json'     -d '{"password":"test123"}'
done
```

**Result:** The first 30 requests returned `200`. Requests 31 through 35
returned `429 Too Many Requests`.

**Conclusion:** The limiter matched the configured 30-request, 60-second
window.

---

### DNS Configuration

**Commands:**

```bash
dig +short thisdoesnotexist12345.khalilsec.no
dig +short api.khalilsec.no
subfinder -d khalilsec.no
```

**Result:** No wildcard DNS record was found. Passive enumeration found
only `www.khalilsec.no` in addition to the root domain.

**Conclusion:** No forgotten, undocumented, or wildcard-exposed subdomain
was found.

---

### API Input Validation

Malformed, oversized, wrong-type, and boundary-case payloads were sent to
`/api/breach-check`, `/api/recon`, and `/api/chat`.

| Endpoint | Test case | Result |
|---|---|---|
| `/api/breach-check` | Oversized email over 256 chars | `400` rejected |
| `/api/breach-check` | Malformed email without `@` | `400` rejected |
| `/api/breach-check` | Non-string type | `400` rejected |
| `/api/breach-check` | Missing field | `400` rejected |
| `/api/recon` | URL instead of bare domain | `400` rejected |
| `/api/recon` | Path traversal attempt | `400` rejected |
| `/api/recon` | Oversized domain over 256 chars | `400` rejected |
| `/api/recon` | Empty string | `400` rejected |
| `/api/chat` | Message array over 40 items | `400` rejected |
| `/api/chat` | Single message over 3000 chars | `400` rejected |
| `/api/chat` | Wrong type for `messages` | `400` rejected |

**Conclusion:** Input validation rejected all malformed and oversized
payloads tested.

**Positive observation:** `/api/breach-check` and `/api/recon` returned
demo data with a clear `"status":"DEMO_FALLBACK"` label. The site did
not present simulated data as real findings.

---

## Final Status

All documented issues are fixed. Follow-up work should focus on adding a
`security.txt` disclosure contact and rerunning the full command-line test
set after each deployment.
