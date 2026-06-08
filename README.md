# Secure-by-Design CI/CD Pipeline with Integrated SOC Automation

> A self-hosted homelab that wires a GitHub Actions security pipeline directly into a full SOC stack — every push triggers SAST/secret scanning, auto-creates a TheHive alert, and ships logs to a Kibana dashboard.

---

## What This Is

A 3-VM homelab built from scratch on VMware, where a vulnerable Python app is used as a target to demonstrate an end-to-end secure development lifecycle:

- Code is pushed → GitHub Actions pipeline fires on a self-hosted runner
- Semgrep (SAST) + Gitleaks (secrets) scan the code
- Results are evaluated against thresholds → HTML report artifact generated
- If findings exceed thresholds → `notify_thehive.py` auto-creates a structured alert in TheHive
- SOC analyst promotes alert to case, adds observables, runs Cortex enrichment
- All pipeline logs ship via Filebeat → Logstash → Elasticsearch → Kibana

This is not a cloud deployment. Everything runs locally, on real VMs, with real services.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        GitHub (remote)                          │
│              Push to main → triggers Actions workflow           │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  VM1 — cicd-runner  (192.168.56.10)                             │
│                                                                 │
│  GitHub Actions self-hosted runner                              │
│  ├── Semgrep SAST scan         → semgrep-results.json           │
│  ├── Gitleaks secret scan      → gitleaks-results.json          │
│  ├── evaluate_and_report.py    → HTML report (artifact)         │
│  ├── notify_thehive.py         → POST alert to TheHive          │
│  └── Filebeat                  → ships logs to VM3:5044         │
└────────────┬────────────────────────────────────────────────────┘
             │                            │
             ▼                            ▼
┌────────────────────────┐   ┌────────────────────────────────────┐
│  VM2 — soc-core        │   │  VM3 — elk-stack                   │
│  (192.168.56.11)       │   │  (192.168.56.12)                   │
│                        │   │                                    │
│  Elasticsearch 7       │   │  Logstash    ← Filebeat (VM1)      │
│  Cassandra 4 (Docker)  │   │  Elasticsearch 7                   │
│  TheHive 5.4 (Docker)  │   │  Kibana      → CI/CD dashboard     │
│  Cortex   (Docker)     │   │                                    │
│  MISP 2.5 (native)     │   └────────────────────────────────────┘
│  MariaDB + Redis       │
└────────────────────────┘
```

**Network:** Two interfaces per VM. NAT (`172.16.46.x`) for outbound internet only. Host-only (`192.168.56.x`) for all internal SOC communication and SSH from Windows host.

---

## Pipeline Flow

```
Developer pushes code
        │
        ▼
GitHub Actions fires on VM1 (self-hosted runner)
        │
        ├── Semgrep SAST ──────────────────────────────────────┐
        │                                                       │
        └── Gitleaks secrets scan ────────────────────────────▶│
                                                               │
                                                               ▼
                                              evaluate_and_report.py
                                              ├── Threshold: SAST ≥ 5 → FAIL
                                              ├── Threshold: secrets ≥ 1 → FAIL
                                              └── HTML report uploaded as artifact
                                                               │
                                                               ▼
                                              notify_thehive.py (auto, Phase 6)
                                              ├── Secrets found → severity HIGH
                                              └── SAST findings → severity MEDIUM
                                                               │
                                                               ▼
                                              TheHive alert created in soc-org
                                                               │
                                              [manual SOC response]
                                                               │
                                                               ▼
                                              Analyst promotes alert → Case
                                              Adds observable (file hash)
                                                               │
                                                               ▼
                                              Cortex runs CIRCLHashlookup
                                              Returns hash reputation data
```

---

## Stack

| Layer | Technology | Version |
|---|---|---|
| CI/CD Runner | GitHub Actions (self-hosted) | 2.334.0 |
| SAST | Semgrep | latest |
| Secret scanning | Gitleaks | 8.18.4 |
| Case management | TheHive | 5.4 (Docker) |
| Analyzer engine | Cortex | 4.0.0 (Docker) |
| Threat intel | MISP | 2.5.37 (native) |
| SOC backend DB | Elasticsearch | 7.17.29 |
| Case DB | Cassandra | 4 (Docker) |
| Log pipeline | Filebeat → Logstash → ES | 7.17.29 |
| Dashboard | Kibana | 7.17.29 |
| OS | Ubuntu 22.04 LTS | all VMs |
| Host | VMware Workstation Pro | Windows 11 |

---

## Build Phases

| Phase | Description | Status |
|---|---|---|
| 1 | VM provisioning, networking, base hardening (UFW, Fail2ban) | ✅ Complete |
| 2 | GitHub Actions self-hosted runner, Semgrep, Gitleaks install | ✅ Complete |
| 3 | Security scanning pipeline, threshold enforcement, HTML report | ✅ Complete |
| 4 | SOC core stack: Elasticsearch, TheHive, Cortex, MISP, Cassandra on VM2 | ✅ Complete |
| 5 | ELK Stack on VM3, Filebeat on VM1, Kibana CI/CD dashboard | ✅ Complete |
| 6 | SOC automation: pipeline auto-creates TheHive alerts on findings | ✅ Complete |
| 7 | Cortex enrichment via CIRCLHashlookup, dashboard export, snapshots | ✅ Complete |
| 8 | MISP integration — Cortex → MISP analyzer | ⏳ Planned |

---

## Key Design Decisions

**Self-hosted runner over GitHub-hosted** — required for the runner to reach internal SOC services on `192.168.56.x`. GitHub-hosted runners have no path to a local homelab.

**Native Elasticsearch, Docker for TheHive/Cortex** — TheHive and Cortex have no working native install path (packages deprecated Dec 2025). Elasticsearch must be native because Docker containers can't bind to the host-only interface without explicit routing.

**Dedicated `cicd-bot` org user in TheHive** — TheHive's admin org does not have `manageAlert/create` permission. A separate `soc-org` with an analyst-profile user is required for API alert creation.

**Fixed Docker subnet** — `172.20.0.0/16` with gateway `172.20.0.1` is hardcoded in `docker-compose.yml`. Without this, the gateway IP changes on every restart, breaking Elasticsearch's `network.host` binding.

**Delayed Elasticsearch start** — a systemd service waits for Docker to create the `172.20.0.1` bridge before starting ES. Without this, ES fails to bind on boot.

---

## Screenshots

> Dashboard, alert views, and pipeline reports available on request.

---

## Author

**Oussama El Gourjt** — [@itsK1M0](https://github.com/itsK1M0)

Full portfolio: [kimo.dev](https://kimo.dev) · Built as a hands-on cybersecurity homelab to learn DevSecOps, SOC tooling, and log pipeline engineering from scratch.

---

*Repository is private. This README serves as a public showcase. Code and configs available upon request for hiring purposes.*
