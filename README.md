# 🛡️ OpenSOC GitOps — Open Source SOC Platform Blueprint

> A modular, production-oriented GitOps blueprint for building and operating a fully integrated open source SOC platform using Docker Compose, Dokploy, Traefik, and GitHub.

---

## Important Disclaimer

This repository is **not a one-click deployment solution**.

It is a documented reference implementation intended to help engineers:

- Understand the architecture and design decisions behind a production-oriented SOC platform
- Deploy and operate each module independently or as an integrated stack
- Adapt the provided configuration examples to their own infrastructure and constraints
- Build their own GitOps-based open source SOC

**Users remain responsible for reviewing, adapting, and validating all configurations before production deployment.**

---

## What is OpenSOC GitOps?

OpenSOC GitOps is a modular SOC platform blueprint built around best-of-breed open source security tools, connected through a GitOps operational model.

Each module is independently deployable and production-oriented. Together, they form a fully integrated SOC platform covering detection, network monitoring, automation, incident response, and real-time alerting.

The platform is designed for:

- **Security Engineers** who need a reproducible, auditable SOC foundation
- **DevSecOps Engineers** who want GitOps-native security operations
- **DevOps / Platform Engineers** integrating security tooling into existing infrastructure
- **Startup Security Teams** building a production SOC without enterprise budgets
- **Open Source SOC Builders** who want documented, adaptable blueprints

---

## Project Philosophy

> "Everything reproducible. Nothing secret in Git. Fully automated deploys."

This repository is intentionally provided as a blueprint rather than a turnkey deployment.

Every module includes:

- Example Docker Compose definitions
- Example configuration files
- Example environment variable structures
- Operational runbooks
- Architecture Decision Records (ADRs)

Engineers are expected to adapt these examples to their own infrastructure, security requirements, domain names, and operational constraints before deployment.

---

## 🏗️ Platform Architecture

```text
                        GitHub Repository
                        (No secrets — blueprints only)
                                │
                                ▼
                        Dokploy Git Sync
                                │
                        ┌───────┴────────┐
                        │                │
               --env-file          absolute bind mounts
               (secrets)           (configuration)
                        │                │
                        └───────┬────────┘
                                ▼
        ┌───────────────────────────────────────────┐
        │              Traefik (SSL / Routing)       │
        └──────┬──────────┬──────────┬──────────────┘
               │          │          │
        ┌──────▼──┐  ┌────▼────┐  ┌─▼──────────┐
        │  Wazuh  │  │Suricata │  │   Shuffle  │
        │  SIEM   │  │  NIDS   │  │    SOAR    │
        └──────┬──┘  └────┬────┘  └─┬──────────┘
               │          │         │
               └──────────┴─────────┘
                           │
                    ┌──────▼──────┐
                    │  TheHive +  │
                    │   Cortex    │
                    │  (IR / TI)  │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │   Discord   │
                    │  Real-time  │
                    │   Alerts    │
                    └─────────────┘
                           │
                    Named Docker Volumes
```

---

## 📦 Modules

| #   | Module                              | Role                                          | Status      | Docs                                                              |
| --- | ----------------------------------- | --------------------------------------------- | ----------- | ----------------------------------------------------------------- |
| 01  | [Wazuh Multi-Node](01-Wazuh-Stack/) | SIEM · Log Management · Agent-based Detection | ✅ Published | [README](01-Wazuh-Stack/README.md) · [Docs](01-Wazuh-Stack/docs/) |
| 02  | Suricata                            | Network IDS · Traffic Analysis                | 🔄 Coming   | —                                                                 |
| 03  | Shuffle SOAR                        | Alert Automation · Playbooks · Orchestration  | 🔄 Coming   | —                                                                 |
| 04  | TheHive + Cortex                    | Incident Response · Threat Intelligence       | 🔄 Coming   | —                                                                 |
| 05  | Discord Alerting                    | Real-time SOC Notifications                   | 🔄 Coming   | —                                                                 |

---

## 🔗 Integration Model

All modules are designed to work together as a connected SOC platform:

```text
Wazuh          →  detects threats, generates alerts
Suricata       →  monitors network traffic, feeds Wazuh
Shuffle SOAR   →  automates alert triage and response workflows
TheHive        →  manages incidents, cases, and observables
Cortex         →  enriches observables with threat intelligence analyzers
Discord        →  delivers real-time SOC alerts and notifications
```

Each module exposes its services on the shared Traefik reverse proxy and communicates over a shared Docker network. Secrets are managed per-module via dedicated `.env` files under `/etc/dokploy/secrets/`.

---

## 🔐 Security Principles

Consistent across all modules:

- Secrets never stored in Git
- External `.env` files loaded via `--env-file` in Dokploy Run Commands
- No TLS certificates in the repository
- Named Docker volumes for all persistent runtime data
- Configuration files and certificates via absolute bind mounts
- Least privilege by design

> [!CAUTION] Never commit across any module:
> 
> - `.env` files or environment variables
> - TLS certificates or private keys
> - Internal user configuration files
> - Backup archives
> - Any file from `/etc/dokploy/secrets/`

> [!WARNING] **Rotate factory credentials before exposure (all modules)**
> 
> Upstream images frequently ship public factory credentials. A stack left on defaults can start cleanly and report healthy while protected only by published passwords. Rotate before exposing any service. For Wazuh specifically, see [Health Check — Check 0](01-Wazuh-Stack/docs/04-health-check.md).

---

## 🚀 Getting Started

Each module is independently deployable. Start with the SIEM foundation:

### 1. Deploy Wazuh (SIEM core)

→ [01-Wazuh-Stack/README.md](01-Wazuh-Stack/README.md) → [Deployment Guide](01-Wazuh-Stack/docs/01-deployment-guide.md)

### Recommended deployment order

```text
1. 01-Wazuh-Stack       ← SIEM foundation — start here
2. 02-Suricata          ← Network visibility layer
3. 04-TheHive-Cortex    ← Incident response platform
4. 03-Shuffle-SOAR      ← Automation and orchestration layer
5. Discord              ← Real-time alerting layer
```

---

## 📋 Common Requirements

All modules share the following base requirements:

|Resource|Minimum|Recommended|
|---|---|---|
|OS|Ubuntu 22.04 LTS / Debian 12|Ubuntu 22.04 LTS|
|Docker Engine|24+|Latest stable|
|Docker Compose Plugin|v2.20+|Latest stable|
|Dokploy|0.4+|Latest stable|
|RAM|16 GB (Wazuh alone)|32 GB+ (full platform)|
|Storage|100 GB SSD|500 GB NVMe SSD|

> [!NOTE] Full platform deployment (all modules running simultaneously) requires significantly more resources than a single module. Size your host accordingly.

---

## 📚 Documentation

Each module ships with its own documentation suite:

| Module           | Deployment                                      | Secret Rotation                                | Troubleshooting                                | Health Check                                | ADR                                                          |
| ---------------- | ----------------------------------------------- | ---------------------------------------------- | ---------------------------------------------- | ------------------------------------------- | ------------------------------------------------------------ |
| Wazuh            | [✅](01-Wazuh-Stack/docs/01-deployment-guide.md) | [✅](01-Wazuh-Stack/docs/02-secret-rotation.md) | [✅](01-Wazuh-Stack/docs/03-troubleshooting.md) | [✅](01-Wazuh-Stack/docs/04-health-check.md) | [✅](01-Wazuh-Stack/docs/05-architecture-decision-records.md) |
| Suricata         | 🔄                                              | 🔄                                             | 🔄                                             | 🔄                                          | 🔄                                                           |
| Shuffle SOAR     | 🔄                                              | 🔄                                             | 🔄                                             | 🔄                                          | 🔄                                                           |
| TheHive + Cortex | 🔄                                              | 🔄                                             | 🔄                                             | 🔄                                          | 🔄                                                           |
| Discord          | 🔄                                              | —                                              | 🔄                                             | 🔄                                          | 🔄                                                           |

The Wazuh module also ships a [Teardown & Clean Reinstall Runbook](01-Wazuh-Stack/docs/06-teardown-reinstall.md) for full removal and from-scratch reinstallation.

---

## 🗺️ Roadmap

- [x] Wazuh Multi-Node GitOps Blueprint
- [ ] Suricata NIDS integration with Wazuh
- [ ] Shuffle SOAR — Wazuh alert automation playbooks
- [ ] TheHive + Cortex — incident response platform
- [ ] Discord real-time SOC alerting
- [ ] Full platform health check runbook
- [ ] SPDX license headers per module

---

## 🚧 Platform-Level Limitations

|Limitation|Notes|
|---|---|
|Single-host deployment model|All modules run on one host — no multi-host orchestration|
|No Kubernetes support|Intentional — see Wazuh ADR-001|
|No enterprise HA|Single instances per service|
|No automated certificate rotation|Manual procedures documented per module|
|No external secrets manager|Vault / AWS Secrets Manager out of scope for v1|

---

## Disclaimer

This project is provided as-is and is not affiliated with, endorsed by, or supported by Wazuh Inc., Suricata, Shuffle, TheHive Project, or any other upstream project.

Users are responsible for reviewing, adapting, and validating all configurations and procedures in their own environments before production use.

---

## 📄 License

This repository uses a dual-license model:

- **Source code, infrastructure examples, Docker Compose files, deployment scripts, and configuration examples** are licensed under the **Apache License 2.0**.
- **Documentation** (README files, `/docs`, runbooks, guides, and ADRs) is licensed under **Creative Commons Attribution 4.0 International (CC BY 4.0)**.

See:

- `LICENSE`
- `LICENSE-DOCS`

---

## Maintainer

**Kevin YAKPOVI** — Security Engineer · SOC Automation · Open Source SOC Builder

- GitHub: [github.com/Kev1-alt](https://github.com/Kev1-alt)
- LinkedIn: [linkedin.com/in/kevin-yakpovi-384b4619a](https://linkedin.com/in/kevin-yakpovi-384b4619a)
- Repository: [OpenSOC-GitOps-Dokploy](https://github.com/Kev1-alt/OpenSOC-GitOps-Dokploy)
