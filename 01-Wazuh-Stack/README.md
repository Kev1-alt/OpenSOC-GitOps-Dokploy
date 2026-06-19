# 🚀 OpenSOC GitOps — Wazuh Multi-Node Blueprint (Dokploy + Traefik)

> A production-oriented GitOps blueprint for deploying and operating a multi-node Wazuh stack using Docker Compose, Dokploy, Traefik, and GitHub.

---

## Important Disclaimer

This repository is **not a one-click deployment solution**.

It is a documented reference implementation intended to help engineers:

- Understand the architecture and design decisions behind a production-oriented Wazuh deployment
- Reproduce the deployment model in their own environment
- Adapt the provided configuration examples to their infrastructure, security requirements, and operational constraints
- Build their own GitOps-based Wazuh platform

**Users remain responsible for reviewing, adapting, and validating all configurations before production deployment.**

---

## Why this project exists

Deploying Wazuh in production is often painful:

- Fragile Docker Compose setups
- Secrets mixed with source code
- Non-reproducible environments
- Difficult upgrades and rollbacks

This project documents a **clean GitOps architecture** that addresses those challenges.

---

## Project Philosophy

This repository is intentionally provided as a blueprint rather than a turnkey deployment.

The objective is to document a reproducible architecture and operational model for running Wazuh in GitOps-oriented environments.

The repository includes:

- Example Docker Compose definitions
- Example OpenSearch Dashboard configuration
- Example environment variable structure
- Operational runbooks
- Architecture Decision Records (ADRs)

Engineers are expected to adapt these examples to their own infrastructure, security requirements, domain names, and operational constraints before deployment.

## Target Audience

This blueprint is primarily intended for:

- Security Engineers
- DevSecOps Engineers
- DevOps Engineers
- Platform Engineers
- Startup Security Teams
- Open Source SOC Builders

It is designed for teams that want to understand and adapt a production-oriented Wazuh deployment model rather than consume a turnkey installation package.

## Core idea

> "Everything reproducible. Nothing secret in Git. Fully automated deploys."

---

## Stack

|Component|Version|
|---|---|
|Wazuh|4.14.5|
|OpenSearch|Multi-node (3 indexers)|
|Docker Compose Plugin|v2.20+|
|Docker Engine|24+|
|Dokploy|0.4+|
|Traefik|SSL termination + Let's Encrypt|
|OS|Ubuntu 22.04 LTS|

---

## 🏗️ Architecture

```text
GitHub Repository
(No secrets — example configurations only)
        │
        ▼
Dokploy Git Sync
        │
        ▼
Dokploy Run Command (--env-file)
        │
        ▼
/etc/dokploy/secrets/wazuh.env
        │
        ▼
Wazuh Multi-Node Cluster
        │
        ▼
Named Docker Volumes (runtime data)  +
Absolute bind mounts (configuration & certificates)
```

---

## 🔐 Security Principles

- Secrets never stored in Git
- External `.env` only — loaded via `--env-file` in the Dokploy Run Command
- No TLS certificates in the repository
- Named Docker volumes for all persistent runtime data
- Absolute bind mounts for configuration files and certificates
- Least privilege by design

> [!CAUTION] Never commit:
>
> - `.env` files or environment variables
> - TLS certificates or private keys
> - `internal_users.yml`
> - Backup archives
> - Any file from `/etc/dokploy/secrets/`

> [!WARNING] **Rotate factory credentials before exposure**
>
> The default `wazuh-docker` install ships public factory passwords (`admin:SecretPassword`, `kibanaserver:kibanaserver`, `wazuh-wui:MyS3cr37P450r.*-`). A stack left on these starts cleanly and reports `green` while protected only by published passwords. Rotate all three before exposing the Dashboard, then confirm with [Health Check — Check 0](docs/04-health-check.md).

---

## Features

✔ Multi-node Wazuh cluster reference architecture
✔ GitOps deployment model
✔ Traefik SSL termination
✔ Named Docker volumes for runtime data
✔ Reproducible deployment examples
✔ Dokploy integration
✔ Production-oriented reference structure
✔ Operational runbooks (secret rotation, health check, troubleshooting)
✔ Architecture Decision Records (ADRs)

---

### Requirements

| **Resource** | **Minimum**              | **Recommended** |
| ------------ | ------------------------ | --------------- |
| **RAM**      | 16 GB                    | 32 GB           |
| **CPU**      | 4 vCPU                   | 8 vCPU          |
| **Storage**  | 100 GB SSD               | 200 GB NVMe     |
| **OS**       | Ubuntu 22.04 / Debian 12 | Ubuntu 22.04    |

---

## Prerequisites

- Docker Engine 24+
- Docker Compose Plugin v2.20+
- Dokploy 0.4+ installed and reachable
- Git + GitHub access
- OpenSSL available on the host
- Open ports: 80 / 443 / 1514 / 1515

---

#### 📁 Repository Structure

```text
OpenSOC-GitOps-Dokploy/
└── 01-Wazuh-Stack/
    ├── README.md
    ├── docs/
    │   ├── 01-deployment-guide.md
    │   ├── 02-secret-rotation.md
    │   ├── 03-troubleshooting.md
    │   ├── 04-health-check.md
    │   └── 05-architecture-decision-records.md
    ├── examples/
    │   └── wazuh.env.example              ← Template — copy and adapt
    └── wazuh-docker/
        ├── docker-compose.yml             ← Example — adapt before use
        └── config/
            ├── wazuh_dashboard/
            │   ├── opensearch_dashboards.yml  ← Example — adapt before use
            │   └── wazuh.yml
            └── wazuh_indexer/
                ├── internal_users.yml         ← git-ignored (real bcrypt hashes — local only)
                ├── wazuh1.indexer.yml
                ├── wazuh2.indexer.yml
                └── wazuh3.indexer.yml
```

---

## ⚙️ Quick Start

> [!IMPORTANT] The steps below illustrate the deployment model documented in this blueprint. Adapt all values — domain names, passwords, project names, paths — to your environment before executing any command.
>
> For the complete procedure, see the [Deployment Guide](docs/01-deployment-guide.md).

### 1. Prepare the repository

```bash
git clone https://github.com/wazuh/wazuh-docker.git ~/wazuh-repository

mkdir -p ~/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/

cp -r ~/wazuh-repository/multi-node/* \
  ~/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/

rm -rf ~/wazuh-repository

cd ~/OpenSOC-GitOps-Dokploy
git init
```

### 2. Create the secrets file

```bash
sudo mkdir -p /etc/dokploy/secrets
sudo nano /etc/dokploy/secrets/wazuh.env
```

```env
# Example — replace all values before use
WAZUH_VERSION=4.14.5
WAZUH_IMAGE_VERSION=4.14.5
WAZUH_TAG_REVISION=1
FILEBEAT_TEMPLATE_BRANCH=4.14.5
WAZUH_FILEBEAT_MODULE=wazuh-filebeat-0.5.tar.gz
WAZUH_UI_REVISION=1

INDEXER_USERNAME=admin
INDEXER_PASSWORD=<STRONG_PASSWORD_MIN_16_CHARS>

API_USERNAME=wazuh-wui
API_PASSWORD=<STRONG_PASSWORD_MIN_16_CHARS>

DASHBOARD_USERNAME=kibanaserver
DASHBOARD_PASSWORD=<STRONG_PASSWORD_MIN_16_CHARS>

WAZUH_DOMAIN=<wazuh.your-domain.com>
```

```bash
sudo chmod 600 /etc/dokploy/secrets/wazuh.env
sudo chown root:root /etc/dokploy/secrets/wazuh.env
```

> [!NOTE] For the very first bring-up you may deploy with the `wazuh-docker` factory defaults to validate assembly, but you must rotate all three credentials (both the plaintext in `wazuh.env` **and** the bcrypt hash in `internal_users.yml`) before exposing the stack. See [Secret Rotation Guide](docs/02-secret-rotation.md).

### 3. Deploy via Dokploy

In **Advanced → Run Command**, configure (single line):

```bash
compose -p <your-project-name> --env-file /etc/dokploy/secrets/wazuh.env -f ./01-Wazuh-Stack/wazuh-docker/docker-compose.yml up -d --remove-orphans
```

> [!NOTE] **Why `--env-file` in the Run Command?**
>
> This ensures all variables are fully resolved by Docker Compose before Wazuh and OpenSearch entrypoint scripts execute — preventing initialization failures common in multi-node clusters when `env_file` is defined inside `docker-compose.yml` instead.
>
> `--env-file` only drives `${VAR}` references in `environment:` blocks; it does not populate bind-mounted files such as `internal_users.yml`. See [ADR-002](docs/05-architecture-decision-records.md).
>
> The `-p <your-project-name>` flag controls the prefix of all Docker resources (volumes, containers, networks). Choose a consistent name before deploying — changing it later requires volume migration.

### 4. Access the Dashboard

```
https://wazuh.your-domain.com
```

---

## Key Design Decisions

### Hybrid Volume Strategy

- Named Docker volumes for runtime and persistent data
- Version-controlled configuration files (minus secrets)
- Absolute bind mounts for configuration and certificates
- GitOps compatible

### External secrets via `--env-file`

- No credential leaks in Git
- Environment isolation
- Variables fully resolved before container initialization
- Partial scope: drives `environment:` blocks only, not bind-mounted hashes

### Traefik SSL termination

- Centralized TLS management
- Automatic Let's Encrypt certificate renewal
- Eliminates SSL conflicts with the Wazuh Dashboard container

→ Full rationale in [Architecture Decision Records](docs/05-architecture-decision-records.md)

---

## 📚 Documentation Suite

Part of the **OpenSOC GitOps Documentation Suite**.

| #   | Document                      | Description                            |
| --- | ----------------------------- | -------------------------------------- |
| 00  | README (this file)            | Project overview and quick start       |
| 01  | [Deployment Guide](docs/01-deployment-guide.md) | Full step-by-step deployment procedure |
| 02  | [Secret Rotation Guide](docs/02-secret-rotation.md) | Credentials rotation runbook           |
| 03  | [Troubleshooting Guide](docs/03-troubleshooting.md) | Incident diagnosis runbook             |
| 04  | [Health Check Guide](docs/04-health-check.md) | Post-deployment validation checklist   |
| 05  | [Architecture Decision Records](docs/05-architecture-decision-records.md) | Design rationale and trade-offs        |
| 06  | [Teardown & Clean Reinstall Runbook](docs/06-teardown-reinstall.md) | Full teardown and from-scratch reinstall |

---

## 🧪 Validation

After deployment, run a quick cluster health check. Dokploy prefixes container names with the project name and a generated suffix, so resolve the real name first:

```bash
IDX1=$(docker ps --filter "name=wazuh1.indexer" --format '{{.Names}}' | head -1)

docker exec "$IDX1" curl -k \
  -u admin:<INDEXER_PASSWORD> \
  https://localhost:9200/_cluster/health?pretty
```

Expected:

```json
{
  "status": "green",
  "number_of_nodes": 3
}
```

> [!NOTE] Throughout the docs, commands use the short service name (`wazuh1.indexer`) for readability. The real container name looks like `soccenter-wazuhstack-a1b2c3-wazuh1.indexer-1` — resolve it as shown above when addressing containers from the host.

→ Full validation checklist, including the **Default Credentials Guard**, in the [Health Check Guide](docs/04-health-check.md)

---

## 🚧 Known Limitations

| Limitation                                | Notes                                                             |
| ----------------------------------------- | ----------------------------------------------------------------- |
| Single Dashboard instance                 | No Dashboard-layer HA                                             |
| No automatic certificate rotation         | Manual rotation documented in **Secret Rotation Guide**           |
| No external secrets manager               | Vault / AWS Secrets Manager out of scope                          |
| No Kubernetes support                     | Intentional — see [Architecture Decision Records](docs/05-architecture-decision-records.md) |
| Manual OpenSearch security initialization | Required by OpenSearch Security plugin design                     |

→ Full rationale in **Architecture Decision Records**

---

## Production Hardening (Recommended Next Steps)

|Priority|Item|
|---|---|
|P1 — Before go-live|Rotate all factory credentials (plaintext + hash)|
|P1 — Before go-live|Encrypted offsite backup of `/etc/dokploy/secrets/`|
|P1 — Before go-live|SSH access restrictions (key-only, no root login)|
|P1 — Before go-live|MFA on Dokploy and GitHub|
|P1 — Before go-live|OpenSearch snapshot repository|
|P2 — Within 90 days|Dedicated secrets manager (Vault, AWS Secrets Manager)|
|P2 — Within 90 days|Centralized log aggregation|
|P2 — Within 90 days|Automated backup verification|

---

## Maturity

|Level|Status|
|---|---|
|Production-oriented reference architecture|✅|
|GitOps-ready|✅|
|Small-team ready|✅|
|Enterprise-hardened|⚠️ P1/P2 hardening required|
|Multi-region|❌ Out of scope|

---

## Disclaimer

This project is provided as-is and is not affiliated with, endorsed by, or supported by Wazuh Inc.

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

**Kevin YAKPOVI** — Security Analyst · Open Source SOC Builder

- GitHub: [github.com/Kev1-alt](https://github.com/Kev1-alt)
- LinkedIn: [linkedin.com/in/kevin-yakpovi-384b4619a](https://linkedin.com/in/kevin-yakpovi-384b4619a)
- Repository: [OpenSOC-GitOps-Dokploy](https://github.com/Kev1-alt/OpenSOC-GitOps-Dokploy)
