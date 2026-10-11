# Lab 9 — Runtime Detection and Policy as Code

## Task 1

### Environment
Falco **0.43.1** on Colima (`--vm-type vz`) so the container sees a real Linux kernel with modern eBPF. Docker Desktop on macOS cannot do this.

### Built-in rules from 9.2
Trigger:
- `docker exec -t lab9-target sh -lc 'echo hello-from-shell'` → needs `-t` (TTY)
- `docker exec lab9-target sh -c 'cat /etc/shadow'`

Counts:
```
   1 "rule":"Read sensitive file untrusted"
   1 "rule":"Terminal shell in container"
   1 "rule":"Write Below Tmp In Container"
   1 "rule":"Suspected Cryptominer Activity"
```

**Terminal shell in container** — fired because a shell ran with an attached TTY (`proc.tty=34816`). Plain `docker exec` without `-t` does not.

```json
{"priority":"Notice","rule":"Terminal shell in container","output":"A shell was spawned in a container with an attached terminal | process=sh command=sh -lc echo hello-from-shell terminal=34816 container_name=lab9-target","output_fields":{"proc.tty":34816,"proc.cmdline":"sh -lc echo hello-from-shell","container.name":"lab9-target"}}
```

**Read sensitive file untrusted** — fired on `cat /etc/shadow` inside the container.

```json
{"priority":"Warning","rule":"Read sensitive file untrusted","output":"Sensitive file opened for reading by non-trusted program | file=/etc/shadow process=cat command=cat /etc/shadow container_name=lab9-target","output_fields":{"fd.name":"/etc/shadow","proc.name":"cat","container.name":"lab9-target"}}
```

### Custom rule (9.3)
File: `labs/lab9/falco/rules/custom-rules.yaml` — rule `Write Below Tmp In Container` (WARNING, tags `[container, drift]`).

Alert JSON after `echo test > /tmp/my-write.txt`:

```json
{"priority":"Warning","rule":"Write Below Tmp In Container","output":"Write under /tmp in container (container=lab9-target user=root file=/tmp/my-write.txt command=sh -lc echo test > /tmp/my-write.txt)","output_fields":{"container.id":"254d200e0ad3","container.name":"lab9-target","fd.name":"/tmp/my-write.txt","proc.cmdline":"sh -lc echo test > /tmp/my-write.txt","user.name":"root"},"time":"2026-10-11T17:49:19.055275026Z"}
```

Incident-relevant field **not** in the human message: `container.id` (`254d200e0ad3`) — that is what you join to `docker inspect` / runtime audit logs when two pods share a name across restarts.

False positive / tuning: a JVM or Node app that unpacks caches under `/tmp` would fire on every cold start. Keep the rule, but narrow with `and not proc.name in (java, node)` (or an allow-list of images) and/or require the write to look like a dropper (`and proc.cmdline contains ".sh"` / unusual parent). That preserves drift detection without paging on every cache write.

## Task 2

### Conftest counts
| Manifest | Result |
|----------|--------|
| `juice-hardened.yaml` | **34 tests, 34 passed, 0 warnings, 0 failures** |
| `juice-unhardened.yaml` | **34 tests, 22 passed, 3 warnings, 9 failures** |
| `juice-compose.yml` | **17 tests, 17 passed, 0 warnings, 0 failures** |

(With `--policy labs/lab9/policies --all-namespaces`, including `extra/hardening.rego`. Hardened also sets `automountServiceAccountToken: false` so the new warn stays clean.)

Two unhardened failures → Rego:
- `container "juice" must set runAsNonRoot: true` → `deny` in `k8s.security` (`labs/lab9/policies/k8s-security.rego`)
- `container "juice" uses disallowed :latest tag` → `deny` for `:latest` in the same package

Compose vs Kubernetes: the same intent (non-root, no privileged, resource limits) lives on different schemas — Compose `services.*.privileged` / `user` vs Deployment `securityContext` — so you need separate Rego packages, not one file pretending the documents match. Compose passing all 15+ checks means that policy is actually evaluating green paths, not only failing forever; it validates the Compose ruleset is wired correctly.

### Extra policy (9.5)
- Policy: `labs/lab9/policies/extra/hardening.rego` (`package extra.hardening`)
  - **deny**: image must contain `@sha256:`
  - **warn**: `automountServiceAccountToken` must be `false`
- Violator: `labs/lab9/manifests/k8s/juice-violates-extra.yaml`

Runs:
```
# hardened
34 tests, 34 passed, 0 warnings, 0 failures

# violates-extra
WARN - ... pod for container "juice" should set automountServiceAccountToken: false
FAIL - ... container "juice" must pin image by digest (@sha256:)
17 tests, 15 passed, 1 warning, 1 failure
```

If I could keep only one control for the digest pin, I would keep **Conftest in CI**: it blocks the bad manifest before schedule. Falco still buys you detection when someone `kubectl edit`s live, bypasses the pipeline, or an attacker mutates a running workload — CI never sees that.

## Bonus

### Rule
`Suspected Cryptominer Activity` in `custom-rules.yaml`: CRITICAL, tags include `mitre_command_and_control`, requires **connect** + **pool port** (`fd.sport in (3333,4444,…)`) + **suspicious process** (`nc` / miner names).

On this Colima kernel, refused connects emit `evt.type=connect` but leave `fd.sport=<NA>` (missing `sys_enter_connect` TOCTOU hook). A local listener on `172.17.0.4:3333` completed the handshake so `fd.sport` populated — same syscall path a real pool would take.

### Alert
```json
{"priority":"Critical","rule":"Suspected Cryptominer Activity","output":"Suspected cryptomining connect (container=lab9-target process=nc dest=172.17.0.4:3333 cmdline=nc -w 2 172.17.0.4 3333)","output_fields":{"container.name":"lab9-target","proc.name":"nc","fd.rip":"172.17.0.4","fd.sport":3333,"proc.cmdline":"nc -w 2 172.17.0.4 3333"},"time":"2026-10-11T17:54:21.065228836Z"}
```

A refused connection is enough in principle because Falco sits on the **syscall** path: it sees `connect()` before (or without) application success. That implies detection does not depend on packet capture of established sessions.

Evasion: changing the pool **port** is cheap (edit one number). Renaming the binary is also cheap. What is expensive is looking like a normal app forever — TLS to an allow-listed destination on 443 with no miner cmdline, or living entirely in-process inside a trusted binary. I would add DNS/`fd.sip.name` denylists for known pool domains, CPU-steal / unusual `proc.exepath` not in the image, and egress policy that only allows expected destinations so “random high port to the internet” is denied at the network layer before Falco has to care.
