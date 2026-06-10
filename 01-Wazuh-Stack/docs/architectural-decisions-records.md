# Architecture Decisions Records — Wazuh Multi-Node Stack

|**Version**|1.0|
|---|---|
|**Author**|Kevin YAKPOVI|
|**Last Updated**|June 2026|
|**Status**|Production|

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

|Document|Read before this guide because...|
|---|---|
|[Deployment Guide](https://claude.ai/chat/01-deployment-guide.md)|Provides the operational context that these decisions support|

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

| Out of scope                       | See instead           |
| ---------------------------------- | --------------------- |
| Step-by-step deployment procedures | Deployment Guide      |
| Credential rotation procedures     | Secret Rotation Guide |
| Incident diagnosis                 | Troubleshooting Guide |
| Post-deployment validation         | Health Check Guide    |

---

### Documentation Suite

Part of the **OpenSOC GitOps Documentation Suite**.

| #      | Document                          | Description                      | Status           |
| ------ | --------------------------------- | -------------------------------- | ---------------- |
| 00     | README                            | Project overview and quick start | ✅ Production     |
| 01     | Deployment Guide                  | Full deployment procedure        | ✅ Production     |
| 02     | Secret Rotation Guide             | Credentials rotation runbook     | ✅ Production     |
| 03     | Troubleshooting Guide             | Incident diagnosis runbook       | ✅ Production     |
| 04     | Health Check Guide                | Post-deployment validation       | ✅ Production     |
| **05** | **Architecture Decision Records** | **This document**                | **✅ Production** |

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

**Status:** Accepted  
**Date:** June 2026

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

**Status:** Accepted  
**Date:** June 2026

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

**Status:** Accepted  
**Date:** June 2026

### Context

Dokploy supports environment variable injection through its UI. However, on complex Docker Compose stacks, this approach introduces two problems:

1. UI-defined variables are harder to audit, version, and reproduce across environments.
2. Dokploy's automatic Git synchronization (`git pull` / reset) can overwrite a local `.env` file placed in the working directory, causing silent credential loss.

Additionally, passing `--env-file` via the Run Command ensures that all variables are fully resolved by Docker Compose before Wazuh and OpenSearch entrypoint scripts execute — a common source of silent initialization failures when using `env_file` inside `docker-compose.yml` instead.

### Decision

Store all secrets in a permanent file at `/etc/dokploy/secrets/wazuh.env`, outside the Git working directory, and load it using the `--env-file` flag in the Run Command.

The Git repository contains zero secrets, zero certificates, and zero environment files.

### Consequences

|Trade-off|Impact|Mitigation|
|---|---|---|
|Secret file lives on host filesystem|Manual management required|Permissions set to `600 root:root`|
|No automatic secret rotation|Credentials persist until manually rotated|Secret Rotation Guide documents full procedure|
|Host must be backed up including secrets path|Secrets lost if host is rebuilt without backup|Documented in deployment prerequisites|

---

## ADR-003 — Hybrid Volume Strategy: Named Volumes for Data, Bind Mounts for Configuration

**Status:** Accepted  
**Date:** June 2026

### Context

The official Wazuh Docker multi-node stack uses bind mounts pointing to absolute host filesystem paths (e.g. `~/wazuh-data/...`, `/home/user/...`). This creates tight coupling to the host directory structure, which conflicts with GitOps workflows and makes server migrations fragile.

A strict named-volume-only approach solves the portability problem but introduces a new one: configuration files that require frequent operational edits (credentials, certificates, cluster settings) become inaccessible without a temporary container, adding friction to routine operations such as secret rotation.

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

**Bind mounts from `./config/`** — for all configuration files that require direct host-level editing:

```yaml
- ./config/wazuh_indexer_ssl_certs/root-ca.pem:...
- ./config/wazuh_indexer/internal_users.yml:...
- ./config/wazuh_dashboard/wazuh.yml:...
- ./config/wazuh_dashboard/opensearch_dashboards.yml:...
- ./config/wazuh_cluster/wazuh_manager.conf:...
```

The `./config/` path is relative to the Docker Compose working directory, which Dokploy consistently resolves to `/etc/dokploy/compose/<APP_ID>/code/` during deployment. No absolute host paths are used anywhere in the stack.

Volume names are prefixed by the Docker Compose project name (`-p <project-name>`), making them consistently addressable across deployments.

### Why this distinction matters

Configuration files are operationally active artifacts — they are edited during secret rotation, certificate renewal, and cluster tuning. Direct host-level access (via `nano`, `sed`, or automated scripts) is a feature, not a limitation. Wrapping every config edit in an alpine container would add unnecessary friction to routine operations.

Persistent data volumes, by contrast, should never be edited directly — they are written exclusively by running containers and must survive container recreation, redeployments, and host maintenance without any manual intervention.

### Consequences

|Trade-off|Impact|Mitigation|
|---|---|---|
|Config files coupled to Dokploy working directory|`./config/` path depends on Dokploy's consistent working directory resolution|Dokploy always resolves `./` from the synced Git repo path — behavior is stable and documented|
|Named volumes not directly browsable|Runtime data inspection requires a temporary container|Alpine-based read pattern available; direct inspection is rarely needed for data volumes|
|Config files must be present before first deploy|Missing files cause container startup failures|Documented in deployment prerequisites; configs are version-controlled in Git|
|Hybrid model requires understanding the distinction|Engineers must know which files are in volumes vs bind mounts|Configuration Reference table in each operational guide documents all file locations|

---

## ADR-004 — Traefik SSL Termination

**Status:** Accepted  
**Date:** June 2026

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

**Status:** Accepted  
**Date:** June 2026

### Context

The following limitations were identified during the design phase and explicitly accepted based on the target audience of this project:

- Startups
- Small security teams
- Open-source SOC builders

Addressing these limitations would introduce additional operational complexity without providing sufficient value for the intended deployment scale.

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

> These limitations are intentional design choices rather than technical gaps.
> 
> Each limitation should be reviewed individually if deployment scale, team size, compliance requirements, or availability requirements change.

---

## ADR-006 — GitOps Deployment Model with Dokploy

**Status:** Accepted  
**Date:** June 2026

### Context

The project requires reproducible deployments, version-controlled infrastructure, and simplified operational workflows for small security teams.

Traditional manual deployments create configuration drift, complicate change tracking, and increase operational complexity when diagnosing or reproducing an environment.

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

© 2026 **Kevin YAKPOVI** | Security Engineer · Open Source SOC Builder

🐙 [github.com/Kev1-alt](https://github.com/Kev1-alt) · 💼 [linkedin.com/in/kevin-yakpovi-384b4619a](https://linkedin.com/in/kevin-yakpovi-384b4619a)

*Published under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — Any reproduction must credit the original author.*
*Part of [OpenSOC GitOps Dokploy](https://github.com/Kev1-alt/OpenSOC-GitOps-Dokploy) · Document maintained at: `01-Wazuh-Stack/docs/[nom-du-fichier.md]`*`_`_
