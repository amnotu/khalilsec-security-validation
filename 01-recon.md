# Recon & Fingerprinting

**Target:** https://khalilsec.no
**Original tools:** whatweb, curl
**Final review date:** 6 July 2026

---

## Commands

```bash
whatweb https://khalilsec.no
curl -sI https://khalilsec.no
```

---

## Findings

### HTTP Security Headers — Strong

The server returned a focused set of security headers:

- `Strict-Transport-Security` with `preload` and `includeSubDomains`
- `Content-Security-Policy` scoped to `'self'`
- explicit allowances for fonts and the form API
- `X-Frame-Options: SAMEORIGIN`
- `X-Content-Type-Options: nosniff`
- CORS scoped to `https://khalilsec.no`, not `*`

**Assessment:** The header set is suitable for this static portfolio site.
The CSP avoids wildcard sources in the observed production policy.

---

### Server Fingerprinting — Suppressed

`Server` and `X-Powered-By` are stripped at the reverse proxy layer.

**Assessment:** This reduces passive information disclosure. It does not
replace patching, but it avoids giving casual scanners extra detail.

---

### Informational: Non-Standard Compliance Headers Removed

An earlier scan found custom headers that implied formal compliance with
GDPR, HIPAA, PCI-DSS, SOC 2, and ISO 27001. These were not standard HTTP
headers, and the site had not gone through those audit processes.

**Assessment:** Removing those headers was the correct choice. A personal
portfolio site should not claim formal audit status without a real audit.
This was a credibility issue, not a technical vulnerability.

---

## Final Status

Recon and fingerprinting results support the current hardening approach:
minimal exposed metadata, scoped browser security headers, and no broad
CORS policy.
