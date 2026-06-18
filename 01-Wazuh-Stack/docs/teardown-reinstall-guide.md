# Teardown & Clean Reinstall Runbook — Wazuh Multi-Node Stack

|**Version**|1.0|
|---|---|
|**Author**|Kevin YAKPOVI|
|**Last Updated**|June 2026|
|**Status**|Published|

---

## Document Purpose

### Objective

Provide a safe, repeatable procedure for completely removing a Wazuh Multi-Node deployment and reinstalling it from scratch — whether for a lab reset or for a controlled production recovery.

This runbook covers full destruction of:

- Project Docker volumes (runtime data)
- The Dokploy compose application directory
- The cloned repository working tree
- External secrets

It deliberately does **not** touch the Dokploy platform itself.

### Blueprint Approach

This guide is based on the OpenSOC GitOps reference architecture (v1 — absolute host paths). The reference project name is `soccenter-wazuhstack` and the reference repository root is `/home/user/OpenSOC-GitOps-Dokploy`. Replace both with your own values before executing any command.

### Documentation Suite

Part of the OpenSOC GitOps Documentation Suite.

| #      | Document                                                             | Description                      | Status           |
| ------ | -------------------------------------------------------------------- | -------------------------------- | ---------------- |
| 00     | [README](README.md)                                                  | Project overview and quick start | ✅ Production     |
| 01     | [Deployment Guide](01-deployment-guide.md)                           | Full deployment procedure        | ✅ Production     |
| 02     | [Secret Rotation Guide](02-secret-rotation.md)                       | Credentials rotation runbook     | ✅ Production     |
| 03     | [Troubleshooting Guide](03-troubleshooting.md)                       | Incident diagnosis runbook       | ✅ Production     |
| 04     | [Health Check Guide](04-health-check.md)                             | Post-deployment validation       | ✅ Production     |
| 05     | [Architecture Decision Records](05-architecture-decision-records.md) | Design rationale and trade-offs  | ✅ Production     |
| **06** | **Teardown & Clean Reinstall Runbook**                               | **This document**                | **✅ Production** |

### Intended Audience

|Role|Relevance|
|---|---|
|Security Engineers|Recovering or rebuilding a corrupted deployment|
|DevSecOps Engineers|Lab resets and reproducible from-scratch installs|
|DevOps Engineers|Executing destructive operations during maintenance windows|
|Platform Engineers|Understanding what is and is not safe to remove on a shared host|

### Prerequisites

- Wazuh Multi-Node deployed via Docker Compose and Dokploy
- Host access with `sudo`
- Basic Docker and Linux administration knowledge

---

## Security Notice

This procedure permanently deletes data, certificates, and secrets. The backup step is mandatory for any environment you may need to restore.

**Never commit** any artifact produced by the backup step below to Git.

> [!NOTE] **Validation status**
> 
> This runbook is validated primarily through full lab teardown-and-rebuild cycles on the reference environment. The production path — with the mandatory backup in Step 1 — is documented and follows the same mechanics, but should be rehearsed in a staging environment before being relied upon for a live recovery.

---

## ⚠️ Critical Safety Rules

> [!CAUTION] **Never run `docker volume prune` or `docker system prune --volumes`**
> 
> A blanket prune removes volumes belonging to **every** project on the host — including Dokploy's own platform volumes and any other module or stack. It is the single most dangerous command in a teardown. This runbook never uses it. Always target volumes by the project prefix, and always list before deleting.

> [!CAUTION] **Never delete the Dokploy platform volumes**
> 
> The following volumes are the Dokploy platform itself. Deleting any of them can destroy the Dokploy UI, its database, and all of its configuration — not just this stack:
> 
> ```text
> dokploy
> dokploy-postgres
> dokploy-redis
> ```
> 
> The filters in this runbook are written to match only the Wazuh project prefix and will never match these. Still, always read the listing output before confirming a deletion.

> [!NOTE] **Container addressing**
> 
> Dokploy prefixes project resources with the project name and a generated suffix. In the reference environment, volumes are named `soccenter-wazuhstack-a1b2c3_<volume>` and containers `soccenter-wazuhstack-a1b2c3-<service>-1`. Your suffix will differ — resolve the real prefix once and reuse it:
> 
> ```bash
> # Discover the actual project prefix from the running volumes
> PROJECT_PREFIX=$(docker volume ls --format '{{.Name}}' | grep -m1 'wazuh-indexer-data-1' | sed 's/_wazuh-indexer-data-1//')
> echo "Project prefix: $PROJECT_PREFIX"
> # Expected example: soccenter-wazuhstack-a1b2c3
> ```

---

## Teardown Order at a Glance

The order is chosen so that no step destroys a resource a later step depends on, and so that secrets survive until the very end:

```text
1. Back up secrets, certs, internal_users.yml      ← do this first, always
2. Stop the stack and remove project volumes       ← while secrets/compose still resolve --env-file
3. Sweep any residual project volumes (by prefix)  ← safety net
4. Remove the Dokploy compose app directory
5. Remove the cloned repository working tree
6. Remove the external secrets directory           ← last
```

---

## Step 1 — Back Up Before Destroying (Mandatory)

For any environment you might need to restore, archive the irreplaceable artifacts first: the secrets file, the certificates, and the active `internal_users.yml` (the only place the real bcrypt hashes live).

```bash
STAMP=$(date +%Y%m%d-%H%M%S)
BACKUP_DIR="/root/wazuh-teardown-backup-$STAMP"
sudo mkdir -p "$BACKUP_DIR"

# Secrets (env + any config snapshots already stored there)
sudo cp -a /etc/dokploy/secrets/ "$BACKUP_DIR/secrets/"

# Certificates and active internal_users.yml from the working tree
sudo cp -a \
  /home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/config/wazuh_indexer_ssl_certs/ \
  "$BACKUP_DIR/wazuh_indexer_ssl_certs/"

sudo cp -a \
  /home/user/OpenSOC-GitOps-Dokploy/01-Wazuh-Stack/wazuh-docker/config/wazuh_indexer/internal_users.yml \
  "$BACKUP_DIR/internal_users.yml"

# Lock down the backup
sudo chmod -R 600 "$BACKUP_DIR"
sudo chown -R root:root "$BACKUP_DIR"

echo "Backup written to $BACKUP_DIR"
ls -la "$BACKUP_DIR"
```

> [!WARNING] Store this backup off-host if the deployment is production. A backup that lives only on the server you are about to rebuild is not a backup. Encrypt it before moving it (e.g. `tar` + `gpg`).

> [!NOTE] For a pure lab reset where you intend to regenerate everything from defaults, you may skip this step. If there is any doubt, do not skip it.

---

## Step 2 — Stop the Stack and Remove Project Volumes

The cleanest removal uses `docker compose down -v`, scoped to the project by `-p`. This works **only while the secrets file and compose file are still present**, because `--env-file` must resolve. This is why secrets are deleted last (Step 6), not here.

```bash
cd /home/user/OpenSOC-GitOps-Dokploy

docker compose -p soccenter-wazuhstack
  --env-file /etc/dokploy/secrets/wazuh.env \
  -f ./01-Wazuh-Stack/wazuh-docker/docker-compose.yml \
  down -v --remove-orphans
```

`down -v` removes the containers, the project network, and the named volumes **declared in this compose project** — nothing outside it.

> [!TIP] If you prefer the Dokploy UI: use **Stop**, then **Delete** on the application. Note that the UI delete may not remove the named volumes; verify with Step 3 and sweep any that remain.

> [!NOTE] **If a volume reports `volume is in use`**
> 
> A volume cannot be removed while any container — even a stopped one — still references it. Force-remove the project containers first, then proceed:
> 
> ```bash
> docker ps -a | grep wazuh
> docker rm -f $(docker ps -aq --filter "name=wazuh")
> ```
> 
> If a specific volume is still blocked, find the holding container and remove it:
> 
> ```bash
> docker ps -a --filter volume=<volume_name>
> docker rm -f <container_id>
> ```

---

## Step 3 — Sweep Residual Project Volumes (Safety Net)

If `down -v` was interrupted, or you are cleaning up after a UI delete, remove any leftover volumes **by project prefix only**. Always list first.

```bash
# Resolve the prefix if not already set
PROJECT_PREFIX=$(docker volume ls --format '{{.Name}}' | grep -m1 'wazuh-indexer-data-1' | sed 's/_wazuh-indexer-data-1//')

# 1. LIST — read this output carefully before deleting anything
docker volume ls --filter "name=${PROJECT_PREFIX}_" --format '{{.Name}}'

# 2. Confirm NONE of the lines above is a dokploy* volume, then delete
docker volume ls --filter "name=${PROJECT_PREFIX}_" -q | xargs -r docker volume rm
```

> [!CAUTION] Inspect the listing in step 1 before running step 2. If `PROJECT_PREFIX` came back empty (e.g. the volumes were already gone), the filter `name=_` could match unintended volumes. If the variable is empty, stop and set the prefix manually — do not run the delete with an empty prefix.

```bash
# Guard: refuse to proceed on an empty prefix
if [ -z "$PROJECT_PREFIX" ]; then
  echo "PROJECT_PREFIX is empty — set it manually, do not delete blindly"
else
  echo "Prefix OK: $PROJECT_PREFIX"
fi
```

---

## Step 4 — Remove the Dokploy Compose Application Directory

Dokploy stores the synced compose application under `/etc/dokploy/compose/<APP_ID>/`. Removing it clears the generated working tree and any stale `.env` Dokploy materialized there.

```bash
# Identify the APP_ID directory (there is usually one per application)
ls -la /etc/dokploy/compose/

# Remove the specific application directory (replace <APP_ID>)
sudo rm -rf /etc/dokploy/compose/<APP_ID>/
```

> [!WARNING] Remove only the `<APP_ID>` directory that belongs to this stack. Do not remove `/etc/dokploy/compose/` itself or directories belonging to other applications.

---

## Step 5 — Remove the Cloned Repository Working Tree

```bash
sudo rm -rf /home/user/OpenSOC-GitOps-Dokploy/
```

> [!NOTE] This deletes the local working tree only. Your GitHub remote is untouched — a clean reinstall re-clones from there.

---

## Step 6 — Remove the External Secrets (Last)

Delete the secrets only after the volumes and compose have been removed, so that nothing still depends on `--env-file`.

```bash
# Final confirmation — these are the credentials and config snapshots
sudo ls -la /etc/dokploy/secrets/

sudo rm -rf /etc/dokploy/secrets/
```

> [!CAUTION] This is irreversible. If you skipped Step 1, the `admin`, `wazuh-wui`, and `kibanaserver` credentials and the real bcrypt hashes are gone for good. Confirm your backup exists before running this.

---

## Step 7 — Verify a Clean State

Before reinstalling, confirm nothing from the project remains — and that Dokploy is intact.

```bash
# No project volumes left
docker volume ls --format '{{.Name}}' | grep wazuh || echo "OK: no wazuh volumes"

# Dokploy platform volumes MUST still be present
docker volume ls --format '{{.Name}}' | grep -E '^dokploy' \
  && echo "OK: Dokploy platform volumes intact"

# No project containers left
docker ps -a --format '{{.Names}}' | grep wazuh || echo "OK: no wazuh containers"

# Secrets and repo gone
ls -la /etc/dokploy/secrets/ 2>/dev/null || echo "OK: secrets removed"
ls -la /home/user/OpenSOC-GitOps-Dokploy/ 2>/dev/null || echo "OK: repo removed"
```

Expected: every line reports `OK`, and the Dokploy platform volumes (`dokploy`, `dokploy-postgres`, `dokploy-redis`) are still listed.

---

## Step 8 — Reinstall From Scratch

The host is now in the same state as a fresh server (minus Dokploy, which is intact). Restart the full deployment procedure from the beginning:

→ [Deployment Guide — Step 1](01-deployment-guide.md)

Reinstall sequence reminder (full detail in the Deployment Guide):

```text
1. Re-clone and prepare the repository        (Deployment Step 1)
2. Recreate /etc/dokploy/secrets/wazuh.env    (Deployment Step 2)
3. Configure dashboard + volume mounts        (Deployment Step 3)
4. Generate certificates BEFORE deploying     (Deployment Step 4)  ← prevents .pem-as-directory corruption
5. Deploy via Dokploy + run securityadmin.sh  (Deployment Step 5)
6. Validate, starting with the Default Credentials Guard  (Health Check — Check 0)
```

> [!IMPORTANT] If you restored secrets and certificates from the Step 1 backup instead of regenerating them, run the Default Credentials Guard ([Health Check — Check 0](04-health-check.md)) to confirm the restored credentials are the rotated ones and not factory defaults.

---

## Full Teardown — Lab Reset (Condensed Operational Sequence)

For a lab environment where the backup step (Step 1) is intentionally skipped, use the condensed sequence below. Unlike a single copy-paste block, it includes the container-removal and volume-conflict handling that a real teardown requires.

**1. Stop the stack from Dokploy** (application → Deploy Settings → **Stop**).

**2. Remove any remaining project containers** (volumes cannot be deleted while a container still uses them):

```bash
docker ps -a | grep wazuh
docker rm -f $(docker ps -aq --filter "name=wazuh")
```

**3. Confirm the project prefix:**

```bash
docker volume ls | grep wazuh
```

**4. Remove project volumes (by prefix only):**

```bash
docker volume ls --filter "name=<project_prefix>_" -q | xargs -r docker volume rm
```

If a volume reports `volume is in use`, find and remove the holding container, then retry:

```bash
docker ps -a --filter volume=<volume_name>
docker rm -f <container_id>
```

**5. Remove the Dokploy application state:**

```bash
sudo rm -rf /etc/dokploy/compose/<APP_ID>/
```

**6. Remove the repository working tree:**

```bash
sudo rm -rf /home/user/OpenSOC-GitOps-Dokploy/
```

**7. Remove external secrets:**

```bash
sudo rm -rf /etc/dokploy/secrets/
```

**8. Verify cleanup:**

```bash
docker ps -a | grep wazuh
docker volume ls | grep <project_prefix>
```

Expected: no Wazuh-related resources remain, and the Dokploy platform volumes (`dokploy`, `dokploy-postgres`, `dokploy-redis`) are still present.

Never substitute any of the above with `docker volume prune`.

---

© 2026 Kevin YAKPOVI · Security Engineer · Open Source SOC Builder

OpenSOC GitOps Dokploy Blueprint — https://github.com/Kev1-alt/OpenSOC-GitOps-Dokploy

Documentation licensed under Creative Commons Attribution 4.0 International (CC BY 4.0). Source code and infrastructure examples licensed under Apache License 2.0.

This document is part of the OpenSOC GitOps Documentation Suite.
