# 🔐 DevSecOps Flask Pipeline — Build, Break, Test, Harden, Monitor

![CI/CD](https://img.shields.io/badge/CI%2FCD-GitHub%20Actions-2088FF?logo=github-actions&logoColor=white)
![SAST](https://img.shields.io/badge/SAST-Bandit-orange)
![SCA](https://img.shields.io/badge/SCA-Safety-yellow)
![Container](https://img.shields.io/badge/Container%20Scan-Trivy-1904DA)
![Docker](https://img.shields.io/badge/Container-Docker-2496ED?logo=docker&logoColor=white)
![Tests](https://img.shields.io/badge/Security%20Tests-15%20passed-success)


An end-to-end DevSecOps lab covering the full lifecycle: exploiting a vulnerable Flask app, fixing it, automating security checks in CI/CD, hardening the container, and responding to a simulated live attack.

---

## 🎯 What this project demonstrates

Most "DevSecOps" student projects stop at "I added a linter to CI." This one goes through five real phases:

1. **Break it** — manually exploit 8 vulnerabilities (SQLi, XSS, command injection, path traversal, insecure deserialization, hardcoded secrets, info disclosure, debug mode) using STRIDE threat modeling first
2. **Fix it** — rewrite the app with parameterized queries, output encoding, subprocess whitelisting, safe file handling
3. **Automate it** — wire Bandit, Safety, Gitleaks, and pytest security tests into GitHub Actions as real merge-blocking gates
4. **Harden it** — apply CIS Docker Benchmark v1.6 (non-root user, read-only FS, seccomp, no-new-privileges) and scan with Trivy
5. **Defend it** — build a custom WAF with rate limiting, then run a full incident response exercise (detect → contain → rotate secrets → post-mortem)

---

## 🛠️ Tech Stack

| Layer | Tools |
|---|---|
| Application | Python, Flask, SQLite |
| SAST | Bandit |
| SCA (dependencies) | Safety |
| Secret Detection | Gitleaks |
| Security Testing | pytest (15 tests, 6 categories) |
| CI/CD | GitHub Actions (5 gated jobs) |
| Container | Docker, Docker Compose |
| Container Scanning | Trivy |
| Hardening Standard | CIS Docker Benchmark v1.6 |
| Runtime Defense | Custom WAF middleware + rate limiter |

---

## 🏗️ Architecture

```
┌──────────────────────────────────────────────────────┐
│  PART 1 — Build & Break                              │
│  STRIDE threat modeling (8 threats identified)        │
│  app_vulnerable.py  ──exploit──▶  4 hands-on attacks  │
│  app_secure.py      ──fixes──▶   all 8 threats closed │
├──────────────────────────────────────────────────────┤
│  PART 2 — Test & Automate                             │
│  Bandit (SAST) + Safety (SCA) + Gitleaks               │
│  pytest security suite (15 tests)                     │
│  GitHub Actions: SAST → SCA → Secrets → Container      │
│                  → Unit Tests → Security Gate          │
│  Branch protection: insecure code blocked from main    │
├──────────────────────────────────────────────────────┤
│  PART 3 — Harden & Monitor                             │
│  Docker: non-root, read-only FS, seccomp, CIS v1.6     │
│  Trivy container scan                                 │
│  Custom WAF (SQLi / XSS / cmd-injection detection)      │
│  Rate limiting + structured JSON event logging          │
│  Incident response: detect → block IP → rotate secret  │
└──────────────────────────────────────────────────────┘
```

---

## 🔍 Part 1 — Threat Modeling & Exploitation

A full **STRIDE** threat model was built before touching any exploit, mapping 8 threats (T1–T8) across Spoofing, Tampering, Repudiation, Information Disclosure, DoS, and Elevation of Privilege.

| ID | Threat | Attack Vector | Fix |
|---|---|---|---|
| T1 | SQL Injection (auth bypass) | `/login` | Parameterized queries |
| T2 | Pickle Deserialization RCE | `/load_profile` | JSON only, no pickle |
| T3 | No Audit Logging | All endpoints | Auth + admin event logging |
| T4 | Password Exposure via API | `/api/users` | Field filtering + auth |
| T5 | OS Command Injection | `/ping?host=` | Arg list, `shell=False` |
| T6 | Path Traversal | `/read_file?file=` | `safe_join` + whitelist |
| T7 | Reflected XSS | `/search?q=` | Jinja2 auto-escape + CSP |
| T8 | Hardcoded Secret Key | Source code | Loaded from env variable |

Four vulnerabilities were manually exploited end-to-end:
- **SQL Injection** — `' or 1=1 --` bypassed login entirely, returning full admin access with no valid password
- **XSS** — both `<script>` and `<img onerror=>` payloads executed unsanitized in the browser, simulating cookie theft
- **OS Command Injection** — `shell=True` combined with unvalidated input allowed arbitrary command execution (`whoami` confirmed)
- **Path Traversal** — `../` sequences exposed the full application source, including the hardcoded secret key

Every one of these was then fixed in `app_secure.py` using industry-standard patterns (see table above).

---

## 🧪 Part 2 — Automated Security Testing

**Bandit (SAST):** 5 findings on the vulnerable app (2 HIGH, 3 MEDIUM) — including `shell=True` (CWE-78) and Flask debug mode (CWE-94). **Zero HIGH/MEDIUM findings** on the secure version: a 100% issue reduction.

**Security test suite (pytest):** 15 tests across 6 categories — authentication, XSS escaping, API auth enforcement, command injection blocking, safe deserialization, security headers, and CSRF protection. All 15 pass.

**CI/CD pipeline (GitHub Actions):** 5 gated jobs — SAST, SCA, Secret Scanning, Container Scan, Security Unit Tests — feeding into a final Security Gate. This isn't decorative: a pull request from the insecure branch was actually blocked from merging into `main` when 3 of the 5 checks failed, proving the gate works in practice rather than just existing on paper.

---

## 🐳 Part 3 — Container Hardening & Runtime Defense

**CIS Docker Benchmark v1.6** applied and verified hands-on:
- Non-root container user (`whoami` → `appuser`, not root)
- Read-only filesystem (write attempts fail with `Read-only file system`)
- `no-new-privileges:true` + full seccomp syscall allowlist
- Trivy scan: 7 HIGH CVEs found, all in OS-level packages — application code itself scans clean

**Custom WAF middleware** detects and blocks SQL injection, XSS, and command injection payloads at the request level, plus rate limiting (HTTP 429 after 30 requests/60s) with structured JSON event logging.

**Incident response exercise:** simulated a P1 SQL injection attack (50 WAF_BLOCK alerts in under 5 minutes). Response included identifying the attacker IP from structured logs, blocking it at the infrastructure level, rotating the `SECRET_KEY` to invalidate all sessions, restarting the service with zero downtime (11.2s), and producing a full incident report with timeline, IOCs, and post-mortem recommendations (Fail2ban, SIEM integration, MFA, external WAF, honeypot endpoint).

---

## ▶️ Run Locally

```bash
git clone https://github.com/ali-amadou/devsecops-flask-pipeline.git
cd devsecops-flask-pipeline

pip install -r requirements.txt

# Vulnerable version (for learning/demo only — do not expose publicly)
python app_vulnerable.py

# Secure version
python app_secure.py

# Run security test suite
pytest test_security.py -v

# Run SAST comparison
python sast_scan.py
```

## 🐳 Run with Docker

```bash
docker compose up -d
docker inspect lab_web | grep -A5 SecurityOpt   # verify hardening
```

---

## 💡 Key Takeaways

- A WAF blocking every malicious request is necessary but not sufficient — without automated response (Fail2ban, SIEM), containment still requires manual log analysis and IP blocking, costing precious minutes during a real attack
- Bandit and pytest results converging on the same conclusion (100% issue reduction) gives confidence the fixes are real, not just hiding the symptom
- Container hardening doesn't eliminate all risk — the Trivy scan still found 7 HIGH CVEs in base OS packages, which is normal and shows why image patching is a continuous process, not a one-time task
- Branch-protected CI/CD gates are only meaningful if they can actually fail — proving it by pushing a deliberately insecure branch and watching it get blocked

---

## 📚 References

- [Bandit Documentation](https://bandit.readthedocs.io/)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker)
- [Trivy](https://github.com/aquasecurity/trivy)
- [STRIDE Threat Modeling](https://owasp.org/www-community/Threat_Modeling_Process)
