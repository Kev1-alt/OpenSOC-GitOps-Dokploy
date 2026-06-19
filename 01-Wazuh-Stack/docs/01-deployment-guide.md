# Deployment Guide — Wazuh Multi-Node Stack with Dokploy & Traefik

| **Version**      | 1.0 — Reference Blueprint |
| ---------------- | ------------------------- |
| **Author**       | Kevin YAKPOVI             |
| **Last Updated** | June 2026                 |
| **Status**       | Published                 |

---
> [!IMPORTANT]
>
> This document describes the v1 Reference Blueprint
> based on absolute host paths.
>
> It was intentionally designed for transparency,
> troubleshooting simplicity, and learning purposes.
>
> A future v2 blueprint will introduce bootstrap-based
> path abstraction for improved portability.

## Document Purpose

### Objective

This guide documents the deployment methodology used to build the OpenSOC GitOps reference architecture for a production-oriented Wazuh multi-node environment using Docker Compose, Dokploy, GitHub, and Traefik.

The objective is to provide a reproducible reference implementation that engineers can follow to:

- Deploy a Wazuh multi-node cluster using named Docker volumes
- Separate secrets from source code
- Improve deployment reproducibility across environments
- Simplify maintenance and upgrades

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

| #      | Document                                                             | Description                              | Status           |
| ------ | -------------------------------------------------------------------- | ---------------------------------------- | ---------------- |
| 00     | [README](../README.md)                                               | Project overview and quick start         | ✅ Production     |
| **01** | **Deployment Guide**                                                 | **This document**                        | **✅ Production** |
| 02     | [Secret Rotation Guide](02-secret-rotation.md)                       | Credentials rotation runbook             | ✅ Production     |
| 03     | [Troubleshooting Guide](03-troubleshooting.md)                       | Incident diagnosis runbook               | ✅ Production     |
| 04     | [Health Check Guide](04-health-check.md)                             | Post-deployment validation               | ✅ Production     |
| 05     | [Architecture Decision Records](05-architecture-decision-records.md) | Design rationale and trade-offs          | ✅ Production     |
| 06     | [Teardown & Clean Reinstall Runbook](06-teardown-reinstall.md)       | Full teardown and from-scratch reinstall | ✅ Production     |

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

## Reference Environment

This guide documents the environment used to validate the OpenSOC GitOps blueprint. All commands use the reference paths shown below.

**Adapt every path, domain, project name, and credential to your own environment before executing any command.**

|Variable|Reference value|Description|
|---|---|---|
|Repository root|`/home/user/OpenSOC-GitOps-Dokploy`|Location of the cloned repository on the host|
|Docker project name|`soccenter-wazuhstack`|Value passed to `-p` in the Run Command|
|Secrets path|`/etc/dokploy/secrets/`|Permanent secrets storage — not in Git|
|Dashboard domain|`wazuh.your-domain.com`|Public FQDN for the Wazuh Dashboard|

Examples of alternative repository roots:

```text
/opt/opensoc/
/srv/security/opensoc/
/data/git/opensoc/
```

All commands throughout this guide that reference `/home/user/OpenSOC-GitOps-Dokploy` should be replaced with the actual path used in your environment.

> [!NOTE] **Container addressing**
> 
> Commands in this guide address containers by their short service name (`wazuh1.indexer`, `wazuh.master`, `wazuh.dashboard`). These are the container **hostnames**. On the host, Dokploy prefixes the actual container names with the project name and a generated suffix (e.g. `soccenter-wazuhstack-a1b2c3-wazuh1.indexer-1`), so `docker exec wazuh1.indexer` will not resolve directly. Resolve the real name once per shell:
> 
> ```bash
> IDX1=$(docker ps --filter "name=wazuh1.indexer" --format '{{.Names}}' | head -1)
> # then: docker exec "$IDX1" ...
> ```
> 
> Treat the short names below as placeholders, the same way `<project-name>` and `wazuh.your-domain.com` are.

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
dokploy command --env-file
        │
        ▼
/etc/dokploy/secrets/wazuh.env
        │
        ▼
Wazuh Multi-Node Cluster
        │
        ▼
Named Docker Volumes (runtime data)
Absolute host paths /home/user/.../ (certificates & configuration)
```

### Core Principles

- Hybrid volume strategy — named volumes for runtime data, absolute bind mounts for configuration
- Infrastructure as Code — all configuration tracked in Git
- Separation of Secrets and Source Code
- Least Privilege
- Reproducible Deployments

---

# Step 1 — Repository Preparation

Clone the official Wazuh Docker repository and isolate the multi-node deployment files.

```bash
git clone https://github.com/wazuh/wazuh-docker.git ~/wazuh-repository

mkdir -p ~/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/

# Copy the contents of multi-node — not the folder itself
cp -r ~/wazuh-repository/multi-node/* \
  ~/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/

rm -rf ~/wazuh-repository

cd ~/OpenSOC-GitOps-Dokploy/
git init
```

Expected repository structure:

```text
OpenSOC-GitOps-Dokploy/
└── 01-Wazuh-Stack/
    └── wazuh-docker/
        ├── docker-compose.yml
        ├── generate-indexer-certs.yml
        └── config/
            ├── wazuh_dashboard/
            │   ├── opensearch_dashboards.yml
            │   └── wazuh.yml
            ├── wazuh_cluster/
            │   ├── wazuh_manager.conf
            │   └── wazuh_worker.conf
            └── wazuh_indexer/
                ├── internal_users.yml         # git-ignored (real bcrypt hashes — local only)
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

# Generated certificate directory
config/wazuh_indexer_ssl_certs/
**/config/wazuh_manager_certs/
**/config/wazuh_dashboard_certs/
# internal_users.yml — real bcrypt hashes must never be committed
config/wazuh_indexer/internal_users.yml
config/wazuh_dashboard/wazuh.yml
# Persistent data & logs
data/
logs/
os-data*/
wazuh-data*/
```

---

# Step 2 — Secrets and Environment Configuration

## Create the External Docker Network

Dokploy normally creates `dokploy-network` during its own installation. The command below is a safety net: it creates the network only if it does not already exist, and the `|| true` makes it harmless to run when the network is already present.

```bash
docker network create dokploy-network || true
```

> [!NOTE] If Dokploy is already installed and running, this network almost certainly exists — running the command will simply report that and continue. Do not delete this network during teardown: it is shared by Dokploy and by any other module deployed on the same host.

## 2.1 Create the Secrets File

```bash
sudo mkdir -p /etc/dokploy/secrets
sudo chmod 700 /etc/dokploy/secrets
sudo nano /etc/dokploy/secrets/wazuh.env
```

> [!CAUTION] Replace all values between `< >` with your own values before saving in case of change (see [Secret Rotation Guide](02-secret-rotation.md)). **Never commit this file.**

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
# Wazuh default value for first deployment

INDEXER_USERNAME=admin
INDEXER_PASSWORD=SecretPassword
API_USERNAME=wazuh-wui
API_PASSWORD=MyS3cr37P450r.*-
DASHBOARD_USERNAME=kibanaserver
DASHBOARD_PASSWORD=kibanaserver

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

> [!CAUTION] **Factory credentials are a silent failure mode**
> 
> The values shown above (`SecretPassword`, `MyS3cr37P450r.*-`, `kibanaserver`) are the **default factory credentials** shipped by the official `wazuh-docker` project. They are published and identical on every default install — treat them as **public**.
> 
> A stack deployed with these values **starts cleanly, reports `green`, and logs no error**. Nothing in the logs signals that the cluster is protected only by well-known passwords. This is the single most dangerous failure mode of this deployment because it looks healthy.
> 
> These defaults are acceptable **only** for the very first bring-up, to validate that the stack assembles. Before the Dashboard is ever exposed through Traefik, rotate **all three** credentials to strong unique values:
> 
> - `INDEXER_PASSWORD` (replaces `SecretPassword`)
> - `API_PASSWORD` (replaces `MyS3cr37P450r.*-`)
> - `DASHBOARD_PASSWORD` (replaces `kibanaserver`)
> 
> Rotation is not a single-file edit: each password has a **plaintext half** (injected from `wazuh.env`) and a **bcrypt-hash half** (in `internal_users.yml`, applied via `securityadmin.sh`). Changing only `wazuh.env` leaves the factory hash live. Follow the full procedure in the [Secret Rotation Guide](02-secret-rotation.md), then confirm with the Default Credentials Guard in the [Health Check Guide](04-health-check.md) — `admin:SecretPassword` must return `401`.

> [!NOTE] **No hash generation is required for the initial deploy**
> 
> The `internal_users.yml` shipped by the official `wazuh-docker` repository already contains the **factory bcrypt hashes** matching the factory passwords above. For the first bring-up you deploy this file as-is — there is nothing to generate. Hash generation (`hash.sh`) becomes necessary only when you change a password, because the hash half must then be regenerated to match. That procedure lives in the [Secret Rotation Guide](02-secret-rotation.md), which is the single place where credential changes — plaintext and hash together — are documented.

## 2.2 Environment File Precedence — `wazuh.env` Is the Single Source of Truth

This deployment loads credentials exclusively through `--env-file /etc/dokploy/secrets/wazuh.env` in the Run Command (see §5.3). That file is the single source of truth for all runtime passwords.

> [!IMPORTANT] **Keep the working-directory `.env` empty or absent**
> 
> Docker Compose automatically reads a `.env` file located in the working directory (here `/home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/.env`) if one is present. Compose variable precedence is:
> 
> ```text
> --env-file (explicit)   >   working-directory .env (implicit)   >   shell environment
> ```
> 
> Because `--env-file` wins, an out-of-date working-directory `.env` will **not** override `wazuh.env` for variable interpolation. However, a stale `.env` left in the working directory is still a trap: it is easy to edit the wrong file during a later rotation and believe a credential changed when it did not.
> 
> **Reference rule for this blueprint:** keep the working-directory `.env` empty or absent, and treat `/etc/dokploy/secrets/wazuh.env` as the only file you ever edit for credentials.
> 
> ```bash
> # Verify no stale .env shadows the secrets file
> ls -la /home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/.env 2>/dev/null \
>   && echo "WARNING: a working-dir .env exists — ensure it is empty or remove it"
> ```

> [!NOTE] If you previously relied on a working-directory `.env` and see `invalid username or password` at the Dashboard despite a running stack, the cause is almost always credential drift between that file and `wazuh.env`. Consolidate onto `wazuh.env` and redeploy. Credential rotation is covered end-to-end in the [Secret Rotation Guide](02-secret-rotation.md).

## 2.3 Back Up Original Configuration Files

```bash
sudo mkdir -p /etc/dokploy/secrets/configs

cd /home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/

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
> - Configuration file changes
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

> [!NOTE] **Session expiry and "Invalid credentials" after logout**
> 
> If the Dashboard returns `Invalid credentials` after disconnect/reconnect without any API-side error, the cause is a stale session cookie not properly invalidated client-side. Clear browser cookies for the domain or test in a private window.

> [!NOTE] This blueprint deploys a single Dashboard instance. For Dashboard-layer HA, see [Architecture Decision Records](05-architecture-decision-records.md).

## 3.2 Volume Strategy — Named Volumes for Data, Absolute Bind Mounts for Configuration

This blueprint applies a hybrid volume strategy:

- **Named Docker volumes** — persistent runtime data that must survive container recreation
- **Absolute bind mounts** — certificates and configuration files requiring direct host-level editing

### Why absolute paths instead of relative paths

The official Wazuh Docker compose uses relative paths (`./config/...`), which work when `docker compose` is run directly from the compose file directory.

**Dokploy does not run `docker compose` from the compose file directory**, which causes relative paths to fail with:

```
mount src=...: not a directory
```

> [!WARNING] **Known failure mode**
> 
> If the host path does not exist or Docker interprets a file path as a directory, the stack will fail with:
> 
> ```
> not a directory: Are you trying to mount a directory onto a file
> ```
> 
> This typically occurs when:
> 
> - The path in the compose file does not exist on the host
> - Dokploy regenerates the service path (APP_ID change)
> - The repository is re-cloned to a different location
> 
> Always verify that all bind mount source paths exist on the host before deploying.

### Indexer volume mounts

```yaml
# wazuh1.indexer — admin certs included for securityadmin.sh
volumes:
  - wazuh-indexer-data-1:/var/lib/wazuh-indexer
  - /home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/config/wazuh_indexer_ssl_certs/root-ca.pem:/usr/share/wazuh-indexer/config/certs/root-ca.pem
  - /home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/config/wazuh_indexer_ssl_certs/wazuh1.indexer-key.pem:/usr/share/wazuh-indexer/config/certs/wazuh1.indexer.key
  - /home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/config/wazuh_indexer_ssl_certs/wazuh1.indexer.pem:/usr/share/wazuh-indexer/config/certs/wazuh1.indexer.pem
  - /home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/config/wazuh_indexer_ssl_certs/admin.pem:/usr/share/wazuh-indexer/config/certs/admin.pem
  - /home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/config/wazuh_indexer_ssl_certs/admin-key.pem:/usr/share/wazuh-indexer/config/certs/admin-key.pem
  - /home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/config/wazuh_indexer/wazuh1.indexer.yml:/usr/share/wazuh-indexer/config/opensearch.yml
  - /home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/config/wazuh_indexer/internal_users.yml:/usr/share/wazuh-indexer/config/opensearch-security/internal_users.yml

# wazuh2.indexer and wazuh3.indexer — no admin certs (least privilege)
volumes:
  - wazuh-indexer-data-2:/var/lib/wazuh-indexer
  - /home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/config/wazuh_indexer_ssl_certs/root-ca.pem:/usr/share/wazuh-indexer/config/certs/root-ca.pem
  - /home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/config/wazuh_indexer_ssl_certs/wazuh2.indexer-key.pem:/usr/share/wazuh-indexer/config/certs/wazuh2.indexer.key
  - /home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/config/wazuh_indexer_ssl_certs/wazuh2.indexer.pem:/usr/share/wazuh-indexer/config/certs/wazuh2.indexer.pem
  - /home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/config/wazuh_indexer/wazuh2.indexer.yml:/usr/share/wazuh-indexer/config/opensearch.yml
  - /home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/config/wazuh_indexer/internal_users.yml:/usr/share/wazuh-indexer/config/opensearch-security/internal_users.yml
```

> [!NOTE] `admin.pem` and `admin-key.pem` are mounted only on `wazuh1.indexer` because `securityadmin.sh` is executed from that node exclusively. Mounting admin credentials on wazuh2 and wazuh3 would violate least privilege.

### Manager and Worker volume mounts

```yaml
# wazuh.master and wazuh.worker — SSL certs for Filebeat
volumes:
  - /home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/config/wazuh_indexer_ssl_certs/root-ca-manager.pem:/etc/ssl/root-ca.pem
  - /home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/config/wazuh_indexer_ssl_certs/wazuh.master.pem:/etc/ssl/filebeat.pem
  - /home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/config/wazuh_indexer_ssl_certs/wazuh.master-key.pem:/etc/ssl/filebeat.key
  - /home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/config/wazuh_cluster/wazuh_manager.conf:/wazuh-config-mount/etc/ossec.conf
```

### Dashboard volume mounts

```yaml
# wazuh.dashboard
volumes:
  - /home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/config/wazuh_indexer_ssl_certs/wazuh.dashboard.pem:/usr/share/wazuh-dashboard/certs/wazuh-dashboard.pem
  - /home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/config/wazuh_indexer_ssl_certs/wazuh.dashboard-key.pem:/usr/share/wazuh-dashboard/certs/wazuh-dashboard-key.pem
  - /home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/config/wazuh_indexer_ssl_certs/root-ca.pem:/usr/share/wazuh-dashboard/certs/root-ca.pem
  - /home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/config/wazuh_dashboard/opensearch_dashboards.yml:/usr/share/wazuh-dashboard/config/opensearch_dashboards.yml
  - /home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/config/wazuh_dashboard/wazuh.yml:/usr/share/wazuh-dashboard/data/wazuh/config/wazuh.yml
  - wazuh-dashboard-config:/usr/share/wazuh-dashboard/data/wazuh/config
  - wazuh-dashboard-custom:/usr/share/wazuh-dashboard/plugins/wazuh/public/assets/custom
```

### Named volumes declaration

Declare all named volumes at the bottom of `docker-compose.yml`:

```yaml
volumes:
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


 Check other modifications like password secrets in the docker-compose model 
---

# Step 4 — Certificate Generation and Permissions

## 4.1 Generate Certificates

```bash
cd /home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/

# Generate certificates using the official temporary container
docker compose -f generate-indexer-certs.yml run --rm generator
```

Certificates are generated directly into `./config/wazuh_indexer_ssl_certs/`.

Verify that all certificates share the same Certificate Authority — **critical**:

```bash
for cert in config/wazuh_indexer_ssl_certs/*.pem; do
  if openssl x509 -in "$cert" -noout -issuer >/dev/null 2>&1; then
    echo "$cert"
    openssl x509 -in "$cert" -noout -issuer
  fi
done
# All certificates must return: issuer=CN=root-ca
# A different issuer means the trust chain is broken
# and OpenSearch authentication will fail
```

## 4.2 Set Certificate Permissions

Certificates are mounted into containers via bind mounts from the host directory. No volume injection step is required.

**Verify the UID used by the indexer container:**

```bash
# Never assume UID 1000 — always verify dynamically
docker run --rm wazuh/wazuh-indexer:4.14.5 id
# → uid=1000(wazuh) gid=1000(wazuh)
```

**Set ownership and permissions:**

```bash
sudo chown -R 1000:1000 \
  /home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/config/wazuh_indexer_ssl_certs/

sudo chmod 640 \
  /home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/config/wazuh_indexer_ssl_certs/*.pem
```

> [!WARNING] Incorrect ownership is the most common cause of silent `securityadmin.sh` failures. Files owned by `root:root` without read access for UID 1000 will cause OpenSearch Security initialization to fail without a clear error message.

**Back up certificates:**

```bash
sudo cp -r \
  /home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/config/wazuh_indexer_ssl_certs/ \
  /etc/dokploy/secrets/certs-backup-$(date +%Y%m%d)/
```

> [!CAUTION] Do **not** delete `config/wazuh_indexer_ssl_certs/` from the working directory. Certificates are served directly from this directory via bind mounts — removing them will break the stack on next restart.



---

# Step 5 — Git Synchronization and Dokploy Configuration

> [!CAUTION] **Certificates must already exist on the host before deploying**
> 
> Do not start this step until Step 4 has generated every `.pem` into `config/wazuh_indexer_ssl_certs/`. If you deploy first, Docker finds the bind-mount source paths missing and **creates them as empty directories** — `admin.pem`, `root-ca.pem`, and the rest become folders instead of files. The mount then "succeeds" onto a directory, nothing can write the real file over it, and the stack fails with `not a directory: Are you trying to mount a directory onto a file`. This is why certificate generation (Step 4) precedes deployment, matching the official Wazuh documentation order.
> 
> ```bash
> # Confirm each cert is a FILE, not a directory, before deploying
> for f in config/wazuh_indexer_ssl_certs/*.pem; do
>   [ -f "$f" ] && echo "OK file: $f" || echo "PROBLEM (missing or dir): $f"
> done
> ```

## 5.1 Push the Repository to GitHub

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

## 5.2 Create the Application in Dokploy

1. Create a new project — **Wazuh-Stack**
2. Add a **Docker Compose** service
3. Connect the GitHub repository
4. Set the Compose path:

```text
./01-Wazuh-Stack/wazuh-docker/docker-compose.yml
```

## 5.3 Environment Variable Loading Strategy

> [!IMPORTANT] **Why `--env-file` in the Run Command rather than `env_file` in `docker-compose.yml`**
> 
> Passing `--env-file` via the Run Command ensures that all variables are fully resolved by Docker Compose **before** Wazuh and OpenSearch entrypoint scripts execute.
> 
> When `env_file` is defined inside `docker-compose.yml`, variables may not be parsed in time for internal OpenSearch and Wazuh configuration scripts — a common cause of silent initialization failures in multi-node clusters.
> 
> The Run Command method is therefore the reference pattern for this deployment.

### Scope of `--env-file` injection

`--env-file` only resolves `${VAR}` references that appear in a service's `environment:` block in `docker-compose.yml`. It does **not** reach into bind-mounted configuration files (`opensearch_dashboards.yml`, `wazuh.yml`, `internal_users.yml`) — those carry literal values or bcrypt hashes that must be edited directly on the host.

| Component | What `--env-file` drives | Literal / hash half (not driven by `--env-file`) |
|---|---|---|
| `wazuh.master` / `wazuh.worker` | `${INDEXER_*}`, `${API_*}` referenced in `environment:` — **direct** | — |
| `wazuh.dashboard` | `${API_PASSWORD}` (→ `wazuh-wui`) and `${DASHBOARD_PASSWORD}` (→ `kibanaserver`) in `environment:` — **direct** | bcrypt hash of `kibanaserver` in `internal_users.yml` |
| `wazuh1/2/3.indexer` | nothing | all credentials via `internal_users.yml` + `securityadmin.sh` |

> [!IMPORTANT] **The invariant behind the factory-credentials trap**
> 
> Every OpenSearch credential has two halves: a **plaintext half** injected at runtime (driven by `--env-file`) and a **bcrypt-hash half** stored in `internal_users.yml` and applied by `securityadmin.sh` (never driven by `--env-file`). Rotating only `wazuh.env` updates the first half and silently leaves the second at its factory value. Both halves must be changed together — see §5.6 and the [Secret Rotation Guide](02-secret-rotation.md). For the full rationale, see [ADR-002](05-architecture-decision-records.md).

## 5.4 Configure the Run Command

In Dokploy → **Advanced → Run Command**, paste the following on a **single line**:

```
compose -p soccenter-wazuhstack --env-file /etc/dokploy/secrets/wazuh.env -f ./01-Wazuh-Stack/wazuh-docker/docker-compose.yml up -d --remove-orphans
```

> [!NOTE] The Run Command must be on a **single line** — Dokploy does not support line continuations with `\`. Multi-line commands will fail silently.

> [!NOTE] The `-p soccenter-wazuhstack` flag controls the Docker project name and therefore the prefix of all named volumes, containers, and networks. Changing this value later requires volume migration.

## 5.5 Initial Deploy

Click **Deploy** in the Dokploy UI and wait for the containers to start.

```bash
# Verify containers are running
docker ps | grep wazuh
```

> [!NOTE] Containers may stop or report errors on this first run — this is expected. Docker creates the named volumes before they are initialized. The message `Error response from daemon: No such container` is a Dokploy UI polling artifact and can be safely ignored if containers are running.

Wait for the cluster to report `green` before initializing the security plugin:

```bash
docker exec wazuh1.indexer curl -k -s \
  -u admin:<INDEXER_PASSWORD> \
  https://localhost:9200/_cluster/health?pretty | grep -E "cluster_name|status"
# Expected: "status" : "green"
```

## 5.6 Initialize the OpenSearch Security Plugin

> [!CAUTION] **This step is mandatory after every initial deployment or volume recreation.**
> 
> Without it, the indexer logs will loop on: `Not yet initialized (you may need to run securityadmin)`

> [!NOTE] On the initial deploy, `securityadmin.sh` applies the **factory hashes** already present in `internal_users.yml` — this is expected and required to initialize the security index. It does not change any credential. Rotating away from the factory values is a separate, mandatory step before exposure (see [Secret Rotation Guide](02-secret-rotation.md)).

Wait for the cluster to form:

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

> [!CAUTION] **`Done with success` alone is not proof of a secured cluster**
> 
> `securityadmin.sh` reports `Done with success` whenever it applies the security config — including when the applied `internal_users.yml` still contains the **factory hashes**. The message confirms the config was loaded, not that credentials were changed. A documented false-positive on `wazuh-docker` is to see `Done with success` while `admin:SecretPassword` still authenticates.
> 
> Always validate the actual credential state, not just the command exit:
> 
> ```bash
> # 1. The NEW password must authenticate → HTTP 200
> docker exec wazuh1.indexer curl -k -s -o /dev/null -w "%{http_code}\n" \
>   -u admin:<NEW_INDEXER_PASSWORD> \
>   https://localhost:9200/_cluster/health
> # Expected: 200
> 
> # 2. The FACTORY password must be rejected → HTTP 401
> docker exec wazuh1.indexer curl -k -s -o /dev/null -w "%{http_code}\n" \
>   -u admin:SecretPassword \
>   https://localhost:9200/_cluster/health
> # Expected: 401  →  a 200 here means the factory credential is still live (critical)
> ```
> 
> A `200` on the second command means the rotation did not take: the bind-mounted `internal_users.yml` still holds the factory hash, or `securityadmin.sh` ran against a stale config directory. Re-check the hash in `config/wazuh_indexer/internal_users.yml` and re-run. This same guard is formalized as Check 0 in the [Health Check Guide](04-health-check.md).
> 
> If `securityadmin.sh` exits without `Done with success` and without a clear error, the cause is almost always incorrect file ownership on the certificate bind mount.
> 
> ```bash
> # Verify UID inside the container
> docker exec wazuh1.indexer id
> 
> # Verify certificate ownership
> docker exec wazuh1.indexer ls -la /usr/share/wazuh-indexer/config/certs/
> # → must show 1000:1000, not root:root
> 
> # Fix ownership on the host directory
> sudo chown -R 1000:1000 \
>   /home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/config/wazuh_indexer_ssl_certs/
> ```

## 5.7 Configure the Public Domain

> [!IMPORTANT] The domain can only be configured **after** the initial deploy. Dokploy needs the containers to be running to detect available services in the **Service Name** dropdown.

In Dokploy → **Domains → Add Domain**:

1. **Service Name** → select `wazuh.dashboard` from the dropdown
2. **Host** → `wazuh.your-domain.com`
3. **Port** → `5601`
4. **HTTPS** → Enabled (automatic Let's Encrypt via Traefik)

Then click **Redeploy** to apply the domain configuration.


---

# Step 6 — Post-Deployment Validation

See [Health Check Guide](04-health-check.md) for the complete validation procedure.

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
git tag -l

git checkout v4.14.4-prod-20240901
git push -f origin main
# Then click Deploy in Dokploy
```

## Rollback via Version Change

```bash
sudo nano /etc/dokploy/secrets/wazuh.env
# Revert WAZUH_VERSION, WAZUH_IMAGE_VERSION, FILEBEAT_TEMPLATE_BRANCH
# Then click Deploy in Dokploy
```

> [!CAUTION] A major version rollback may cause OpenSearch index incompatibilities. Always test rollbacks on a staging environment first.

---

# Step 8 — Production Hardening

## P1 — Before go-live

|Item|Why|
|---|---|
|Encrypted offsite backup of `/etc/dokploy/secrets/`|Secrets are the single point of recovery|
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

For design rationale on accepted limitations, see [Architecture Decision Records](05-architecture-decision-records.md).

---

## Deployment Maturity

|Level|Status|
|---|---|
|Production-oriented reference architecture|✅|
|GitOps-ready|✅|
|Small-team / Lab ready|✅|
|Enterprise-hardened|⚠️ P1/P2 hardening required|
|Multi-region|❌ Out of scope|

---

## Disclaimer

This project is provided as-is and is not affiliated with, endorsed by, or supported by Wazuh Inc.

Users are responsible for reviewing, adapting, and validating all configurations and procedures in their own environments before production use.

---

© 2026 Kevin YAKPOVI · Security Engineer · Open Source SOC Builder

OpenSOC GitOps Dokploy Blueprint — https://github.com/Kev1-alt/OpenSOC-GitOps-Dokploy

Documentation licensed under Creative Commons Attribution 4.0 International (CC BY 4.0). Source code and infrastructure examples licensed under Apache License 2.0.

This document is part of the OpenSOC GitOps Documentation Suite.
