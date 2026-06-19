# Health Check Guide — Wazuh Multi-Node Stack

| **Version**      | 1.0           |
| ---------------- | ------------- |
| **Author**       | Kevin YAKPOVI |
| **Last Updated** | June 2026     |
| **Status**       | Published     |

---

## Document Purpose

### Objective

Provide a standardized validation procedure for verifying the operational health of a Wazuh Multi-Node GitOps deployment.

Use these checks:

- After initial deployment
- After upgrades
- After configuration changes
- During troubleshooting
- During routine operational reviews

### Blueprint Approach

This guide is part of the OpenSOC GitOps reference architecture. Container names, domains, project identifiers, and deployment-specific values should be adapted to match the target environment.

### Documentation Suite

Part of the OpenSOC GitOps Documentation Suite.

| #      | Document                                       | Description                      | Status           |
| ------ | ---------------------------------------------- | -------------------------------- | ---------------- |
| 00     | [README](../README.md)                            | Project overview and quick start | ✅ Production     |
| 01     | [Deployment Guide](01-deployment-guide.md)     | Full deployment procedure        | ✅ Production     |
| 02     | [Secret Rotation Guide](02-secret-rotation.md) | Credentials rotation runbook     | ✅ Production     |
| 03     | [Troubleshooting Guide](03-troubleshooting.md) | Incident diagnosis runbook       | ✅ Production     |
| **04** | **Health Check Guide**                         | **This document**                | **✅ Production** |
| 05     | [Architecture Decision Records](05-architecture-decision-records.md) | Design rationale and trade-offs | ✅ Production |
| 06     | [Teardown & Clean Reinstall Runbook](06-teardown-reinstall.md) | Full teardown and from-scratch reinstall | ✅ Production |

### Intended Audience

|Role|Relevance|
|---|---|
|Security Engineers|Primary audience — post-deployment and routine validation|
|DevOps Engineers|Service and container health verification|
|Platform Engineers|Infrastructure and network layer validation|
|Cloud Engineers|Environment-specific adaptation|
|Startup Security Teams|Operational reference|

### Prerequisites

- Wazuh Multi-Node deployed via Docker Compose
- Dokploy operational
- Basic Docker and Linux administration knowledge

---

## Scope

This guide validates the operational health of:

- Wazuh Manager Cluster (Master + Worker)
- Wazuh Indexer Cluster (3-node OpenSearch)
- Wazuh Dashboard
- Filebeat
- Traefik Reverse Proxy

**Out of scope:**

- Security hardening verification
- Performance benchmarking
- Agent deployment validation
- Detection rule testing

---

## Blueprint Notice

This guide is based on the OpenSOC GitOps reference architecture.

Container names, domains, project identifiers, and deployment-specific values should be adapted to match the target environment.

Commands and file paths shown throughout this document should be treated as reference procedures and adapted to match the target environment.

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

## Dependency Chain

Always validate from the bottom of the stack upward. A failure at any layer renders all dependent layers above it unavailable.

```
User
    ↓
Traefik                    (SSL termination · routing)
    ↓
Wazuh Dashboard (5601)     (interface · depends on API)
    ↓
Wazuh API (55000)          (core · depends on Indexer)
    ↓
OpenSearch Cluster (9200)  (storage · foundation)
```

> [!NOTE] A yellow cluster status indicates reduced redundancy but the stack is generally operational. A red status means data may be inaccessible — investigate immediately before proceeding with other checks.

---

## Success Criteria

The deployment is considered healthy when all of the following conditions are met:

- Factory/default credentials are rejected (`401`) — **see Check 0**
- All containers report `Up` status
- OpenSearch cluster status is `green`
- All 3 indexer nodes are present
- All shards are in `STARTED` state
- Filebeat connectivity tests succeed on both Master and Worker
- Wazuh API returns `HTTP 401` (running, authentication required)
- Dashboard returns `HTTP 200` or `302`
- Traefik reports no TLS, certificate, or routing errors
- Dashboard UI loads successfully and displays agents and alerts

---

## Check 0 — Default Credentials Guard

**Run this check first, before anything else.** A stack deployed with the public factory credentials shipped by `wazuh-docker` starts cleanly, reports `green`, and logs no error — it looks perfectly healthy while being protected only by published passwords. No other check in this guide detects that condition. This one does.

The factory defaults are public and identical on every default install:

| Account | Factory value |
|---|---|
| `admin` (Indexer) | `SecretPassword` |
| `wazuh-wui` (API) | `MyS3cr37P450r.*-` |
| `kibanaserver` (Dashboard) | `kibanaserver` |

Each factory credential **must be rejected**. A `200` (or a valid token) is a **critical finding**.

```bash
# 1. Indexer admin — factory value must return 401
docker exec wazuh1.indexer curl -k -s -o /dev/null -w "%{http_code}\n" \
  -u admin:SecretPassword \
  https://localhost:9200/_cluster/health
# Expected: 401   (200 = factory credential still live → CRITICAL)

# 2. Wazuh API — factory value must return 401
docker exec wazuh.master curl -k -s -o /dev/null -w "%{http_code}\n" \
  -u wazuh-wui:MyS3cr37P450r.*- \
  -X POST https://localhost:55000/security/user/authenticate
# Expected: 401
```

| Result | Interpretation | Action |
|---|---|---|
| `401` on all | Factory credentials are dead | ✅ Continue to Check 1 |
| `200` on indexer | `admin` hash never rotated in `internal_users.yml` | Rotate now — [Secret Rotation §1](02-secret-rotation.md) |
| `200`/token on API | `wazuh-wui` never rotated | Rotate now — [Secret Rotation §2](02-secret-rotation.md) |

> [!CAUTION] A passing `green` cluster with live factory credentials is **not** a healthy deployment. The plaintext half of the credential may have been changed in `wazuh.env` while the bcrypt hash in `internal_users.yml` still holds the factory value — `--env-file` never touches that hash. See [ADR-002 — Scope of `--env-file` injection](05-architecture-decision-records.md).

---

## Check 1 — Container Status

Verify that all Wazuh containers are running.

```bash
docker ps --format "table {{.Names}}\t{{.Status}}" | grep wazuh
```

Expected result: all containers in `Up` state.

Watch for `Restarting (...)` or `Exited (...)` — these states indicate configuration issues or insufficient resources.

```bash
# Quick check for restart loops
docker ps --format "table {{.Names}}\t{{.Status}}" | grep -E "wazuh|Restarting"
# → no lines containing "Restarting" should appear
```

---

## Check 2 — OpenSearch Cluster Health

```bash
docker exec wazuh1.indexer curl -k -s \
  -u admin:<INDEXER_PASSWORD> \
  https://localhost:9200/_cluster/health?pretty
```

Expected result:

```json
{
  "status": "green",
  "number_of_nodes": 3,
  "active_shards_percent_as_number": 100.0
}
```

|Status|Interpretation|Action|
|---|---|---|
|`green`|All nodes and shards healthy|✅ Continue|
|`yellow`|Reduced redundancy, data accessible|Investigate missing replica|
|`red`|Critical — data may be inaccessible|Stop and investigate immediately|

---

## Check 3 — OpenSearch Node Topology

Verify that all three indexer nodes have joined the cluster.

```bash
docker exec wazuh1.indexer curl -k -s \
  -u admin:<INDEXER_PASSWORD> \
  https://localhost:9200/_cat/nodes?v
```

Expected result: three nodes listed — `wazuh1.indexer`, `wazuh2.indexer`, `wazuh3.indexer`.

> [!WARNING] If fewer than 3 nodes appear, the cluster is degraded. A two-node cluster loses quorum if one additional node fails. Investigate the missing node before proceeding.

---

## Check 4 — Shard Allocation

Verify that all shards are properly allocated and no unassigned shards remain.

```bash
docker exec wazuh1.indexer curl -k -s \
  -u admin:<INDEXER_PASSWORD> \
  https://localhost:9200/_cat/shards?v | grep -v STARTED
```

Expected result: no output — all shards in `STARTED` state.

```bash
# Count unassigned shards
docker exec wazuh1.indexer curl -k -s \
  -u admin:<INDEXER_PASSWORD> \
  https://localhost:9200/_cluster/health?pretty | grep unassigned_shards
# Expected: "unassigned_shards": 0
```

---

## Check 5 — Filebeat Connectivity

Verify connectivity between Filebeat and the OpenSearch cluster on both Manager nodes.

```bash
# Master
docker exec wazuh.master filebeat test output

# Worker
docker exec wazuh.worker filebeat test output
```

Expected result on each:

```
talk to server... OK
```

> [!IMPORTANT] Filebeat connectivity failures on the Master or Worker indicate that alerts are not being indexed. Verify environment variables (`INDEXER_USERNAME`, `INDEXER_PASSWORD`) and OpenSearch cluster health before investigating further.

---

## Check 6 — Wazuh API Availability

Verify that the API service is responding and TLS is operational.

```bash
docker exec wazuh.master curl -k -s \
  https://localhost:55000/ \
  -w "\nHTTP:%{http_code}"
```

Expected result:

```
{"title": "Unauthorized", "detail": "No authorization token provided"}
HTTP:401
```

> [!TIP] `HTTP:401` confirms the API is running and TLS is functional. It does not indicate an authentication problem — it means the endpoint is reachable and requires credentials as expected.

---

## Check 7 — Dashboard Availability

Verify that the Dashboard service is responding.

```bash
docker exec wazuh.dashboard curl -s \
  -o /dev/null \
  -w "%{http_code}\n" \
  http://localhost:5601
```

Expected result: `200` or `302`

```bash
# Check for Dashboard errors
docker logs wazuh.dashboard --tail 20 | grep -E "error|Error|failed|FATAL"
# → no lines should appear
```

---

## Check 8 — Traefik Reverse Proxy

Inspect recent Traefik logs for routing or TLS errors.

```bash
docker logs traefik --tail 20 2>&1 | grep -E "error|certificate|bad gateway|unreachable"
# → no lines should appear
```

```bash
# Verify external HTTPS access
curl -s -o /dev/null -w "%{http_code}\n" https://wazuh.your-domain.com
# Expected: 200 or 302
```

---

## Dashboard Validation

Open https://wazuh.your-domain.com in a browser and log in using the credentials defined in your external `wazuh.env` file.

The exact location of this file depends on your environment. The reference implementation uses:

`/etc/dokploy/secrets/wazuh.env`

Verify the following:

- Dashboard loads successfully with no errors
- Agents appear in the inventory
- Alerts are indexed and visible (check `wazuh-alerts-*` index)
- Cluster health indicator shows green
- No authentication errors are displayed in the UI or logs

---

## Quick Troubleshooting Reference

| Symptom                       | Possible Cause                                 | Next Step                                                   |
| ----------------------------- | ---------------------------------------------- | ----------------------------------------------------------- |
| Factory password returns 200  | Credential hash never rotated                  | [Secret Rotation Guide](02-secret-rotation.md)              |
| HTTP 502 from Traefik         | Dashboard container unavailable                | Check 1 → Check 7                                           |
| Login failure                 | Credential mismatch                            | See [Secret Rotation Guide](02-secret-rotation.md)          |
| Empty Dashboard / no alerts   | OpenSearch cluster not ready or Filebeat issue | Check 2 → Check 5                                           |
| Cluster status yellow         | Missing replica node                           | Check 3                                                     |
| Cluster status red            | Node or shard allocation failure               | Check 3 → Check 4                                           |
| Missing alerts                | Filebeat indexing failure                      | Check 5                                                     |
| Dashboard unreachable         | Traefik routing or certificate issue           | Check 8                                                     |
| Variables empty in containers | `--env-file` misconfigured                     | See [Troubleshooting Guide — Case 1 Step 4](03-troubleshooting.md) |

---

## Configuration Reference

The OpenSOC GitOps blueprint uses a hybrid storage model:

- Configuration files and certificates are bind-mounted from absolute host paths
- Runtime data is stored in named Docker volumes
- Secrets remain external to Git in `/etc/dokploy/secrets/`

| File / Component | Location | Purpose | In Git |
|------------------|----------|----------|:------:|
| `wazuh.env` | `/etc/dokploy/secrets/` | Credentials and environment variables | ❌ |
| `internal_users.yml` | `config/wazuh_indexer/` (bind mount) | OpenSearch Security users (bcrypt hashes) | ❌ |
| `opensearch_dashboards.yml` | `config/wazuh_dashboard/` (bind mount) | Dashboard configuration | ✅ |
| `wazuh.yml` | `config/wazuh_dashboard/` (bind mount) | Dashboard ↔ API integration | ✅ |
| `ossec.conf` | `config/wazuh_cluster/` (bind mount) | Manager cluster configuration | ✅ |
| TLS certificates | `config/wazuh_indexer_ssl_certs/` (bind mount) | Internal TLS trust chain | ❌ |
| `docker-compose.yml` | Repository | Stack blueprint | ✅ |
| `wazuh*.indexer.yml` | Repository | OpenSearch node configuration | ✅ |
| OpenSearch data | Named Docker volumes | Cluster runtime data | ❌ |
| Manager logs | Named Docker volumes | Wazuh runtime data | ❌ |
| Dashboard runtime data | Named Docker volumes | Dashboard state | ❌ |

> [!IMPORTANT]
>
> Configuration files and certificates are intentionally bind-mounted to simplify operational tasks such as certificate renewal, secret rotation, and cluster tuning.
>
> Runtime data remains stored in named Docker volumes and should never be modified directly.

---

© 2026 Kevin YAKPOVI · Security Engineer · Open Source SOC Builder

OpenSOC GitOps Dokploy Blueprint — https://github.com/Kev1-alt/OpenSOC-GitOps-Dokploy

Documentation licensed under Creative Commons Attribution 4.0 International (CC BY 4.0). Source code and infrastructure examples licensed under Apache License 2.0.

This document is part of the OpenSOC GitOps Documentation Suite.
