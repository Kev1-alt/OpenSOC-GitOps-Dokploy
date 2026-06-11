# Deployment Guide — Wazuh Multi-Node Stack with Dokploy & Traefik

|**Version**|1.0|
|---|---|
|**Author**|Kevin YAKPOVI|
|**Last Updated**|June 2026|
|**Status**|Production|

---

## Document Purpose

### Objective

This guide documents the deployment methodology used to build the OpenSOC GitOps reference architecture for a production-oriented Wazuh multi-node environment using Docker Compose, Dokploy, GitHub, and Traefik.

The objective is to provide a reproducible reference implementation that engineers can follow to:

- Deploy a Wazuh multi-node cluster using named Docker volumes
- Separate secrets from source code
- Improve deployment reproducibility across environments
- Simplify maintenance and upgrades
- Reduce dependency on host-specific filesystem paths

### Blueprint Approach

This guide documents a reference implementation, not a turnkey deployment procedure.

The repository provides:

- Example Docker Compose definitions
- Example OpenSearch Dashboard configuration
- Example environment variable structures
- Operational runbooks

These examples are intended to be reviewed and adapted by engineers according to:

- Infrastructure and networking constraints
- Security and compliance requirements
- Domain naming conventions
- Operational practices

**Review and customize all configuration examples before production use.**

### Documentation Suite

Part of the **OpenSOC GitOps Documentation Suite**.

| #      | Document                                       | Description                      | Status           |
| ------ | ---------------------------------------------- | -------------------------------- | ---------------- |
| 00     | [README](../README.md)                         | Project overview and quick start | ✅ Production     |
| **01** | **Deployment Guide**                           | **This document**                | **✅ Production** |
| 02     | [Secret Rotation Guide](02-secret-rotation.md) | Credentials rotation runbook     | ✅ Production     |
| 03     | [Troubleshooting Guide](03-troubleshooting.md) | Incident diagnosis runbook       | ✅ Production     |
| 04     | [Health Check Guide](04-health-check.md)       | Post-deployment validation       | ✅ Production     |
| 05     | [Architecture Decision Records](05-adr.md)     | Design rationale and trade-offs  | ✅ Production     |

### Audience

|Role|Relevance|
|---|---|
|Security Engineers|Understanding the full stack topology before modifying or extending it|
|DevSecOps Engineers|Owning the deployment pipeline, secret management, and volume lifecycle|
|DevOps Engineers|Integrating the stack into an existing Dokploy/Traefik infrastructure|
|Platform Engineers|Adapting the reference architecture to organizational constraints|
|Startup Security Teams|Operating a production-oriented SIEM without a dedicated infrastructure team|

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

## Prerequisites

### Host Requirements

|Resource|Minimum|Recommended|
|---|---|---|
|OS|Ubuntu 22.04 LTS / Debian 12|Ubuntu 22.04 LTS|
|RAM|16 GB|32 GB|
|CPU|4 vCPU|8 vCPU|
|Storage|100 GB SSD|200 GB NVMe SSD|

### Required Software

|Software|Version|
|---|---|
|Docker Engine|24+|
|Docker Compose Plugin|v2.20+|
|Dokploy|0.4+|
|Git|2+|
|OpenSSL|Any recent version|

### Network Requirements

- Public DNS record pointing to the server
- TCP 80 and 443 open inbound (Traefik)
- UDP 1514 open for Wazuh agent events
- TCP 1515 open for Wazuh agent enrollment

> [!NOTE] **OpenSearch kernel requirement**
> 
> OpenSearch requires `vm.max_map_count=262144` on the host.
> 
> Add to `/etc/sysctl.conf`:
> 
> ```bash
> vm.max_map_count=262144
> ```
> 
> Apply immediately:
> 
> ```bash
> sudo sysctl -p
> ```

### Tested Environment

|Component|Version|
|---|---|
|Ubuntu|22.04 LTS|
|Docker Engine|28.x|
|Docker Compose Plugin|v2.x|
|Dokploy|0.23.x|
|Wazuh|4.14.5|

> [!NOTE] This guide has been validated against the versions listed above. Newer or older versions of Docker, Dokploy, Wazuh, or Ubuntu may require adjustments.

---

## Architecture Overview

```text
GitHub Repository
(No secrets — example configurations only)
        │
        ▼
Dokploy Git Sync
        │
        ▼
docker compose --env-file
        │
        ▼
/etc/dokploy/secrets/wazuh.env
        │
        ▼
Wazuh Multi-Node Cluster
        │
        ▼
Named Docker Volumes (runtime data)
Bind Mounts ./config/ (certificates & configuration)
```

### Core Principles

- Hybrid volume strategy — named volumes for data, bind mounts for configuration
- Infrastructure as Code — all configuration tracked in Git
- Separation of Secrets and Source Code
- Least Privilege
- Reproducible Deployments

---

# Step 1 — Repository Preparation

Clone the official Wazuh Docker repository and isolate the multi-node deployment files.

```bash
git clone https://github.com/wazuh/wazuh-docker.git ~/wazuh-repository

mkdir -p ~/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/

cp -r \
  ~/wazuh-repository/multi-node \
  ~/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker

rm -rf ~/wazuh-repository

cd ~/OpenSOC-GitOps-Dokploy/
git init
```

Expected repository structure:

```text
OpenSOC-GitOps-Dokploy/
├── .gitignore
├── README.md
└── 01-Wazuh-Stack/
    └── wazuh-docker/
        ├── docker-compose.yml
        ├── generate-indexer-certs.yml
        └── config/
            ├── wazuh_dashboard/
            │   └── opensearch_dashboards.yml
            ├── wazuh_cluster/
            │   ├── wazuh_manager.conf
            │   └── wazuh_worker.conf
            └── wazuh_indexer/
                ├── wazuh1.indexer.yml
                ├── wazuh2.indexer.yml
                └── wazuh3.indexer.yml
```

Configure `.gitignore` at the repository root:

```gitignore
# Secrets & environment
.env
*.env
secrets/

# TLS certificates & keys
*.pem
*.key
*.crt
*.csr
*.p12
*.key.pem

# Sensitive Wazuh configurations
config/wazuh_indexer_ssl_certs/
config/wazuh_indexer/internal_users.yml
config/wazuh_dashboard/opensearch_dashboards.yml
config/wazuh_dashboard/wazuh.yml
config/wazuh_cluster/wazuh_manager.conf
config/wazuh_cluster/wazuh_worker.conf

# Persistent data & logs
data/
logs/
os-data*/
wazuh-data*/
```

---

# Step 2 — Secrets and Environment Configuration

## Create the External Docker Network

```bash
docker network create dokploy-network || true
```

## Create a Persistent Secrets File

```bash
sudo mkdir -p /etc/dokploy/secrets
sudo chmod 700 /etc/dokploy/secrets
sudo nano /etc/dokploy/secrets/wazuh.env
```

> [!CAUTION] Replace all values between `< >` with your own values before saving. **Never commit this file.**

```env
# ============================================================
# WAZUH VERSIONS — Edit to update the stack
# ============================================================
WAZUH_VERSION=4.14.5
WAZUH_IMAGE_VERSION=4.14.5
WAZUH_TAG_REVISION=1
FILEBEAT_TEMPLATE_BRANCH=4.14.5
WAZUH_FILEBEAT_MODULE=wazuh-filebeat-0.5.tar.gz
WAZUH_UI_REVISION=1

# ============================================================
# CREDENTIALS — Never commit these values
# ============================================================
INDEXER_USERNAME=admin
INDEXER_PASSWORD=<STRONG_PASSWORD_MIN_16_CHARS>

API_USERNAME=wazuh-wui
API_PASSWORD=<STRONG_PASSWORD_MIN_16_CHARS>

DASHBOARD_USERNAME=kibanaserver
DASHBOARD_PASSWORD=<STRONG_PASSWORD_MIN_16_CHARS>

# ============================================================
# NETWORK
# ============================================================
WAZUH_DOMAIN=<wazuh.your-domain.com>
```

Secure the file immediately:

```bash
sudo chmod 600 /etc/dokploy/secrets/wazuh.env
sudo chown root:root /etc/dokploy/secrets/wazuh.env
```

## Back Up Original Configuration Files

Before modifying any configuration, retain a cold backup:

```bash
sudo mkdir -p /etc/dokploy/secrets/configs

cd ~/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/

sudo cp config/wazuh_indexer/internal_users.yml          /etc/dokploy/secrets/configs/
sudo cp config/wazuh_dashboard/opensearch_dashboards.yml /etc/dokploy/secrets/configs/
sudo cp config/wazuh_dashboard/wazuh.yml                 /etc/dokploy/secrets/configs/
sudo cp config/wazuh_cluster/wazuh_manager.conf          /etc/dokploy/secrets/configs/
sudo cp config/wazuh_cluster/wazuh_worker.conf           /etc/dokploy/secrets/configs/
```

> [!WARNING] **Version upgrades — Mandatory checks**
> 
> When upgrading Wazuh, do not simply change `WAZUH_VERSION` and redeploy. Always verify:
> 
> - Filebeat template compatibility
> - OpenSearch Security Plugin schema changes
> - Configuration file changes (`manager.conf`, etc.)
> - Official release notes: https://documentation.wazuh.com/current/release-notes/
> 
> **Always test upgrades on a staging environment first.**

---

# Step 3 — Volume Architecture and Dashboard Configuration

## 3.1 Dashboard Configuration for Traefik SSL Termination

By default, the official Wazuh Docker deployment enables HTTPS directly inside the Dashboard container. When deployed behind Traefik, this causes SSL conflicts and HTTP 502 errors.

Disable HTTPS inside the Dashboard and allow Traefik to handle SSL termination.

Update `config/wazuh_dashboard/opensearch_dashboards.yml`:

```yaml
server.host: 0.0.0.0
server.port: 5601

# High availability — all three indexers declared
opensearch.hosts:
  - "https://wazuh1.indexer:9200"
  - "https://wazuh2.indexer:9200"
  - "https://wazuh3.indexer:9200"

opensearch.ssl.verificationMode: full
opensearch.requestHeadersWhitelist: ["securitytenant","Authorization"]

opensearch_security.multitenancy.enabled: false
opensearch_security.readonly_mode.roles: ["kibana_read_only"]

# Traefik handles external HTTPS — Dashboard listens on plain HTTP internally
server.ssl.enabled: false
#server.ssl.key: "/usr/share/wazuh-dashboard/certs/wazuh-dashboard-key.pem"
#server.ssl.certificate: "/usr/share/wazuh-dashboard/certs/wazuh-dashboard.pem"
opensearch.ssl.certificateAuthorities:
  - "/usr/share/wazuh-dashboard/certs/root-ca.pem"

uiSettings.overrides.defaultRoute: /app/wz-home

# Session configuration — 8 hours
# Default value of 15 minutes causes "Invalid credentials" after inactivity
opensearch_security.cookie.ttl: 28800000
opensearch_security.session.ttl: 28800000
opensearch_security.session.keepalive: true
opensearch_security.cookie.secure: true
```

> [!NOTE] This blueprint deploys a single Dashboard instance. For Dashboard-layer HA, see [ADR-005](05-adr.md#adr-005--accepted-architectural-limitations).

> [!NOTE] **Session expiry and "Invalid credentials" after logout**
> 
> If the Dashboard returns `Invalid credentials` after disconnect/reconnect without any API-side error, the cause is a stale session cookie not properly invalidated client-side. Clear browser cookies for the domain or test in a private window to confirm. The 8-hour TTL configured above prevents this under normal usage patterns.

## 3.2 Volume Strategy — Named Volumes for Data, Bind Mounts for Configuration

This blueprint applies a hybrid volume strategy based on the nature of the data being stored. See [ADR-003](05-adr.md#adr-003--hybrid-volume-strategy-named-volumes-for-data-bind-mounts-for-configuration) for the full rationale.

**Named Docker volumes** — for all persistent runtime data that must survive container recreation and redeployments:

```yaml
volumes:
  wazuh-indexer-data-1:
  wazuh-indexer-data-2:
  wazuh-indexer-data-3:
  master-wazuh-logs:
  master-wazuh-etc:
  master-filebeat-etc:
  master-filebeat-var:
  # ... all runtime state volumes declared at the bottom of docker-compose.yml
```

**Bind mounts from `./config/`** — for certificates and configuration files that require direct host-level editing during operations (secret rotation, certificate renewal, cluster tuning):

```yaml
# Certificates — mounted directly from the working directory
- ./config/wazuh_indexer_ssl_certs/root-ca.pem:/usr/share/wazuh-indexer/config/certs/root-ca.pem
- ./config/wazuh_indexer_ssl_certs/wazuh1.indexer.pem:/usr/share/wazuh-indexer/config/certs/wazuh1.indexer.pem
# ...

# Configuration files
- ./config/wazuh_indexer/internal_users.yml:/usr/share/wazuh-indexer/config/opensearch-security/internal_users.yml
- ./config/wazuh_dashboard/wazuh.yml:/usr/share/wazuh-dashboard/data/wazuh/config/wazuh.yml
- ./config/wazuh_dashboard/opensearch_dashboards.yml:/usr/share/wazuh-dashboard/config/opensearch_dashboards.yml
- ./config/wazuh_cluster/wazuh_manager.conf:/wazuh-config-mount/etc/ossec.conf
```

The `./config/` path is relative to the Docker Compose working directory. Dokploy consistently resolves this to `/etc/dokploy/compose/<APP_ID>/code/` during deployment — no absolute host paths are required.

> [!NOTE] Configuration files mounted via bind mounts are directly editable on the host without any container intermediary. This is intentional — operations such as secret rotation and certificate renewal rely on direct file access. See [Secret Rotation Guide](02-secret-rotation.md) for procedures that depend on this behavior.

Declare all named volumes at the bottom of `docker-compose.yml`:

```yaml
volumes:
  wazuh-indexer-config:
  wazuh-certs:

  master-wazuh-api-configuration:
  master-wazuh-etc:
  master-wazuh-logs:
  master-wazuh-queue:
  master-wazuh-var-multigroups:
  master-wazuh-integrations:
  master-wazuh-active-response:
  master-wazuh-agentless:
  master-wazuh-wodles:
  master-filebeat-etc:
  master-filebeat-var:

  worker-wazuh-api-configuration:
  worker-wazuh-etc:
  worker-wazuh-logs:
  worker-wazuh-queue:
  worker-wazuh-var-multigroups:
  worker-wazuh-integrations:
  worker-wazuh-active-response:
  worker-wazuh-agentless:
  worker-wazuh-wodles:
  worker-filebeat-etc:
  worker-filebeat-var:

  wazuh-indexer-data-1:
  wazuh-indexer-data-2:
  wazuh-indexer-data-3:

  wazuh-dashboard-config:
  wazuh-dashboard-custom:
```

---

# Step 4 — Git Synchronization and Dokploy Configuration

## 4.1 Push the Repository to GitHub

```bash
cd ~/OpenSOC-GitOps-Dokploy

git add .
git commit -m "deploy: wazuh multi-node blueprint — named volumes, external secrets"

git remote add origin <YOUR_GITHUB_REPOSITORY_URL>
git branch -M main
git push -u origin main

# Create a deployment tag before every deploy — enables clean rollback
git tag v4.14.5-prod-$(date +%Y%m%d)
git push origin --tags
```

## 4.2 Create the Application in Dokploy

1. Create a new project — **Wazuh-Stack**
2. Add a **Docker Compose** service
3. Connect the GitHub repository
4. Set the Compose path:

```text
./01-Wazuh-Stack/wazuh-docker/docker-compose.yml
```

## 4.3 Environment Variable Loading Strategy

> [!IMPORTANT] **Why `--env-file` in the Run Command rather than `env_file` in `docker-compose.yml`**
> 
> Passing `--env-file` via the Run Command ensures that all variables are fully resolved by Docker Compose **before** Wazuh and OpenSearch entrypoint scripts execute.
> 
> When `env_file` is defined inside `docker-compose.yml`, variables may not be parsed in time for internal OpenSearch and Wazuh configuration scripts — a common cause of silent initialization failures in multi-node clusters.
> 
> The Run Command method is therefore the reference pattern for this deployment.

## 4.4 Configure the Run Command

In Dokploy → **Advanced → Run Command**:

```bash
compose -p <your-project-name> \
  --env-file /etc/dokploy/secrets/wazuh.env \
  -f ./01-Wazuh-Stack/wazuh-docker/docker-compose.yml \
  up -d --remove-orphans
```

> [!NOTE] The `-p` flag controls the Docker project name and therefore the prefix of all named volumes, containers, and networks. Choose a consistent name before deploying — changing it later requires volume migration. Example: `-p opensoc-wazuh` → volumes named `opensoc-wazuh_wazuh-indexer-data-1`, containers named `opensoc-wazuh-wazuh1.indexer-1`, etc.

## 4.5 Configure the Public Domain

In Dokploy → **Domains → Add Domain**:

```text
Domain : wazuh.your-domain.com
Port   : 5601
HTTPS  : Enabled (automatic Let's Encrypt via Traefik)
```

---

# Step 5 — Certificate Generation and Permissions

## 5.1 Generate Certificates

```bash
cd ~/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/

# Generate certificates using the official temporary container
docker compose -f generate-indexer-certs.yml run --rm generator
```

Verify that all certificates share the same Certificate Authority — **critical**:

```bash
for cert in config/wazuh_indexer_ssl_certs/*.pem; do
  if openssl x509 -in "$cert" -noout -issuer >/dev/null 2>&1; then
    echo "$cert"
    openssl x509 -in "$cert" -noout -issuer
  fi
done
# All certificates must return: issuer=CN=root-ca
# A different issuer on any certificate means the trust chain is broken
# and OpenSearch authentication will fail
```

## 5.2 Set Certificate Permissions

Certificates are mounted into the indexer containers via bind mounts from `./config/wazuh_indexer_ssl_certs/`. No volume injection step is required.

**Verify the UID used by the indexer container:**

```bash
# Never assume UID 1000 — always verify dynamically
docker run --rm wazuh/wazuh-indexer:4.14.5 id
# → uid=1000(wazuh) gid=1000(wazuh) groups=1000(wazuh)
# Use the UID returned by this command in the chown commands below
```

**Set ownership and permissions on the host directory:**

```bash
sudo chown -R 1000:1000 config/wazuh_indexer_ssl_certs/
sudo chmod 640 config/wazuh_indexer_ssl_certs/*.pem
```

> [!WARNING] Incorrect ownership is the most common cause of silent `securityadmin.sh` failures. Files owned by `root:root` without read access for UID 1000 will cause OpenSearch Security initialization to fail without a clear error message.

**Back up certificates and clean the working directory:**

```bash
sudo cp -r config/wazuh_indexer_ssl_certs/ \
  /etc/dokploy/secrets/certs-backup-$(date +%Y%m%d)/

# Remove from working directory — certificates must not be committed
rm -rf config/wazuh_indexer_ssl_certs/
```

## 5.3 Initial Deployment

Click **Deploy** in the Dokploy UI.

Containers may stop or report errors on the first run. This is expected — Docker creates the named volumes before they are initialized.

## 5.4 Initialize the OpenSearch Security Plugin

> [!CAUTION] **This step is mandatory after every initial deployment or volume recreation.**
> 
> Without it, the indexer logs will loop on: `Not yet initialized (you may need to run securityadmin)`
> 
> `securityadmin.sh` is the standard initialization mechanism for Wazuh 4.x / OpenSearch. Check Wazuh release notes on future upgrades — this procedure may evolve.

> [!NOTE] Container names in the commands below use the service hostname (`wazuh1.indexer`). Docker resolves these within the stack network. If running commands from the host outside the stack network, use the full container name: `<your-project-name>-wazuh1.indexer-1`.

Wait for the cluster to form before running the initialization:

```bash
docker logs wazuh1.indexer 2>&1 | grep -E "started|Security|initialized" | tail -5
```

Run `securityadmin`:

```bash
docker exec -e JAVA_HOME=/usr/share/wazuh-indexer/jdk \
  -it wazuh1.indexer \
  bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/securityadmin.sh \
  -cd /usr/share/wazuh-indexer/config/opensearch-security/ \
  -icl -nhnv \
  -cacert /usr/share/wazuh-indexer/config/certs/root-ca.pem \
  -cert /usr/share/wazuh-indexer/config/certs/admin.pem \
  -key /usr/share/wazuh-indexer/config/certs/admin-key.pem \
  -h wazuh1.indexer -p 9200
```

Expected output: `Done with success`

> [!WARNING] **Silent failure — Permissions**
> 
> If `securityadmin.sh` exits without `Done with success` and without a clear error, the cause is almost always incorrect file ownership on the certificate files.
> 
> ```bash
> # Verify UID inside the container
> docker exec wazuh1.indexer id
> 
> # Verify certificate ownership — must match the UID above, not root:root
> docker exec wazuh1.indexer ls -la /usr/share/wazuh-indexer/config/certs/
> 
> # Fix ownership on the host bind mount directory if needed
> sudo chown -R 1000:1000 config/wazuh_indexer_ssl_certs/
> ```

---

# Step 6 — Post-Deployment Validation

Run a full health check after deployment.

→ See [Health Check Guide](04-health-check.md) for the complete validation procedure.

**Quick cluster validation:**

```bash
docker exec wazuh1.indexer curl -k \
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

**Verify environment variables are injected:**

```bash
docker exec wazuh.master env | grep -E "INDEXER_USERNAME|INDEXER_PASSWORD|API_USERNAME"
# Expected: non-empty values
```

---

# Step 7 — Rollback

## Rollback via Git Tag

```bash
# List available tags
git tag -l

# Roll back to a previous tag
git checkout v4.14.4-prod-20240901
git push -f origin main

# Then click Deploy in Dokploy
```

## Rollback via Version Change

```bash
sudo nano /etc/dokploy/secrets/wazuh.env

# Revert:
# WAZUH_VERSION=4.14.4
# WAZUH_IMAGE_VERSION=4.14.4
# FILEBEAT_TEMPLATE_BRANCH=4.14.4

# Then click Deploy in Dokploy
```

> [!CAUTION] A major version rollback may cause OpenSearch index incompatibilities. If the cluster enters `yellow` or `red` state after rollback, an index reset may be required. Always test rollbacks on a staging environment first.

---

# Step 8 — Production Hardening

This blueprint is production-oriented for startups, small teams, and lab environments.

The following hardening steps are recommended before or shortly after go-live.

## P1 — Before go-live

|Item|Why|
|---|---|
|Encrypted offsite backup of `/etc/dokploy/secrets/`|Secrets are the single point of recovery — protect them|
|SSH access restrictions (key-only, no root login)|Reduces host attack surface|
|MFA on Dokploy and GitHub|Protects the deployment pipeline|
|OpenSearch snapshot repository|Enables index-level recovery|

## P2 — Within 90 days

|Item|Why|
|---|---|
|Dedicated secrets manager (Vault, AWS Secrets Manager)|Replaces host-level file management at scale|
|Centralized log aggregation|Audit trail outside the monitored system|
|Automated backup verification|Confirms backups are restorable, not just created|
|Log retention policy|Compliance and storage management|
|Regular security review of Wazuh rules and agent coverage|Ongoing detection quality|

> For design rationale on accepted limitations of this blueprint, see [Architecture Decision Records](05-adr.md).
---

## Deployment Maturity

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

© 2026 **Kevin YAKPOVI** | Security Engineer · Open Source SOC Builder

🐙 [github.com/Kev1-alt](https://github.com/Kev1-alt) · 💼 [linkedin.com/in/kevin-yakpovi-384b4619a](https://linkedin.com/in/kevin-yakpovi-384b4619a)

*Published under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — Any reproduction must credit the original author.*
*Part of [OpenSOC GitOps Dokploy](https://github.com/Kev1-alt/OpenSOC-GitOps-Dokploy) · Document maintained at: `01-Wazuh-Stack/docs/[nom-du-fichier.md]`*`_`_
