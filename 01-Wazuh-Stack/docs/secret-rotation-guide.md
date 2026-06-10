# Secret Rotation Guide — Wazuh Multi-Node Stack

|**Version**|1.0|
|---|---|
|**Author**|Kevin YAKPOVI|
|**Last Updated**|June 2026|
|**Status**|Production|

---

## Document Purpose

### Objective

Provide a structured procedure for rotating credentials on a Wazuh Multi-Node deployment without service interruption where possible, and with minimal downtime where a restart is required.

### Blueprint Approach

This guide is based on the OpenSOC GitOps reference architecture.

Actual file locations, project names, Docker Compose project identifiers, and deployment paths may differ depending on local implementation choices.

Commands and file paths shown throughout this document should be treated as reference procedures and adapted to match the target environment.

### Documentation Suite

Part of the **OpenSOC GitOps Documentation Suite**.

| #      | Document                                       | Description                      | Status           |
| ------ | ---------------------------------------------- | -------------------------------- | ---------------- |
| 00     | [README](../README.md)                         | Project overview and quick start | ✅ Production     |
| 01     | [Deployment Guide](01-deployment-guide.md)     | Full deployment procedure        | ✅ Production     |
| **02** | **Secret Rotation Guide**                      | **This document**                | **✅ Production** |
| 03     | [Troubleshooting Guide](03-troubleshooting.md) | Incident diagnosis runbook       | ✅ Production     |
| 04     | [Health Check Guide](04-health-check.md)       | Post-deployment validation       | ✅ Production     |
| 05     | [Architecture Decision Records](05-adr.md)     | Design rationale and trade-offs  | ✅ Production     |

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

## Warnings

> [!CAUTION] Always back up `/etc/dokploy/secrets/` before any secret rotation. This directory is the source of truth for all credentials. Files present in Docker volumes may be recreated or replaced during maintenance.

> [!WARNING] Some secret rotations require a Dashboard or Manager restart. Perform these operations during a maintenance window appropriate for your environment.

---

## Operational Impact

|Secret|Affected Component|Expected Impact|
|---|---|---|
|`admin` (Indexer)|OpenSearch cluster|No service interruption|
|`kibanaserver`|Dashboard|Dashboard reconnection required after deployment|
|`wazuh-wui`|Dashboard + Manager|Dashboard and Manager restart required|

---

## Prerequisites — Configuration Backup

Before any secret rotation, retain a reference copy of all configuration files.

```bash
sudo mkdir -p /etc/dokploy/secrets/configs

cd ~/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/

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
cd ~/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/

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

```bash
sudo nano /etc/dokploy/secrets/wazuh.env
# Update: INDEXER_PASSWORD=<NEW_PASSWORD>
```

Then click **Deploy** in Dokploy to propagate to the Dashboard and Manager.

---

# 2. Rotating the API Password (`wazuh-wui`)

**Impact:** Dashboard and Manager restart required — schedule a maintenance window.

```bash
# ============================================================
# wazuh-wui PASSWORD ROTATION
# Stack: Wazuh Multi-Node / Docker Compose / Dokploy
# Author: Kevin YAKPOVI
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
# Without this step, the Dashboard loses API access after restart
cd ~/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/

sudo sed -i 's/password: ".*"/password: "<NEW_PASSWORD>"/' \
  config/wazuh_dashboard/wazuh.yml

# Verify the change
grep "password" config/wazuh_dashboard/wazuh.yml

# Update the cold backup
sudo cp config/wazuh_dashboard/wazuh.yml \
  /etc/dokploy/secrets/configs/wazuh.yml

# Update wazuh.env
sudo sed -i 's/API_PASSWORD=.*/API_PASSWORD=<NEW_PASSWORD>/' \
  /etc/dokploy/secrets/wazuh.env

# Step 5 — Restart in the correct order
sudo docker restart wazuh.master
sleep 10
sudo docker restart wazuh.dashboard

# Step 6 — Final verification
sudo docker exec wazuh.master curl -k -s \
  -u wazuh-wui:<NEW_PASSWORD> \
  -X POST https://localhost:55000/security/user/authenticate \
  | jq '.error // "Auth OK"'
```

> [!WARNING] Updating `config/wazuh_dashboard/wazuh.yml` at Step 4 is mandatory. This file is bind-mounted directly into the Dashboard container. Without this update, the Dashboard will lose access to the API after a restart.

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
cd ~/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/

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

```bash
sudo nano /etc/dokploy/secrets/wazuh.env
# Update: DASHBOARD_PASSWORD=<NEW_PASSWORD>
```

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
docker logs wazuh.dashboard --tail 50 | grep -iE "error|failed"
# Expected: no lines returned
```

## Manager

```bash
# Check for authentication errors toward the indexer
docker logs wazuh.master --tail 50 | grep -E "401|Unauthorized|failed"
# Expected: no lines returned
```

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

© 2026 **Kevin YAKPOVI** | Security Engineer · Open Source SOC Builder

🐙 [github.com/Kev1-alt](https://github.com/Kev1-alt) · 💼 [linkedin.com/in/kevin-yakpovi-384b4619a](https://linkedin.com/in/kevin-yakpovi-384b4619a)

*Published under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — Any reproduction must credit the original author.*
*Part of [OpenSOC GitOps Dokploy](https://github.com/Kev1-alt/OpenSOC-GitOps-Dokploy) · Document maintained at: `01-Wazuh-Stack/docs/[nom-du-fichier.md]`*`_`_
