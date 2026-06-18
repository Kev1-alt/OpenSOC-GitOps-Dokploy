# Architecture Decisions Records — Wazuh Multi-Node Stack

|**Version**|1.0|
|---|---|
|**Author**|Kevin YAKPOVI|
|**Last Updated**|June 2026|
|**Status**|Published|

---

## Document Purpose

### Objective

Document the architectural decisions made for the OpenSOC GitOps stack, including the context that motivated each decision, the alternatives that were considered, and the trade-offs that were explicitly accepted.

### Audience

|Role|Relevance|
|---|---|
|Security Engineers|Understanding why the stack is shaped the way it is before modifying it|
|DevSecOps Engineers|Evaluating whether this architecture fits a new environment|
|DevOps Engineers|Assessing migration paths and future evolution|
|Platform Engineers|Integrating this stack into a broader infrastructure|
|Startup Security Teams|Making an informed adoption decision|

### Prerequisites

#### Technical prerequisites

|Requirement|Details|
|---|---|
|Wazuh Multi-Node|Deployed via Docker Compose and Dokploy|
|Docker Engine|24.0+|
|Docker Compose Plugin|v2.20+|

#### Required reading

| Document                                   | Read before this guide because...                             |
| ------------------------------------------ | ------------------------------------------------------------- |
| [Deployment Guide](01-deployment-guide.md) | Provides the operational context that these decisions support |

---

### Scope

**This document covers:**

- Blueprint vs turnkey deployment approach
- Deployment platform choice (Docker Compose vs Kubernetes)
- Secrets management approach
- Volume strategy (named volumes vs bind mounts)
- SSL termination model (Traefik vs in-container)
- GitOps model and Dokploy integration
- Accepted limitations and known trade-offs

**This document does NOT cover:**

|Out of scope|See instead|
|---|---|
|Step-by-step deployment procedures|Deployment Guide|
|Credential rotation procedures|Secret Rotation Guide|
|Incident diagnosis|Troubleshooting Guide|
|Post-deployment validation|Health Check Guide|

---

### Documentation Suite

Part of the **OpenSOC GitOps Documentation Suite**.

| #      | Document                                                       | Description                              | Status           |
| ------ | -------------------------------------------------------------- | ---------------------------------------- | ---------------- |
| 00     | [README](README.md)                                            | Project overview and quick start         | ✅ Production     |
| 01     | [Deployment Guide](01-deployment-guide.md)                     | Full deployment procedure                | ✅ Production     |
| 02     | [Secret Rotation Guide](02-secret-rotation.md)                 | Credentials rotation runbook             | ✅ Production     |
| 03     | [Troubleshooting Guide](03-troubleshooting.md)                 | Incident diagnosis runbook               | ✅ Production     |
| 04     | [Health Check Guide](04-health-check.md)                       | Post-deployment validation               | ✅ Production     |
| **05** | **Architecture Decision Records**                              | **This document**                        | **✅ Production** |
| 06     | [Teardown & Clean Reinstall Runbook](06-teardown-reinstall.md) | Full teardown and from-scratch reinstall | ✅ Production     |

---

## Security Notice

This deployment intentionally keeps secrets outside the Git repository.

**Never commit:**

- `.env` files and environment variables
- TLS certificates and private keys
- `internal_users.yml`
- Backup archives
- Any file from `/etc/dokploy/secrets/`

Always verify file permissions and ownership before running any command that touches secrets or certificates. Do not expose `/etc/dokploy/secrets/` over unencrypted backups or shared snapshots.

---

# Architecture Decision Records

---

## ADR-000 — Blueprint Rather Than Turnkey Deployment

**Status:** Accepted **Date:** June 2026

### Context

Production Wazuh deployments vary significantly across organizations: different project naming conventions, domain structures, network topologies, secret management approaches, and infrastructure constraints make a single prescriptive configuration impossible to apply universally without modification.

A turnkey approach — hardcoded project names, fixed paths, pre-configured credentials — optimizes for the first deployment at the expense of every subsequent adaptation, migration, and maintenance operation.

### Decision

This project is a **reference architecture** (blueprint), not a turnkey deployment.

All environment-specific values are parameterized:

- Docker Compose project name: `-p <project-name>` (not hardcoded)
- Secret file location: `/etc/dokploy/secrets/wazuh.env` (conventional path, not enforced)
- Domain: defined in `wazuh.env`, not in source code
- Credentials: defined in `wazuh.env`, never in the repository

Commands and file paths shown throughout the documentation represent reference procedures. Engineers are expected to adapt them to their target environment before execution.

### Consequences

|Trade-off|Impact|Mitigation|
|---|---|---|
|Requires active adaptation by the engineer|Cannot be deployed by copy-paste alone|Each document includes explicit adaptation notes|
|Slightly higher initial effort|More deliberate first deployment|Reduces errors caused by blind copy-paste|
|No hardcoded defaults to fall back on|Engineer must define all values|`wazuh.env` template with placeholder values provided|

> This is a deliberate design choice. Engineers who understand the architecture they are deploying operate and maintain it more effectively than those who follow a script without context.

---

## ADR-001 — Docker Compose over Kubernetes

**Status:** Accepted **Date:** June 2026

### Context

This project targets startups, small security teams, and open source SOC builders operating with limited infrastructure resources and lean engineering teams.

A production Kubernetes deployment of a 3-node Wazuh cluster requires:

- StatefulSets with PersistentVolumeClaims for each indexer node
- A CNI plugin and cluster-level RBAC configuration
- An ingress controller replacing Traefik
- Ongoing node, control plane, and etcd management

This operational overhead does not match a team whose primary job is investigating alerts, not maintaining an orchestration platform.

### Decision

Use Docker Compose exclusively. Kubernetes is intentionally out of scope.

### Consequences

|Trade-off|Impact|Mitigation|
|---|---|---|
|No automated horizontal scaling|Fixed 3-node indexer cluster|Vertical scaling via host resize|
|No self-healing pod rescheduling|Container restarts via Docker restart policies|Health check runbook provided|
|Single-host deployment|Host failure = full stack downtime|Volume data preserved across restarts via named volumes|
|Less built-in automation|Manual operations for some tasks|Runbooks cover all critical procedures|

> This is a deliberate architectural decision, not a gap to address later. Revisit when team size or alert volume justifies the Kubernetes operational investment.

---

## ADR-002 — External Secrets via `--env-file`

**Status:** Accepted **Date:** June 2026

### Context

Dokploy supports environment variable injection through its UI. However, on complex Docker Compose stacks, this approach introduces two problems:

1. UI-defined variables are harder to audit, version, and reproduce across environments.
2. Docker Compose auto-reads a `.env` file from the working directory. When Dokploy synchronizes the Git repository (`git pull` / reset), a working-directory `.env` placed there is fragile and easy to desynchronize from the real secrets, causing silent credential drift.

Additionally, passing `--env-file` via the Run Command ensures that all variables are fully resolved by Docker Compose before Wazuh and OpenSearch entrypoint scripts execute — a common source of silent initialization failures when using `env_file` inside `docker-compose.yml` instead.

### Decision

Store all secrets in a permanent file at `/etc/dokploy/secrets/wazuh.env`, outside the Git working directory, and load it using the `--env-file` flag in the Run Command. The working-directory `.env` is kept empty or absent so that `wazuh.env` is the single source of truth.

The Git repository contains zero secrets, zero certificates, and zero environment files.

### Scope of `--env-file` injection

`--env-file` is **not** a blanket credential-injection mechanism. It only resolves `${VAR}` references that appear in a service's `environment:` block in `docker-compose.yml`. It does not reach into bind-mounted configuration files. Understanding this boundary is essential to diagnosing authentication failures and to avoiding the factory-credentials trap.

Compose variable precedence (highest first):

```text
--env-file (explicit)   >   working-directory .env (implicit)   >   shell environment
```

Effective source per component:

|Component|What `--env-file` drives|Effective source|Driven by `--env-file`?|
|---|---|---|---|
|`wazuh.master` / `wazuh.worker`|`${INDEXER_*}`, `${API_*}` in `environment:`|runtime env injection|✅ direct|
|`wazuh.dashboard`|`${API_PASSWORD}` (→ `wazuh-wui`), `${DASHBOARD_PASSWORD}` (→ `kibanaserver`) in `environment:`|runtime env injection|✅ direct|
|`wazuh.dashboard` (indexer auth hash)|bcrypt hash of `kibanaserver`|`internal_users.yml` (bind mount)|❌ literal hash|
|`wazuh1/2/3.indexer`|—|`internal_users.yml` + `securityadmin.sh`|❌ never|
|Bind-mounted configs (`opensearch_dashboards.yml`, `wazuh.yml`, `internal_users.yml`)|—|literal file content on host|❌ never|

### The two-halves invariant

Every OpenSearch credential exists as two halves that must match:

- a **plaintext half**, injected at runtime from `wazuh.env` (driven by `--env-file`), and
- a **bcrypt-hash half**, stored in `internal_users.yml` and applied by `securityadmin.sh` (never driven by `--env-file`).

`--env-file` only updates the plaintext half. A rotation that edits `wazuh.env` but not the hash leaves the indexer verifying against the old or factory hash, with no error in the logs. This is the root cause of the factory-credentials silent failure mode addressed in the Deployment Guide (§2.1, §5.6), the Secret Rotation Guide, and the Health Check Guide (Check 0).

### Consequences

|Trade-off|Impact|Mitigation|
|---|---|---|
|Secret file lives on host filesystem|Manual management required|Permissions set to `600 root:root`|
|No automatic secret rotation|Credentials persist until manually rotated|Secret Rotation Guide documents full procedure|
|Host must be backed up including secrets path|Secrets lost if host is rebuilt without backup|Documented in deployment prerequisites|
|`--env-file` scope is partial|Bind-mounted hashes are not injected — easy to rotate only one half|Two-halves invariant documented; factory-credential guard in Health Check Check 0|

---

## ADR-003 — Hybrid Volume Strategy: Named Volumes for Data, Bind Mounts for Configuration

**Status:** Accepted **Date:** June 2026

### Context

The official Wazuh Docker multi-node stack uses bind mounts pointing to absolute host filesystem paths. Configuration files that require frequent operational edits (credentials, certificates, cluster settings) must remain directly editable on the host, while persistent runtime data must survive container recreation without manual intervention.

### Decision

Apply a hybrid approach based on the nature of the data being stored:

**Named Docker volumes** — for all persistent runtime data:

```yaml
- wazuh-indexer-data-1:/var/lib/wazuh-indexer
- wazuh-indexer-data-2:/var/lib/wazuh-indexer
- wazuh-indexer-data-3:/var/lib/wazuh-indexer
- master-wazuh-logs:/var/ossec/logs
- wazuh-dashboard-config:/usr/share/wazuh-dashboard/data/wazuh/config
# ... all runtime state volumes
```

**Absolute bind mounts** — for all configuration files and certificates that require direct host-level editing:

```yaml
- /home/user/.../config/wazuh_indexer_ssl_certs/root-ca.pem:...
- /home/user/.../config/wazuh_indexer/internal_users.yml:...
- /home/user/.../config/wazuh_dashboard/wazuh.yml:...
- /home/user/.../config/wazuh_dashboard/opensearch_dashboards.yml:...
- /home/user/.../config/wazuh_cluster/wazuh_manager.conf:...
```

This v1 blueprint uses **absolute host paths** for bind mounts, because Dokploy does not run `docker compose` from the compose-file directory and relative `./config/` paths fail with `not a directory`. A future v2 blueprint will introduce bootstrap-based path abstraction.

### Why this distinction matters

Configuration files are operationally active artifacts — they are edited during secret rotation, certificate renewal, and cluster tuning. Direct host-level access is a feature, not a limitation. Persistent data volumes, by contrast, should never be edited directly — they are written exclusively by running containers and must survive container recreation.

### Consequences

|Trade-off|Impact|Mitigation|
|---|---|---|
|Config paths coupled to the host directory layout|Absolute paths must exist before first deploy|Documented in deployment prerequisites|
|Named volumes not directly browsable|Runtime data inspection requires a temporary container|Direct inspection is rarely needed for data volumes|
|Config files must be present before first deploy|Missing files cause container startup failures|Configs are version-controlled in Git (minus secrets)|
|Hybrid model requires understanding the distinction|Engineers must know which files are in volumes vs bind mounts|Configuration Reference table in each operational guide|

---

## ADR-004 — Traefik SSL Termination

**Status:** Accepted **Date:** June 2026

### Context

The official Wazuh Dashboard container enables HTTPS internally by default. When placed behind Traefik, this creates a TLS conflict: Traefik attempts to forward HTTPS to a backend that expects to manage its own TLS, resulting in 502 errors and certificate routing failures.

### Decision

Disable HTTPS inside the Dashboard container (`server.ssl.enabled: false`) and delegate all SSL termination to Traefik, which manages Let's Encrypt certificates automatically.

Internal container-to-container traffic (Dashboard → OpenSearch) retains TLS using the internally-generated certificates from `generate-indexer-certs.yml`.

### Consequences

|Trade-off|Impact|Mitigation|
|---|---|---|
|Dashboard container exposes plain HTTP internally|Only reachable via Docker network, not exposed to public|Docker network isolation|
|Let's Encrypt dependency|Certificate renewal requires ports 80/443 to be accessible|Traefik handles renewal automatically|
|Single Dashboard instance|No Dashboard-layer HA|Documented as known limitation — see ADR-005|

---

## ADR-005 — Accepted Architectural Limitations

**Status:** Accepted **Date:** June 2026

### Context

The following limitations were identified during the design phase and explicitly accepted based on the target audience of this project: startups, small security teams, and open-source SOC builders.

### Decision

The following architectural limitations are intentionally accepted:

|Limitation|Rationale|
|---|---|
|Single Wazuh Dashboard instance|Sufficient for environments with fewer than ~20 concurrent users|
|No Dashboard high availability|Additional session management complexity is not justified at this scale|
|No automatic certificate rotation|Manual rotation procedures are documented and operationally acceptable|
|No external secrets manager|Vault, AWS Secrets Manager, and similar solutions introduce unnecessary operational overhead at this scale|
|No automated OpenSearch snapshot repository|Operational requirements are met through documented manual procedures|
|No Kubernetes support|See ADR-001|
|Manual OpenSearch Security initialization|Required by OpenSearch Security design; performed only during initial deployment or volume recovery|

### Consequences

|Impact|Assessment|
|---|---|
|Reduced infrastructure complexity|Positive|
|Faster deployment and onboarding|Positive|
|Lower operational cost|Positive|
|Reduced resilience compared to enterprise architectures|Accepted|
|Increased reliance on documented operational procedures|Accepted|

> These limitations are intentional design choices rather than technical gaps. Each limitation should be reviewed individually if deployment scale, team size, compliance requirements, or availability requirements change.

---

## ADR-006 — GitOps Deployment Model with Dokploy

**Status:** Accepted **Date:** June 2026

### Context

The project requires reproducible deployments, version-controlled infrastructure, and simplified operational workflows for small security teams. Traditional manual deployments create configuration drift, complicate change tracking, and increase operational complexity.

### Decision

Use Git as the single source of truth for infrastructure definitions and Dokploy as the deployment orchestrator.

- Infrastructure configuration is stored in Git
- Secrets, certificates, and credentials remain outside the repository
- Deployments are triggered and managed through Dokploy

### Consequences

|Trade-off|Impact|Mitigation|
|---|---|---|
|Git history becomes infrastructure history|Easier auditing and rollback|Branch protection and pull request reviews recommended|
|Secrets remain external|Additional operational management|Secret Rotation Guide documents the full procedure|
|Dependency on Dokploy|Potential platform lock-in|Docker Compose stack remains fully portable and executable independently|
|Infrastructure changes require Git commits|Additional discipline required|Improved reproducibility and traceability across environments|

> GitOps is a deliberate architectural choice intended to reduce configuration drift and improve operational consistency across environments.

---

© 2026 Kevin YAKPOVI · Security Engineer · Open Source SOC Builder

OpenSOC GitOps Dokploy Blueprint — https://github.com/Kev1-alt/OpenSOC-GitOps-Dokploy

Documentation licensed under Creative Commons Attribution 4.0 International (CC BY 4.0). Source code and infrastructure examples licensed under Apache License 2.0.

This document is part of the OpenSOC GitOps Documentation Suite.
