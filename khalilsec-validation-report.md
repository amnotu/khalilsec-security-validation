# khalilsec.no Validation Report

**Date:** 6 July 2026
**Scope:** Uploaded Markdown files, uploaded Jupyter notebook, and safe
checks against `https://khalilsec.no`.

---

## Executive Summary

I reviewed all five uploaded artifacts and produced final versions for
each one. The Markdown files were cleaned and made more consistent. The
notebook had three concrete issues: an old domain in one ffuf output, a
misspelled code fence, and an empty code cell. Those are fixed in the
final notebook.

The local shell environment could not run live `curl` or `openssl` checks
against `khalilsec.no` because outbound network access and DNS resolution
failed in the sandbox. Browser-based fetch checks did work. They confirmed
that the site loads over HTTPS and that `/.env` returns `403 Forbidden`.

---

## Files Reviewed

| File | Result | Changes made |
|---|---|---|
| `00-initial-baseline.md` | Finalized | Added the Host header fix section, normalized code blocks, cleaned wording. |
| `01-recon.md` | Finalized | Added metadata, clearer commands, and final status. |
| `02-web-scan.md` | Finalized | Normalized structure and clarified scan results. |
| `03-findings-and-fixes.md` | Finalized | Added Host header issue as Finding 4 and cleaned the verification steps. |
| `Untitled.ipynb` | Finalized | Fixed old domain, fixed typo, replaced empty code cell with validation summary code. |

---

## Local Artifact Validation

The local validation script checked these items:

- each file exists
- each file is readable as UTF-8
- each document has an H1 heading
- fenced code blocks are balanced
- known bad patterns are absent from final files
- each file ends with a newline
- notebook JSON is valid

### Issues Found in Uploaded Originals

| Issue | File | Status |
|---|---|---|
| Old `work.gd` domain in ffuf output | `Untitled.ipynb` | Fixed |
| Misspelled fence language `ymal` | `Untitled.ipynb` | Fixed |
| Empty code cell | `Untitled.ipynb` | Fixed |
| Host header fix missing from Markdown baseline | `00-initial-baseline.md` | Fixed |
| Finding 4 missing from findings summary | `03-findings-and-fixes.md` | Fixed |
| Missing final newlines | Markdown originals | Fixed |

---

## Live Validation Status

### Browser Fetch Checks

| Check | Result |
|---|---|
| `https://khalilsec.no` | Loaded successfully. |
| `https://khalilsec.no/.env` | Returned `403 Forbidden`. |
| `https://khalilsec.no/server.js` | Returned the SPA shell, not exposed server source. |

### Shell Checks

These commands were attempted from the sandbox:

```bash
curl -sS -I --max-time 15 https://khalilsec.no
curl -sS -I --max-time 15 http://khalilsec.no
curl -sS -I --max-time 15 https://khalilsec.no/.env
openssl s_client -connect khalilsec.no:443 -servername khalilsec.no -tls1_2
```

They failed because the sandbox could not resolve or connect to the
external host. The failure was environmental, not a target finding.

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

## Final Recommendation

Use the final files in this folder as the clean report set. Then rerun
`validate-khalilsec.sh` from your own terminal or VM where DNS and outbound
HTTPS work. That will give you fresh raw header, method, CORS, TLS, and
host-header evidence.
