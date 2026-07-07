# khalilsec.no Validation Report

**Date:** 6 July 2026
**Scope:** Uploaded Markdown files, uploaded Jupyter notebook, and safe
checks against `https://khalilsec.no`.

---

## Files Reviewed

| File | Result | Changes made |
|---|---|---|
| `00-initial-baseline.md` | Finalized | Added the Host header fix section, normalized code blocks, cleaned wording. |
| `01-recon.md` | Finalized | Added metadata, clearer commands, and final status. |
| `02-web-scan.md` | Finalized | Normalized structure and clarified scan results. |
| `03-findings-and-fixes.md` | Finalized | Added Host header issue as Finding 4 and cleaned the verification steps. |
---

## Live Validation Status

### Browser Fetch Checks

| Check | Result |
|---|---|
| `https://khalilsec.no` | Loaded successfully. |
| `https://khalilsec.no/.env` | Returned `403 Forbidden`. |
| `https://khalilsec.no/server.js` | Returned the SPA shell, not exposed server source. |

---

## Security Result Summary

| Area | Status | Notes |
|---|---|---|
| HTTPS availability | Pass by browser fetch | Site loaded over HTTPS. |
| Sensitive extension blocking | Pass by browser fetch | `/.env` returned `403 Forbidden`. |
| Server source exposure | No exposure seen | `/server.js` returned the SPA shell. |
| Headers | Not rerun by shell | Prior docs show HSTS, CSP, XFO, and nosniff. |
| TLS | Not rerun by shell | Prior docs show modern TLS only. |
| CORS | Not rerun by shell | Prior docs show no origin reflection. |
| HTTP methods | Not rerun by shell | Prior docs show `TRACE` and `CONNECT` blocked after fix. |
| Host header handling | Not rerun by shell | Final docs include expected `421` behavior after fix. |
| Rate limiter | Not rerun by shell | Prior docs show 30 allowed, then `429`. |
| API validation | Not rerun by shell | Prior docs show malformed inputs rejected. |

---

