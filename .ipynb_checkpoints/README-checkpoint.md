# khalilsec.no Security Validation

This repository documents a self-assessment of my own portfolio site, `khalilsec.no`.

The goal was to test the site like a small real-world target, document findings clearly, fix weak spots, and verify the final state.

## Scope

- Target: `https://khalilsec.no`
- Asset owner: MHD Khalil Arabieh
- Test type: Self-authorized web security validation
- Focus areas:
  - HTTP headers
  - TLS configuration
  - Content discovery
  - CORS behavior
  - HTTP methods
  - Host header handling
  - Input validation
  - Rate limiting
  - Sensitive file exposure

## Reports

1. [`00-initial-baseline.md`](00-initial-baseline.md)  
   First test state before hardening.

2. [`01-recon.md`](01-recon.md)  
   Recon and fingerprinting results.

3. [`02-web-scan.md`](02-web-scan.md)  
   TLS, application-layer checks, content discovery, and proxy review.

4. [`03-findings-and-fixes.md`](03-findings-and-fixes.md)  
   Findings, fixes, and verification.

5. [`khalilsec-validation-report.md`](khalilsec-validation-report.md)  
   Final validation summary.

## Notes

This was performed only against infrastructure I own or control.

The reports are written as a portfolio case study. They show testing method, risk judgment, fixes, and verification.

This project is licensed under the MIT License.