# Troubleshooting Guide — Wazuh Multi-Node Stack


|**Version**|1.0|
|---|---|
|**Author**|Kevin YAKPOVI|
|**Last Updated**|June 2026|
|**Status**|Production|


---

## Document Purpose

### Objective

Provide a structured troubleshooting methodology for diagnosing and resolving common issues affecting a Wazuh Multi-Node deployment running on Dokploy and exposed through Traefik.

### Blueprint Approach

This guide is part of the OpenSOC GitOps reference architecture. It documents the diagnostic methodology used in the reference implementation. Organizations that modify network topology, container names, service discovery, or reverse proxy configurations may need to adapt diagnostic commands accordingly.

### Documentation Suite

Part of the OpenSOC GitOps Documentation Suite.

| #      | Document                                       | Description                      | Status           |
| ------ | ---------------------------------------------- | -------------------------------- | ---------------- |
| 00     | [README](../README.md)                         | Project overview and quick start | ✅ Production     |
| 01     | [Deployment Guide](01-deployment-guide.md)     | Full deployment procedure        | ✅ Production     |
| **02** | [Secret Rotation Guide](02-secret-rotation.md) | Credentials rotation runbook     | **✅ Production** |
| 03     | Troubleshooting Guide                          | **This document**                | ✅ Production     |
| 04     | [Health Check Guide](04-health-check.md)       | Post-deployment validation       | ✅ Production     |
| 05     | [Architecture Decision Records](05-adr.md)     | Design rationale and trade-offs  | ✅ Production     |

### Intended Audience

|Role|Relevance|
|---|---|
|Security Engineers|Primary audience — incident response and diagnosis|
|DevOps Engineers|Container and infrastructure troubleshooting|
|Platform Engineers|Network and reverse proxy diagnosis|
|Cloud Engineers|Environment-specific adaptation|
|Startup Security Teams|Operational reference|

### Prerequisites

- Wazuh Multi-Node deployed via Docker Compose
- Dokploy operational
- Basic Docker and Linux administration knowledge

---

## Scope

This guide covers the diagnosis of the most common incidents affecting a Wazuh Multi-Node deployment orchestrated by Dokploy and exposed through Traefik as a reverse proxy.

**Covered:**

- 502 Bad Gateway errors
- OpenSearch authentication failures
- Security plugin initialization errors
- TLS certificate inconsistencies
- Dashboard API connectivity errors (ERROR 3099)
- Environment variable injection issues
- Host-level resource exhaustion

**Out of scope:**

- Wazuh Agents
- Detection rules
- Third-party integrations (Shuffle, TheHive, Cortex, etc.)

---

## Blueprint Notice

This guide is based on the OpenSOC GitOps reference architecture.

Organizations that modify network topology, container names, service discovery, or reverse proxy configurations may need to adapt diagnostic commands accordingly.

Commands and file paths shown throughout this document should be treated as reference procedures and adapted to match the target environment.

### Storage Model

The reference architecture uses a hybrid storage model:

- Named Docker volumes for runtime data
- Bind mounts from `./config/` for operational configuration

Examples:

```
Runtime Data
├── wazuh-indexer-data-*
├── wazuh-manager-data
├── wazuh-dashboard-data
└── wazuh-logs-*
```

```
Configuration
├── ./config/wazuh_dashboard/
├── ./config/wazuh_indexer/
├── ./config/wazuh_indexer_ssl_certs/
└── ./config/wazuh_cluster/
```

For the rationale behind this design, see:

```
ADR-003 — Hybrid Volume Strategy
```

---

## Diagnostic Model

All incidents should be analyzed according to the actual request flow within the stack:

```
User
    ↓
Traefik          ← Entry point — generates 502 errors
    ↓
Dashboard        ← Interface — depends on API
    ↓
API Manager      ← Core — depends on Indexer
    ↓
OpenSearch       ← Storage — foundation of the stack
```

A failure at any layer renders all dependent layers above it unavailable.

**Always isolate the issue at the lowest layer before moving upward through the stack.**

---

## Case 0 — Server Health Assessment

Before performing any advanced troubleshooting, verify the overall health of the host.

Many Wazuh incidents are caused by:

- Disk exhaustion
- Memory exhaustion
- OOM Killer events
- Container restart loops
- Excessive CPU utilization

### Verify Container Status

```bash
docker ps -a
```

Expected result: `STATUS = Up` on all containers.

Watch for `Restarting (...)` or `Exited (...)` — these states indicate configuration issues or insufficient resources.

### Verify Memory Usage

```bash
free -h
```

Watch for `available` close to `0` — insufficient memory prevents Dashboard or Indexer services from starting correctly.

### Verify Disk Usage

```bash
df -h
```

Watch for `Use% > 90%` — disk saturation frequently causes OpenSearch failures, index corruption, and Dashboard startup errors.

### Verify System Load

```bash
uptime
```

Compare the load average against the number of available CPU cores.

### Verify Docker Resource Usage

```bash
docker stats --no-stream
```

Review container memory consumption, CPU utilization, and abnormal resource patterns.

---

## Case 1 — 502 Bad Gateway

### Diagnosis in Under 2 Minutes

**Step 0 — Verify Traefik**

A 502 error is most often generated by Traefik, not Wazuh.

```bash
docker logs traefik --tail 50 2>&1 | \
  grep -E "bad gateway|unreachable|connection refused|error"
```

|Result|Interpretation|Action|
|---|---|---|
|`bad gateway` or `server unreachable`|Backend unreachable|Proceed to Step 1|
|`certificate` or `TLS`|Certificate issue|Check Dokploy Let's Encrypt config|
|No errors|Traefik OK|Proceed to Step 1|

---

**Step 1 — Verify the Dashboard**

```bash
docker exec wazuh.dashboard \
  curl -s -o /dev/null -w "%{http_code}\n" http://localhost:5601
```

|Result|Interpretation|Action|
|---|---|---|
|`200` or `302`|Dashboard operational|DNS or Traefik config issue → check domain in Dokploy|
|`connection refused`|Dashboard service stopped|Check Dashboard logs|
|Timeout|Dashboard blocked or overloaded|Check server resources|

```bash
# If Dashboard is not responding — check logs
docker logs wazuh.dashboard --tail 50 | grep -E "error|Error|failed|FATAL"
```

---

**Step 2 — Verify the API Manager**

```bash
docker exec wazuh.master curl -k -s \
  https://localhost:55000/ \
  -w "\nHTTP_CODE:%{http_code}"
```

|Result|Interpretation|Action|
|---|---|---|
|`{"title":"Unauthorized"}` + `HTTP_CODE:401`|API active, TLS OK, endpoint reachable|Check `wazuh-wui` credentials|
|`connection refused`|API service stopped|Check Manager logs|
|Timeout|API blocked or overloaded|Check server resources|

```bash
# If API is not responding — check logs
docker logs wazuh.master --tail 50 | grep -E "error|Error|401|Unauthorized|failed"
```

---

**Step 3 — Verify OpenSearch**

If the indexer is down, there is no point continuing up the chain.

```bash
docker exec wazuh1.indexer curl -k -s \
  -u admin:<INDEXER_PASSWORD> \
  https://localhost:9200/_cluster/health?pretty
```

| Result               | Interpretation                  | Action                                             |
| -------------------- | ------------------------------- | -------------------------------------------------- |
| `"status": "green"`  | Healthy cluster, 3 active nodes | Issue is upstream                                  |
| `"status": "yellow"` | Degraded cluster                | Identify missing node                              |
| `"status": "red"`    | Critical cluster state          | Check indexer logs                                 |
| `401 Unauthorized`   | Invalid credentials             | See [Secret Rotation Guide](02-secret-rotation.md) |
| `connection refused` | Indexer not started             | Check indexer logs                                 |

```bash
# Verify node membership
docker exec wazuh1.indexer curl -k -s \
  -u admin:<INDEXER_PASSWORD> \
  https://localhost:9200/_cat/nodes?v
```

Expected:

```
wazuh1.indexer
wazuh2.indexer
wazuh3.indexer
```

Review logs if necessary:

```
docker logs wazuh1.indexer --tail 100 | \
grep -Ei "error|failed|exception"
```

---

**Step 4 — Verify Environment Variables**

A common issue with Dokploy — variables remain empty if `--env-file` is misconfigured.

```bash
docker exec wazuh.master env | \
  grep -E "INDEXER_USERNAME|INDEXER_PASSWORD|API_USERNAME|API_PASSWORD"
```

|Result|Interpretation|Action|
|---|---|---|
|Non-empty values|Variables OK|Issue is elsewhere|
|Empty values|`--env-file` misconfigured|Check Run Command in Dokploy → Advanced|

```bash
# Verify the secrets file is accessible
ls -la /etc/dokploy/secrets/wazuh.env
# → must exist and not be empty
```

Expected:

```
-rw------- root root wazuh.env
```

Verify the Dokploy Run Command uses:

```
--env-file /etc/dokploy/secrets/wazuh.env
```

> [!IMPORTANT] **Why `--env-file` in the Run Command rather than `env_file` in `docker-compose.yml`**
> 
> Passing `--env-file` via the Run Command ensures that all variables are fully resolved by Docker Compose before Wazuh and OpenSearch entrypoint scripts execute.
> 
> Failure to do so may result in:
> - Empty credentials
> - OpenSearch authentication failures
> - Dashboard startup failures
> - Cluster initialization issues
>  
See: ADR-002 — External Secrets via --env-file

**Step 5 — Verify Certificates**

Verify certificate validity directly from inside the indexer.

```bash
# Verify certificate validity
docker exec wazuh1.indexer \
  openssl x509 \
  -in /usr/share/wazuh-indexer/config/certs/admin.pem \
  -noout -subject -issuer -dates
```

Expected:

```
admin.pem
admin-key.pem
root-ca.pem
```

Verify mount visibility from inside the container:

```
docker exec wazuh1.indexer \
ls -la /usr/share/wazuh-indexer/config/certs/
```

If files are missing:

- verify Git repository synchronization
- verify bind mount definitions
- verify file permissions

Because the reference architecture uses bind mounts for certificates, certificate troubleshooting should always begin in:

```
./config/wazuh_indexer_ssl_certs/
```

and not in Docker volumes.

See:

```
ADR-003 — Hybrid Volume Strategy
```
---

## Case 2 — Authentication Finally Failed

### Symptom

OpenSearch logs contain repeated entries:

```
Authentication finally failed for admin from 10.x.x.x:XXXXX
```

### Diagnosis

**Identify the source container:**

```bash
# Find which IP is sending failed auth requests
docker logs wazuh1.indexer 2>&1 | \
  grep "Authentication finally failed" | \
  awk '{print $NF}' | sort | uniq -c | sort -rn | head -10

# Map the IP address to a container
docker inspect wazuh.master | jq '.[0].NetworkSettings.Networks'
docker inspect wazuh.dashboard | jq '.[0].NetworkSettings.Networks'
```

**Verify environment variables of the failing component:**

```bash
docker exec wazuh.master env | grep -E "INDEXER_USERNAME|INDEXER_PASSWORD"
docker exec wazuh.dashboard env | grep -E "DASHBOARD_USERNAME|DASHBOARD_PASSWORD"
```

**Test authentication manually:**

```bash
docker exec wazuh1.indexer curl -k -s \
  -u admin:<INDEXER_PASSWORD> \
  https://localhost:9200/_cluster/health?pretty
# → 200 OK = correct password, variable injection issue → fix --env-file
# → 401 = incorrect password → see Secret Rotation Guide
```

---

# Case 3 — Not Yet Initialized

### Symptom

```
Not yet initialized (you may need to run securityadmin)
```

### Cause

The OpenSearch Security plugin has not been initialized. This typically occurs after:

- Initial deployment
- Indexer data volume recreation
- Security configuration reset
- Certificate replacement with incorrect permissions

### Resolution

```
# Wait for the cluster to formdocker logs wazuh1.indexer 2>&1 | \grep -E "started|Security|initialized" | tail -5
```

```
# Run securityadmindocker exec -e JAVA_HOME=/usr/share/wazuh-indexer/jdk \  -it wazuh1.indexer \  bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/securityadmin.sh \  -cd /usr/share/wazuh-indexer/config/opensearch-security/ \  -icl -nhnv \  -cacert /usr/share/wazuh-indexer/config/certs/root-ca.pem \  -cert /usr/share/wazuh-indexer/config/certs/admin.pem \  -key /usr/share/wazuh-indexer/config/certs/admin-key.pem \  -h wazuh1.indexer -p 9200
```

Expected result:

```
Done with success
```

> [!WARNING]  
> **Silent Failure — Certificate Permissions**
> 
> If `securityadmin.sh` exits without `Done with success` and without a clear error message, the issue is frequently caused by certificate file permissions.
> 
> In this reference architecture, certificates are stored under:
> 
> ```
> ./config/wazuh_indexer_ssl_certs/
> ```
> 
> and mounted into the containers using bind mounts.
> 
> Certificate files must be readable by the `wazuh-indexer` process (UID 1000 inside the container).

### Verify permissions

```
docker exec wazuh1.indexer id
```

```
docker exec wazuh1.indexer \  ls -la /usr/share/wazuh-indexer/config/certs/
```

Expected:

```
1000 1000
```

or equivalent read permissions for the wazuh-indexer user.

### Fix permissions

```
sudo chown -R 1000:1000 \./config/wazuh_indexer_ssl_certs
```

```
sudo chmod 640 \./config/wazuh_indexer_ssl_certs/*.pem
```

Then redeploy the stack and rerun `securityadmin.sh`.

---

# Case 4 — SSLHandshakeException / certificate_unknown

### Symptom

```
SSLHandshakeException: (certificate_unknown) Received fatal alert: certificate_unknown
```

### Cause

An inconsistent certificate trust chain.

This usually happens when:

- Certificates were regenerated partially
- Files from different certificate generations were mixed
- Certificates were manually replaced individually
- A different Root CA was introduced

### Diagnosis

```
docker exec wazuh1.indexer bash -c "openssl verify \  -CAfile /usr/share/wazuh-indexer/config/certs/root-ca.pem \  /usr/share/wazuh-indexer/config/certs/admin.pem"
```

Expected result:

```
admin.pem: OK
```

If verification fails, all certificates must be regenerated from the same CA.

### Resolution

> [!CAUTION]
> 
> A full certificate regeneration is required.
> 
> Do not replace individual certificates. All files must originate from the same certificate generation process.

Follow the complete procedure documented in:

```
Deployment Guide → Generate Certificates
```

After generating the new certificates:

```
cp generated-certs/* \./config/wazuh_indexer_ssl_certs/
```

Apply permissions:

```
sudo chown -R 1000:1000 \./config/wazuh_indexer_ssl_certs
```

```
sudo chmod 640 \./config/wazuh_indexer_ssl_certs/*.pem
```

Verify the files:

```
ls -la ./config/wazuh_indexer_ssl_certs/
```

Then redeploy from Dokploy.

---

## Case 5 — ERROR 3099 (Dashboard)

### Symptom

```
Error: 3099 - Server not ready yet
```

### Cause

The Dashboard cannot communicate with the API Manager on port 55000.

### Diagnosis

```bash
# Verify API availability
docker exec wazuh.master curl -k -s \
  https://localhost:55000/ \
  -w "\nHTTP_CODE:%{http_code}"

# Verify Dashboard → API connectivity
docker exec wazuh.dashboard curl -k -s \
  https://wazuh.master:55000/ \
  -w "\nHTTP_CODE:%{http_code}"

# Verify wazuh-wui authentication
TOKEN=$(docker exec wazuh.master curl -k -s \
  -u wazuh-wui:<API_PASSWORD> \
  -X POST https://localhost:55000/security/user/authenticate \
  | grep -o '"token":"[^"]*"' | cut -d'"' -f4)

echo "Token: $TOKEN"
# If empty → invalid wazuh-wui password → see Secret Rotation Guide
```

---

## Summary Table

|Symptom|Layer|Quick Check|Action|
|---|---|---|---|
|502 Bad Gateway|Traefik|`docker logs traefik`|Verify backend connectivity|
|Dashboard unavailable|Dashboard|`curl localhost:5601`|Review Dashboard logs|
|ERROR 3099|Dashboard / API|`curl localhost:55000`|Verify API and credentials|
|Authentication finally failed|Indexer|Check env variables|Fix `--env-file` or rotate password|
|Not yet initialized|OpenSearch Security|Run securityadmin|Initialize security plugin|
|certificate_unknown|TLS|`openssl verify`|Full certificate regeneration|
|Empty variables|Dokploy|`docker exec env`|Fix Run Command `--env-file`|
|Restarting containers|Docker|`docker ps -a`|Verify server resources|
|Disk full|Host|`df -h`|Free disk space|
|Memory exhausted|Host|`free -h`|Increase available memory|

---

## References

| Resource                          | URL                                              |
| --------------------------------- | ------------------------------------------------ |
| Wazuh Documentation               | https://documentation.wazuh.com                  |
| OpenSearch Security Documentation | https://opensearch.org/docs/latest/security/     |
| Dokploy Documentation             | https://docs.dokploy.com                         |
| Traefik Documentation             | https://doc.traefik.io/traefik/                  |
| Secret Rotation Guide             | [02-secret-rotation.md](02-secret-rotation.md)   |
| Deployment Guide                  | [01-deployment-guide.md](01-deployment-guide.md) |

---

---

© 2026 **Kevin YAKPOVI** | Security Engineer · Open Source SOC Builder

🐙 [github.com/Kev1-alt](https://github.com/Kev1-alt) · 💼 [linkedin.com/in/kevin-yakpovi-384b4619a](https://linkedin.com/in/kevin-yakpovi-384b4619a)

*Published under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — Any reproduction must credit the original author.*
*Part of [OpenSOC GitOps Dokploy](https://github.com/Kev1-alt/OpenSOC-GitOps-Dokploy) · Document maintained at: `01-Wazuh-Stack/docs/[nom-du-fichier.md]`*`_
