# Secret Rotation Guide — Wazuh Multi-Node Stack

|**Version**|1.0|
|---|---|
|**Author**|Kevin YAKPOVI|
|**Last Updated**|June 2026|
|**Status**|Published|

---

## Document Purpose

### Objective

Provide a structured procedure for rotating credentials on a Wazuh Multi-Node deployment without service interruption where possible, and with minimal downtime where a restart is required.

### Blueprint Approach

This guide is based on the OpenSOC GitOps reference architecture (v1 — absolute host paths).

Actual file locations, project names, Docker Compose project identifiers, and deployment paths may differ depending on local implementation choices.

Commands and file paths shown throughout this document should be treated as reference procedures and adapted to match the target environment. The reference repository root used here is `/home/user/OpenSOC-GitOps-Dokploy` — replace it with your actual path.

### Documentation Suite

Part of the **OpenSOC GitOps Documentation Suite**.

| #      | Document                                                             | Description                              | Status           |
| ------ | -------------------------------------------------------------------- | ---------------------------------------- | ---------------- |
| 00     | [README](README.md)                                                  | Project overview and quick start         | ✅ Production     |
| 01     | [Deployment Guide](01-deployment-guide.md)                           | Full deployment procedure                | ✅ Production     |
| **02** | **Secret Rotation Guide**                                            | **This document**                        | **✅ Production** |
| 03     | [Troubleshooting Guide](03-troubleshooting.md)                       | Incident diagnosis runbook               | ✅ Production     |
| 04     | [Health Check Guide](04-health-check.md)                             | Post-deployment validation               | ✅ Production     |
| 05     | [Architecture Decision Records](05-architecture-decision-records.md) | Design rationale and trade-offs          | ✅ Production     |
| 06     | [Teardown & Clean Reinstall Runbook](06-teardown-reinstall.md)       | Full teardown and from-scratch reinstall | ✅ Production     |

### Audience

|Role|Relevance|
|---|---|
|Security Engineers|Owning the credential lifecycle and rotation schedule|
|DevSecOps Engineers|Integrating secret rotation into operational procedures|
|DevOps Engineers|Executing rotation procedures during maintenance windows|
|Platform Engineers|Adapting rotation procedures to organizational constraints|
|Startup Security Teams|Managing credentials on a lean operational model|

### Prerequisites

#### Technical prerequisites

|Requirement|Details|
|---|---|
|Wazuh Multi-Node|Deployed and running via Docker Compose and Dokploy|
|Dokploy|Operational and reachable|
|`jq`|Installed on the host — `sudo apt install jq`|
|Docker Engine|24+|

#### Required reading

| Document                                   | Read before this guide because...  |
| ------------------------------------------ | ---------------------------------- |
| [Deployment Guide](01-deployment-guide.md) | Stack must be deployed and running |

---

### Scope

**This document covers:**

- OpenSearch administrator account rotation (`admin`)
- OpenSearch Dashboard account rotation (`kibanaserver`)
- Wazuh API account rotation (`wazuh-wui`)
- Post-rotation validation for all components

**This document does NOT cover:**

| Out of scope              | See instead                                |
| ------------------------- | ------------------------------------------ |
| TLS certificate rotation  | Planned for a future guide                 |
| Wazuh cluster key renewal | Wazuh official documentation               |
| Agent token rotation      | Wazuh official documentation               |
| Initial deployment        | [Deployment Guide](01-deployment-guide.md) |

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

## Source of Truth for Credentials

A rotation touches a credential in several places. To avoid silent drift, partial rotation, or a Dashboard/Indexer mismatch, treat exactly one location per artifact as authoritative and treat the rest as derived copies.

|Artifact|Authoritative source|Derived / backup copies|
|---|---|---|
|Runtime passwords (`INDEXER_PASSWORD`, `API_PASSWORD`, `DASHBOARD_PASSWORD`)|`/etc/dokploy/secrets/wazuh.env` (loaded via `--env-file`)|Working-directory `.env` (`/home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/.env`) — **must stay empty or absent; never edited for rotation**|
|OpenSearch bcrypt hashes|Active bind-mounted file `config/wazuh_indexer/internal_users.yml` (applied via `securityadmin.sh`)|`/etc/dokploy/secrets/configs/internal_users.yml` — **cold backup snapshot only**|
|Configuration templates|Git repository|— (real secrets and real hashes are git-ignored and never committed)|

> [!CAUTION] **`wazuh.env` is the only file you should edit for credentials**
> 
> Docker Compose also auto-reads a `.env` in the working directory (`/home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/.env`). Because the Run Command passes `--env-file /etc/dokploy/secrets/wazuh.env` explicitly, `wazuh.env` wins variable precedence — but a stale working-directory `.env` is still a trap: it is easy to edit the wrong file and believe a credential rotated when it did not.
> 
> **Reference rule:** keep the working-directory `.env` empty or absent, and rotate credentials only in `/etc/dokploy/secrets/wazuh.env`. If your environment does keep a populated working-directory `.env`, re-sync it on every rotation — the procedures below include that step. See [Deployment Guide — §2.2](01-deployment-guide.md).

---

## How Credentials Are Stored — Two Halves per Password

Every OpenSearch-side credential exists in **two places that must always match**:

|Half|Where it lives|How it is changed|
|---|---|---|
|**Plaintext** (presented at connection time)|injected at runtime from `wazuh.env` via `--env-file` into the service `environment:`|edit `wazuh.env`|
|**Bcrypt hash** (verified by the indexer)|`config/wazuh_indexer/internal_users.yml`, bind-mounted into the indexers|edit the hash, then apply with `securityadmin.sh`|

`--env-file` only ever updates the **plaintext half**. It never touches `internal_users.yml`. A rotation that edits `wazuh.env` but not the hash leaves the cluster authenticating against the **old (or factory) hash** — and nothing in the logs flags it. This is why every indexer-side rotation below updates `internal_users.yml` **and** runs `securityadmin.sh`, then validates that the old/factory password is rejected.

> [!CAUTION] **Never leave factory hashes in place**
> 
> The default `wazuh-docker` install ships public factory credentials (`admin:SecretPassword`, `kibanaserver:kibanaserver`, `wazuh-wui:MyS3cr37P450r.*-`). Until both halves are rotated, the cluster is protected only by published passwords. After any rotation, confirm the factory value is dead — e.g. `admin:SecretPassword` must return `401` (see §5 and the [Health Check Guide](04-health-check.md) Check 0).

---

> [!NOTE] **Container addressing**
> 
> Commands address containers by their short service name (`wazuh1.indexer`, `wazuh.master`, `wazuh.dashboard`) — these are the container **hostnames**. On the host, Dokploy prefixes the real container names with the project name and a generated suffix (e.g. `soccenter-wazuhstack-a1b2c3-wazuh1.indexer-1`), so `docker exec wazuh1.indexer` will not resolve directly. Resolve the real name once per shell:
> 
> ```bash
> IDX1=$(docker ps --filter "name=wazuh1.indexer" --format '{{.Names}}' | head -1)
> # then: docker exec "$IDX1" ...
> ```
> 
> Treat the short names as placeholders to adapt to your environment.

---

## Warnings

> [!CAUTION] Always back up `/etc/dokploy/secrets/` before any secret rotation. This directory is the source of truth for all credentials. Files present in Docker volumes may be recreated or replaced during maintenance.

> [!WARNING] Some secret rotations require a Dokploy **Deploy or Redeploy** to propagate the new values. Perform these operations during a maintenance window appropriate for your environment.

---

## Operational Impact

|Secret|Affected Component|Expected Impact|
|---|---|---|
|`admin` (Indexer)|OpenSearch cluster|No service interruption|
|`kibanaserver`|Dashboard|Dashboard reconnection required after deployment|
|`wazuh-wui`|Dashboard + Manager|Dokploy Redeploy required (recreates Dashboard + Manager)|

---

## Prerequisites — Configuration Backup

Before any secret rotation, retain a reference copy of all configuration files.

```bash
sudo mkdir -p /etc/dokploy/secrets/configs

cd /home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/

sudo cp config/wazuh_indexer/internal_users.yml          /etc/dokploy/secrets/configs/
sudo cp config/wazuh_dashboard/opensearch_dashboards.yml /etc/dokploy/secrets/configs/
sudo cp config/wazuh_dashboard/wazuh.yml                 /etc/dokploy/secrets/configs/
sudo cp config/wazuh_cluster/wazuh_manager.conf          /etc/dokploy/secrets/configs/
sudo cp config/wazuh_cluster/wazuh_worker.conf           /etc/dokploy/secrets/configs/
```

---

# 1. Rotating the Indexer Password (OpenSearch `admin`)

**Impact:** No service interruption — change applied live.

## 1.1 Generate the new bcrypt hash

```bash
docker run --rm -it \
  -e JAVA_HOME=/usr/share/wazuh-indexer/jdk \
  wazuh/wazuh-indexer:4.14.5 \
  /usr/share/wazuh-indexer/plugins/opensearch-security/tools/hash.sh
```

> [!NOTE] The hash differs on every run — this is expected. bcrypt uses a random salt. Only verification against the plaintext password matters, not visual hash comparison.

## 1.2 Update both storage locations

`internal_users.yml` is mounted via bind mount from the repository working directory. Edit it directly on the host — no volume access is required.

```bash
cd /home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/

# Edit the active file (bind-mounted into all three indexer containers)
sudo nano config/wazuh_indexer/internal_users.yml

# Update the cold backup to keep both in sync
sudo cp config/wazuh_indexer/internal_users.yml \
  /etc/dokploy/secrets/configs/internal_users.yml
```

Replace the `hash:` value under the `admin:` user with the new hash in **both files**.

## 1.3 Apply live to the cluster

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

## 1.4 Propagate the new password

Update the authoritative secrets file:

```bash
sudo nano /etc/dokploy/secrets/wazuh.env
# Update: INDEXER_PASSWORD=<NEW_PASSWORD>
```

> [!NOTE] `wazuh.env` is the single source of truth, loaded via `--env-file`. Do not edit the working-directory `.env` — it must stay empty or absent (see Source of Truth above).

Then click **Deploy** in Dokploy to propagate to the Dashboard and Manager.

---

# 2. Rotating the API Password (`wazuh-wui`)

**Impact:** Dokploy Redeploy required (recreates Dashboard + Manager) — schedule a maintenance window.

```bash
# ============================================================
# wazuh-wui PASSWORD ROTATION
# Stack: Wazuh Multi-Node / Docker Compose / Dokploy
# ============================================================

# Step 1 — Retrieve the current admin JWT token
TOKEN=$(sudo docker exec wazuh.master bash -c \
  "curl -k -s -u wazuh-wui:<CURRENT_PASSWORD> \
  -X POST https://localhost:55000/security/user/authenticate" \
  | jq -r '.data.token')

echo "Token: $TOKEN"

# Step 2 — Find the ID associated with wazuh-wui
sudo docker exec wazuh.master curl -k -s \
  -X GET "https://localhost:55000/security/users" \
  -H "Authorization: Bearer $TOKEN" \
  | jq '.data.affected_items[] | {id, username}'

# Step 3 — Update the password via the API
# Replace <WAZUH_WUI_ID> with the ID returned in Step 2
sudo docker exec wazuh.master bash -c "curl -k -s \
  -X PUT 'https://localhost:55000/security/users/<WAZUH_WUI_ID>' \
  -H 'Authorization: Bearer $TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{\"password\": \"<NEW_PASSWORD>\"}'"

# Step 4 — Update wazuh.yml on the host (bind mount — critical)
# Without this step, the Dashboard loses API access after the redeploy
cd /home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/

sudo sed -i 's/password: ".*"/password: "<NEW_PASSWORD>"/' \
  config/wazuh_dashboard/wazuh.yml

# Verify the change
grep "password" config/wazuh_dashboard/wazuh.yml

# Update the cold backup
sudo cp config/wazuh_dashboard/wazuh.yml \
  /etc/dokploy/secrets/configs/wazuh.yml

# Update the authoritative secrets file
sudo nano /etc/dokploy/secrets/wazuh.env
# Update: API_PASSWORD=<NEW_PASSWORD>
# Note: wazuh.env is the single source of truth (--env-file).
# Do not edit the working-directory .env — it stays empty/absent.

# Step 5 — Click Deploy or Redeploy in Dokploy to propagate the new password

# Step 6 — Final verification (run after the deploy completes)
sudo docker exec wazuh.master curl -k -s \
  -u wazuh-wui:<NEW_PASSWORD> \
  -X POST https://localhost:55000/security/user/authenticate \
  | jq '.error // "Auth OK"'
```

> [!WARNING] Updating `config/wazuh_dashboard/wazuh.yml` at Step 4 is mandatory. This file is bind-mounted directly into the Dashboard container. Without this update, the Dashboard will lose access to the API after the deploy.

> [!WARNING] Updating `config/wazuh_dashboard/wazuh.yml` at Step 4 is mandatory. This file is bind-mounted directly into the Dashboard container. Without this update, the Dashboard will lose access to the API after the redeploy.

---

# 3. Rotating the Dashboard Password (`kibanaserver`)

**Impact:** Dashboard reconnection required after deployment.

## 3.1 Generate the new bcrypt hash

```bash
docker run --rm -it \
  -e JAVA_HOME=/usr/share/wazuh-indexer/jdk \
  wazuh/wazuh-indexer:4.14.5 \
  /usr/share/wazuh-indexer/plugins/opensearch-security/tools/hash.sh
```

## 3.2 Update `internal_users.yml`

`internal_users.yml` is mounted via bind mount — edit it directly on the host.

```bash
cd /home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/

# Edit the active file
sudo nano config/wazuh_indexer/internal_users.yml

# Update the cold backup
sudo cp config/wazuh_indexer/internal_users.yml \
  /etc/dokploy/secrets/configs/internal_users.yml
```

Replace the `hash:` value under the `kibanaserver:` user with the new hash in **both files**.

## 3.3 Apply via securityadmin

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

## 3.4 Propagate

Update the authoritative secrets file:

```bash
sudo nano /etc/dokploy/secrets/wazuh.env
# Update: DASHBOARD_PASSWORD=<NEW_PASSWORD>
```

> [!NOTE] `wazuh.env` is the single source of truth (`--env-file`). Do not edit the working-directory `.env` — it stays empty or absent.

Then trigger a new Dokploy deployment.

---

# 4. Appendix A — Special Cases

## Passwords Containing Special Characters

If the password contains special characters such as `!`, `$`, or backticks, store it in a variable using single quotes to avoid shell interpretation:

```bash
# Single quotes prevent bash from interpreting special characters
PWD='YOUR_PASSWORD_WITH_!_HERE'

TOKEN=$(sudo docker exec wazuh.master bash -c \
  "curl -k -s -u wazuh-wui:${PWD} \
  -X POST https://localhost:55000/security/user/authenticate" \
  | jq -r '.data.token')

echo "Token: $TOKEN"
```

Then resume at Step 2 of the `wazuh-wui` rotation procedure.

> [!CAUTION] Never use double quotes to assign a password containing `!` — bash will attempt to expand it as a history event and the variable will be empty or malformed.

---

# 5. Post-Rotation Validation

After each secret rotation, verify all components in the following order.

## OpenSearch

```bash
docker exec wazuh1.indexer curl -k -s \
  -u admin:<NEW_PASSWORD> \
  https://localhost:9200/_cluster/health?pretty
```

Expected:

```json
{
  "status": "green"
}
```

### Default Credentials Guard (factory passwords must be dead)

A rotation is only complete when the **old/factory** password stops working. Verify each rotated credential is rejected:

```bash
# Indexer admin — factory value must return 401
docker exec wazuh1.indexer curl -k -s -o /dev/null -w "%{http_code}\n" \
  -u admin:SecretPassword \
  https://localhost:9200/_cluster/health
# Expected: 401   (200 = factory hash still live → rotation incomplete)

# Wazuh API — factory value must return 401
sudo docker exec wazuh.master curl -k -s -o /dev/null -w "%{http_code}\n" \
  -u wazuh-wui:MyS3cr37P450r.*- \
  -X POST https://localhost:55000/security/user/authenticate
# Expected: 401
```

> [!CAUTION] A `200` (or a valid token) from any factory credential above means that credential's **hash half** was never rotated — `securityadmin.sh` ran against the old `internal_users.yml`, or only `wazuh.env` was edited. Re-run the corresponding rotation section and re-apply `securityadmin.sh`.

## API Manager

```bash
sudo docker exec wazuh.master curl -k -s \
  -u wazuh-wui:<NEW_PASSWORD> \
  -X POST https://localhost:55000/security/user/authenticate \
  | jq '.error // "Auth OK"'
```

Expected: `Auth OK`

## Dashboard

```bash
# Frontend connectivity — from inside the container
docker exec wazuh.dashboard curl -s \
  -o /dev/null -w "%{http_code}\n" \
  http://localhost:5601
# Expected: 200 or 302

# Check for authentication errors in logs
docker logs wazuh.dashboard --tail 50 | grep -iE "unauthorized|invalid credentials|authentication failed"
# Expected: no lines returned
```

## Manager

```bash
# Check for authentication errors toward the indexer
docker logs wazuh.master --tail 50 | grep -E "401|Unauthorized|authentication failed"
# Expected: no lines returned
```

> [!NOTE] These log checks deliberately target **authentication-level** errors (`401`, `Unauthorized`, `invalid credentials`), because those are the exact symptoms of an incomplete rotation. Unrelated INFO/WARN lines are not a concern here.

## Dokploy

Verify that the last deployment completed successfully:

```bash
# No container should be in a restart loop
docker ps --format "table {{.Names}}\t{{.Status}}" | grep -E "wazuh|Restarting"
# Expected: no lines containing "Restarting"
```

Also verify in the Dokploy UI:

- **Status:** Running on all containers
- **Deployments tab:** last deployment with no failure event

---

→ For a full stack health check after rotation, see [Health Check Guide](04-health-check.md)

---

© 2026 Kevin YAKPOVI · Security Engineer · Open Source SOC Builder

OpenSOC GitOps Dokploy Blueprint — https://github.com/Kev1-alt/OpenSOC-GitOps-Dokploy

Documentation licensed under Creative Commons Attribution 4.0 International (CC BY 4.0). Source code and infrastructure examples licensed under Apache License 2.0.

This document is part of the OpenSOC GitOps Documentation Suite.
