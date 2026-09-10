# Lab 1 — Deploy OWASP Juice Shop & Course Workflow

## Triage report

### Asset
- **Image tag:** `bkimminich/juice-shop:v20.0.0`
- **Image digest:** `bkimminich/juice-shop@sha256:fd58bdc9745416afce8184ee0666278a436574633ea7880365153a63bfd418b0`
- **Host OS:** macOS 26.5.2 (Darwin 25.5.0 arm64)
- **Docker version:** Docker version 29.2.1, build a5c7197

### Deployment
- **Run command:** `docker run -d --name juice-shop -p 127.0.0.1:3000:3000 bkimminich/juice-shop:v20.0.0`
- **Access URL:** http://127.0.0.1:3000
- **Bind:** localhost only (`127.0.0.1:3000->3000/tcp`). Binding to all interfaces would publish a deliberately vulnerable app on the LAN (dorm Wi‑Fi), so localhost-only is required.
- **Restart policy:** none (default `unless-stopped` was not set)

### Health
```
NAMES        STATUS          PORTS
juice-shop   Up 22 seconds   127.0.0.1:3000->3000/tcp
HTTP 200
{"version":"20.0.0"}
46
```

### Surface
- **Login / registration:** Account menu (top right) exposes Login and Customer Registration; both forms submit to REST endpoints without TLS on the lab bind.
- **Products:** Storefront lists products; `/api/Products` returns 46 items with prices and descriptions for anonymous callers.
- **Admin / account area:** Admin areas are discoverable via client routes and API paths (e.g. score board, administration); some are gated only by client-side checks.
- **Console errors:** Browser DevTools shows the usual Angular/JS noise plus recruiting header hints; no crash, but verbose client-side state.
- **Local storage / cookies:** Session/token material and language prefs appear in Application → Local Storage / cookies after browsing; product review GETs (`/api/Products/<id>/reviews`) succeed without authentication.

### Headers
```
HTTP/1.1 200 OK
Access-Control-Allow-Origin: *
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
Feature-Policy: payment 'self'
X-Recruiting: /#/jobs
...
```

| Header | Present? |
|--------|----------|
| Content-Security-Policy | **missing** |
| Strict-Transport-Security | **missing** |
| X-Content-Type-Options | present (`nosniff`) |
| X-Frame-Options | present (`SAMEORIGIN`) |

Missing CSP and HSTS map to OWASP Top 10:2025 **A02 Security Misconfiguration**.

### Top 3 risks
1. **Missing CSP / open CORS (`Access-Control-Allow-Origin: *`)** — Any origin can call the API from a browser, and without CSP XSS payloads have freer rein. **A02 Security Misconfiguration / A03 Injection.**
2. **Unauthenticated product/review surface** — Anonymous clients can read products and reviews; combined with IDOR-style challenges this expands the attack surface before login. **A01 Broken Access Control.**
3. **Cleartext HTTP on a vulnerable-by-design app** — No HSTS and lab traffic is HTTP; session tokens in storage are trivial to intercept on a shared network if the bind were widened. **A02 Security Misconfiguration / A07 Identification and Authentication Failures.**

## PR template

- **Path:** `.github/PULL_REQUEST_TEMPLATE.md`
- **Sections:** Goal, Changes, Testing, Artifacts & Screenshots
- **Checklist:** title follows `feat(labN): <topic>`; no secrets or large temp files; `submissions/labN.md` exists
- **Draft PR:** https://github.com/inno-devops-labs/DevSecOps-Intro/pull/1659 (template auto-fills Goal / Changes / Testing / Artifacts & Screenshots + checklist)

## GitHub community

Stars signal maintainers that the project is useful and worth maintaining; following classmates and staff makes it easier to discover PRs, discuss blockers, and keep a shared feed of course-related work. Stars/follows were attempted via API — if scope was missing, complete them in the browser (see “what you must do”).

## Bonus: CI smoke test

- **Workflow path:** `.github/workflows/lab1-smoke.yml`
- **Run URL:** https://github.com/LeoVesinML/DevSecOps-Intro/actions/runs/34472272627
- **Job URL:** https://github.com/LeoVesinML/DevSecOps-Intro/actions/runs/34472272627/job/102854697564
- **Duration:** ~17s (job `smoke` ~14s); conclusion **success**
- **Curl excerpt from job log:**
  ```
  {"version":"20.0.0"}
  ready after 3s
  ```
- **Fork PR used to trigger Actions:** https://github.com/LeoVesinML/DevSecOps-Intro/pull/1  
  (course PR remains https://github.com/inno-devops-labs/DevSecOps-Intro/pull/1659 — upstream runs stay `action_required` until maintainers approve fork workflows)
