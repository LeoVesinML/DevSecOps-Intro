# Lab 7 — Container / Kubernetes Hardening

## Task 1

### Image vulns (HIGH+CRITICAL)
`trivy image bkimminich/juice-shop:v20.0.0 --severity HIGH,CRITICAL`

| Severity | Count |
|----------|------:|
| CRITICAL | 11 |
| HIGH | 74 |
| **total** | **85** |
| with FixedVersion | **77** |
| no FixedVersion | **8** |

### Lab 4 Grype vs this Trivy
Lab 4 Grype total (all severities, from SBOM path) was **181**. This Trivy run is **HIGH+CRITICAL only (85)**. Different severity floor + different vulnerability DBs / matchers explain most of the gap — not that one tool is “wrong”.

### Top 10 fixable (7.2)
```
CRITICAL	CVE-2023-46233	crypto-js 3.3.0 -> 4.2.0
CRITICAL	CVE-2026-71851	crypto-js 3.3.0 -> 4.0.0
CRITICAL	CVE-2015-9235	jsonwebtoken 0.1.0 -> 4.2.2
CRITICAL	CVE-2015-9235	jsonwebtoken 0.4.0 -> 4.2.2
CRITICAL	CVE-2019-10744	lodash 2.4.2 -> 4.17.12
CRITICAL	CVE-2026-59873	tar 4.4.19 -> 7.5.19
CRITICAL	CVE-2026-59873	tar 6.2.1 -> 7.5.19
CRITICAL	CVE-2026-59873	tar 7.5.15 -> 7.5.19
HIGH	CVE-2026-14456	libssl3t64 3.5.5-1~deb13u2 -> 3.5.7-1~deb13u2
HIGH	CVE-2026-45447	libssl3t64 3.5.5-1~deb13u2 -> 3.5.6-1~deb13u2
```

### Dockerfile (`trivy config /tmp/df-demo`)
| ID | Severity | Impact |
|----|----------|--------|
| DS-0001 | MEDIUM | Floating `FROM node:latest` — rebuilds silently pick a new base; supply-chain drift |
| DS-0002 | HIGH | Last `USER root` — container escape / host compromise has full root in the container |
| DS-0004 | MEDIUM | `EXPOSE 22` — advertises SSH into the container attack surface |
| DS-0026 | LOW | Missing `HEALTHCHECK` — orchestrators cannot detect a hung process |

### No-fix CVEs
Do not wait for zero. Compensating controls: PSS `restricted`, NetworkPolicy default-deny, digest pin, drop ALL caps, `readOnlyRootFilesystem`, runtime detection (Lab 9). Tell a manager: “zero HIGH/CRITICAL on a fat Node teaching image is not a realistic sprint goal without an upstream rebuild; we reduce *exploitability* of the residual set.”

## Task 2

### Namespace labels
```yaml
pod-security.kubernetes.io/enforce: restricted
pod-security.kubernetes.io/warn: restricted
pod-security.kubernetes.io/audit: restricted
```
(plus `*-version: latest`)

### ServiceAccount
`automountServiceAccountToken: false` on SA `juice-shop` and on the pod spec.

### securityContext blocks
Pod:
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 65532
  runAsGroup: 65532
  seccompProfile:
    type: RuntimeDefault
```
Container:
```yaml
securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]
```
Image pinned: `bkimminich/juice-shop@sha256:fd58bdc9745416afce8184ee0666278a436574633ea7880365153a63bfd418b0`  
(from `docker inspect … RepoDigests`; image `User=65532`). Requests/limits set; NetworkPolicy Ingress+Egress (DNS to kube-system only for egress).

### Proof pod Ready + UID
```text
NAME                         READY   STATUS    RESTARTS   AGE
juice-shop-dfddd4657-fn7ls   1/1     Running   0          118s
readOnlyRootFilesystem=true runAsUser=65532 READY=true
```
Command: `kubectl -n juice-shop get pod -l app=juice-shop -o jsonpath='{.items[0].spec.securityContext.runAsUser}'` → `65532`

### Trivy k8s summaries (HIGH/CRITICAL)
| Namespace | Resource | Vuln C/H | Misconfig H | Secrets H |
|-----------|----------|----------|-------------|-----------|
| juice-plain | Deployment/juice | 11 / 74 | **3** | 2 |
| juice-shop | Deployment/juice-shop | 22 / 148 | **1** | 4 |

Misconfigurations drop under hardening (3 → 1 HIGH) because PSS/securityContext/NetworkPolicy remove common insecure defaults. Vulnerability counts do **not** shrink from hardening alone — they reflect the image contents (and juice-shop also has an init container using the same image, so Trivy counts findings twice). Only a rebuild changes the vuln column.

### Blocked vs voluntary
- **Blocked by `restricted` / had to change:** must `runAsNonRoot`, drop ALL caps, `allowPrivilegeEscalation: false`, no privileged / host namespaces.
- **Voluntary (profile does not require):** NetworkPolicy, digest pin, and `readOnlyRootFilesystem` (bonus).

## Bonus — readOnlyRootFilesystem

### `docker diff` (paths that matter)
Writes / changes under:
- `/juice-shop/.well-known/...` (CSAF metadata rewrite)
- `/juice-shop/frontend/dist/frontend/` (`index.html`, assets, videos)
- `/juice-shop/ftp`, `/juice-shop/i18n`, `/juice-shop/data` (sqlite)
- (also `/tmp`, logs in longer runs)

### Volume layout
`emptyDir` mounts for `tmp`, `ftp`, `i18n`, `data`, `logs`, `frontend/dist/frontend`, `.well-known`. Init container (`/nodejs/bin/node`, no `sh` in image) seeds trees that ship files in the image so a blank mount does not hide them.

### Hard directories
`/juice-shop/data` and `/juice-shop/frontend/dist/frontend` (and `.well-known`) cannot be empty mounts — they need seeded content from the image. Solved with initContainer copy into the emptyDirs.

### Proof HTTP 200
```text
curl http://127.0.0.1:18080/rest/admin/application-version -> HTTP 200 {"version":"20.0.0"}
curl http://127.0.0.1:18080/ -> HTTP 200
```
(via `kubectl -n juice-shop port-forward svc/juice-shop 18080:3000`)
